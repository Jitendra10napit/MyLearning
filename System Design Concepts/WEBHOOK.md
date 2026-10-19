Here's the full flow at code level, showing **who calls whom and when**. I'll use two .NET apps:

- **PaymentProvider**: plays the role of Razorpay (the system that *sends* the webhook)
- **MyShop**: your application (the system that *receives* it)

## The timeline: who calls whom

```
STEP 0 (once, at setup)   MyShop ──── "register my URL" ────► PaymentProvider
STEP 1 (customer pays)    Customer ── POST /payments ───────► PaymentProvider
STEP 2 (event happens)    PaymentProvider saves the payment and raises the event "payment.success"
STEP 3 (webhook call)     PaymentProvider ── POST /webhooks/payment ──► MyShop   ◄── THIS IS THE WEBHOOK
STEP 4 (reply)            MyShop ──── 200 OK ────► PaymentProvider
STEP 5 (if failed)        PaymentProvider retries STEP 3 after 1 min, 5 min, 30 min...
```

The key point: **in Step 3 the provider calls you**, not the other way around. Your code never asks "has the payment happened?" It just waits for the call.

---

## Step 0: Registration (MyShop tells the provider its URL)

This happens **once**, usually through the provider's dashboard. In code it looks like this:

```csharp
// PaymentProvider side: stores who wants to be notified
public class Subscription
{
    public string Url { get; set; }      // "https://myshop.com/webhooks/payment"
    public string Event { get; set; }    // "payment.success"
    public string Secret { get; set; }   // "whsec_abc123", shared with MyShop
}

app.MapPost("/subscriptions", (Subscription sub, AppDb db) =>
{
    db.Subscriptions.Add(sub);
    db.SaveChanges();
    return Results.Ok();
});
```

MyShop makes this call once:
```csharp
await http.PostAsJsonAsync("https://provider.com/subscriptions", new {
    Url = "https://myshop.com/webhooks/payment",
    Event = "payment.success",
    Secret = "whsec_abc123"
});
```

---

## Steps 1–3: Customer pays → provider fires the webhook

```csharp
// ===================== PaymentProvider app =====================

// STEP 1: the customer's payment hits the provider
app.MapPost("/payments", async (PaymentRequest req, AppDb db, WebhookSender sender) =>
{
    var payment = new Payment { Id = "pay_123", OrderId = req.OrderId, Amount = req.Amount, Status = "Success" };
    db.Payments.Add(payment);
    await db.SaveChangesAsync();

    // STEP 2: the event happened, so notify everyone who subscribed to it
    await sender.SendAsync("payment.success", payment);

    return Results.Ok(payment);
});

// STEP 3: the webhook sender, which does the actual call to MyShop
public class WebhookSender(AppDb db, HttpClient http)
{
    public async Task SendAsync(string eventName, object data)
    {
        var subscribers = db.Subscriptions.Where(s => s.Event == eventName).ToList();

        foreach (var sub in subscribers)
        {
            var body = JsonSerializer.Serialize(new {
                id = Guid.NewGuid().ToString(),     // unique event id
                @event = eventName,
                data
            });

            // Sign the body with the shared secret so MyShop can trust it
            var signature = Convert.ToHexString(
                HMACSHA256.HashData(Encoding.UTF8.GetBytes(sub.Secret), Encoding.UTF8.GetBytes(body)));

            var request = new HttpRequestMessage(HttpMethod.Post, sub.Url)
            {
                Content = new StringContent(body, Encoding.UTF8, "application/json")
            };
            request.Headers.Add("X-Signature", signature);

            // ►►► THIS LINE IS THE WEBHOOK: the provider calls YOUR URL ◄◄◄
            var response = await http.SendAsync(request);

            if (!response.IsSuccessStatusCode)
                db.RetryQueue.Add(new RetryItem(sub.Url, body, signature, attempt: 1)); // STEP 5
        }
        await db.SaveChangesAsync();
    }
}
```

---

## Step 4: MyShop receives the call

```csharp
// ===================== MyShop app =====================

// MyShop never calls this itself. The provider calls it in STEP 3.
app.MapPost("/webhooks/payment", async (HttpRequest request, ShopDb db) =>
{
    // 1. Read the raw body
    var body = await new StreamReader(request.Body).ReadToEndAsync();

    // 2. Is it really from the provider? Recompute the signature and compare.
    var received = request.Headers["X-Signature"].ToString();
    var expected = Convert.ToHexString(
        HMACSHA256.HashData(Encoding.UTF8.GetBytes("whsec_abc123"), Encoding.UTF8.GetBytes(body)));

    if (received != expected)
        return Results.Unauthorized();          // fake call, reject it

    var evt = JsonSerializer.Deserialize<WebhookEvent>(body)!;

    // 3. Already processed? (The provider may send the same event twice.)
    if (db.ProcessedEvents.Any(e => e.Id == evt.Id))
        return Results.Ok();

    // 4. Do the business work
    var order = db.Orders.First(o => o.Id == evt.Data.OrderId);
    order.Status = "Paid";
    db.ProcessedEvents.Add(new ProcessedEvent { Id = evt.Id });
    await db.SaveChangesAsync();

    // 5. Reply 200 so the provider knows it was delivered and won't retry
    return Results.Ok();
});
```

---

## Step 5: Retry when MyShop was down

```csharp
// PaymentProvider: a background job that runs every minute
public class RetryWorker(AppDb db, HttpClient http) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            foreach (var item in db.RetryQueue.Where(r => r.NextTryAt <= DateTime.UtcNow).ToList())
            {
                var req = new HttpRequestMessage(HttpMethod.Post, item.Url)
                    { Content = new StringContent(item.Body, Encoding.UTF8, "application/json") };
                req.Headers.Add("X-Signature", item.Signature);

                var res = await http.SendAsync(req, ct);           // calls MyShop again

                if (res.IsSuccessStatusCode) db.RetryQueue.Remove(item);
                else { item.Attempt++; item.NextTryAt = DateTime.UtcNow.AddMinutes(Math.Pow(5, item.Attempt)); }
            }
            await db.SaveChangesAsync(ct);
            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}
```

---

## Quick summary table

| Step | Who calls | Whom | When | Code |
|---|---|---|---|---|
| 0 | MyShop | Provider | Once, at setup | `POST /subscriptions` |
| 1 | Customer | Provider | When paying | `POST /payments` |
| 3 | **Provider** | **MyShop** | **Right after the payment succeeds** | `http.SendAsync(...)` → `POST /webhooks/payment` |
| 4 | MyShop | Provider (the reply) | Immediately | `return Results.Ok()` |
| 5 | Provider | MyShop | Only if Step 3 failed | `RetryWorker` |

**Compare this with a normal API.** With a normal API, MyShop would call `GET /payments/pay_123` again and again to check the status. With a webhook, MyShop just exposes `POST /webhooks/payment` and waits. The provider does the calling when the event happens.

**To test locally:** the provider can't reach `localhost`, so use **ngrok** (`ngrok http 5000`) or **VS Dev Tunnels** to get a public URL, and register that as your webhook URL.
