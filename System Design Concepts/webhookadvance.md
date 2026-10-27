# Webhook Notes: Payment Service + Acme (.NET / C#)

## 1. What is a webhook?

An HTTP **callback**: instead of the client asking "did anything change?" (polling), **the server calls the client's URL** when an event happens.

- **Normal API:** Client → Server ("give me data")
- **Webhook:** Server → Client ("something happened")

## 2. The three actors

| Actor | Role | Example URL | Has public URL? |
|---|---|---|---|
| **Acme UI** | Browser/mobile app the customer uses | `acme.com` | ❌ cannot receive webhooks |
| **Acme Backend** | Client's .NET API, **receives** the webhook and **verifies** | `api.acme.com` | ✅ |
| **Payment Service** | Your .NET API, **sends** the webhook, **source of truth** | `pay.example.com` | ✅ |

## 3. Credentials (where they live)

| Credential | Used for | Stored in |
|---|---|---|
| **API key** | Acme Backend → Payment Service calls (create payment, confirm status) | Acme Backend config only |
| **Webhook secret** | Payment Service signs webhooks; Acme Backend verifies them | Both backends, **never** in the UI |
| **User JWT** | Acme UI → Acme Backend | Browser |

## 4. Complete flow, step by step

```
 Acme UI              Acme Backend                    Payment Service
    │                      │                                 │
 0  │                      │─ POST /api/webhooks ───────────►│ register URL (one time)
    │                      │◄─ 201 {secret} ─────────────────│
    │                      │                                 │
 1  │─ POST /orders/42/checkout ─►│                          │
 2  │                      │─ POST /api/payments ───────────►│ create pay_789
    │                      │◄─ {paymentUrl} ─────────────────│
    │◄─ {paymentUrl} ──────│                                 │
 3  │─────────── redirect to hosted payment page ───────────►│ customer pays
 4  │◄────────── 302 redirect to returnUrl ──────────────────│ ⚠ UX only, don't trust
    │ "Processing…"        │                                 │
 5  │                      │◄─ POST /webhooks/payment ───────│ signed webhook
    │                      │   verify signature, enqueue     │
    │                      │── 200 OK ──────────────────────►│ mark Delivered
 6  │                      │─ GET /api/payments/pay_789 ────►│ confirm (background)
    │                      │◄─ {Succeeded, 1999} ────────────│
    │                      │   match amount → order = Paid   │
 7  │◄─ SignalR / polling ─│                                 │
    │ "Payment successful ✅"                                 │
```

## 5. Endpoint map: who calls whom

| # | Caller → Callee | Endpoint | Auth |
|---|---|---|---|
| 0 | Acme Backend → Payment Service | `POST /api/webhooks` | API key |
| 1 | Acme UI → Acme Backend | `POST /api/orders/{id}/checkout` | JWT |
| 2 | Acme Backend → Payment Service | `POST /api/payments` | API key |
| 3 | Browser → Payment Service | `GET /checkout/{paymentId}` | none (hosted page) |
| 4 | Payment Service → Browser | 302 to `returnUrl` | none |
| **5** | **Payment Service → Acme Backend** | **`POST /webhooks/payment`** ← **THE WEBHOOK** | **HMAC signature** |
| 6 | Acme Backend → Payment Service | `GET /api/payments/{id}` | API key |
| 7 | Acme UI → Acme Backend | SignalR or `GET /api/orders/{id}` | JWT |

## 6. Payment Service side (sender)

**Components:**
1. **`WebhooksController`**: `POST /api/webhooks` saves a subscription (url, events, secret).
2. **`WebhookPublisher`**: when a payment succeeds, inserts a `WebhookDelivery` row (Pending). **No HTTP call here.**
3. **`WebhookDispatcher`** (`BackgroundService`): picks up pending rows, signs them, POSTs to the Acme URL, retries on failure.
4. **`PaymentsController.Get`**: `GET /api/payments/{id}` lets Acme confirm the status.

**Signing:**
```csharp
public static string Sign(string secret, string id, long ts, string body)
{
    using var hmac = new HMACSHA256(Convert.FromBase64String(secret));
    return "v1," + Convert.ToBase64String(
        hmac.ComputeHash(Encoding.UTF8.GetBytes($"{id}.{ts}.{body}")));
}
```

**Headers sent:**
```
Webhook-Id: evt_555            ← unique, same on retries
Webhook-Timestamp: 1761559200
Webhook-Signature: v1,abc...
```

**Retry rules:**
- `2xx` → Delivered
- `5xx`, timeout, or `429` → retry with exponential backoff (1m, 2m, 4m, …)
- After N attempts → Failed (alert or disable the subscription)

## 7. Acme Backend side (receiver and verifier)

**Fast handler** (return 200 quickly):
```csharp
app.MapPost("/webhooks/payment", async (HttpRequest req, Channel<string> queue) =>
{
    var body = await new StreamReader(req.Body).ReadToEndAsync();      // raw body
    if (!IsValidSignature(req.Headers, body, webhookSecret))
        return Results.Unauthorized();
    await queue.Writer.WriteAsync(body);
    return Results.Ok();
});
```

**Background processor** runs the verification checklist:

| # | Check | Protects against |
|---|---|---|
| ① | **HMAC signature + timestamp (≤5 min)** | Fake or replayed webhooks |
| ② | **referenceId, amount, currency match the order** | Wrong order, underpayment |
| ③ | **`GET /api/payments/{id}`** from the Payment Service | Trusting the payload blindly |
| ④ | **Idempotency** (already Paid? already seen Webhook-Id?) | Duplicate deliveries |

Then: `order.Status = "Paid"` → push to the UI via SignalR.

**Reconciliation job** (safety net): every 15 min or nightly, find `PendingPayment` orders older than X minutes, call `GET /api/payments/{id}`, and update them.

## 8. Acme UI side

- Redirects the customer to `paymentUrl`.
- On return, shows "Processing…"
- Learns the result **only from the Acme Backend**, by SignalR push or polling `GET /api/orders/{id}`.
- **Never** verifies payment, **never** holds secrets, **never** receives webhooks.

## 9. Golden rules

1. **The webhook goes server → server.** The UI can't receive it.
2. **The Payment Service is the source of truth.** The Acme Backend verifies. The UI only displays.
3. **The redirect is for UX; the webhook and API confirmation are the truth.**
4. **Always verify the HMAC signature** on the **raw body**, using a constant-time compare.
5. **Return 200 fast** and process in the background.
6. **Expect duplicates** (at-least-once delivery), so make processing idempotent.
7. **Don't assume order.** Re-fetch the current state with the GET API.
8. **Queue in the sender.** Business code writes to an outbox table, and a worker sends.
9. **Retry with backoff**, keep delivery logs, allow manual redeliver.
10. **Have a reconciliation job** for webhooks that never arrive.
11. **HTTPS only.** Block private IPs when registering URLs (SSRF).

## 10. One-line conclusion

> **The Payment Service *tells* (webhook) → the Acme Backend *checks* (signature + amount + GET confirm + idempotency) → the Acme UI *shows* (SignalR or polling).**

This is the same pattern Stripe, PayPal and Razorpay use, and [Standard Webhooks](https://www.standardwebhooks.com/) formalizes the headers and signing.

I can also turn these notes into a saved page you can open again later.
