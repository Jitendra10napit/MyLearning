# Forward Proxy vs. Reverse Proxy — Simple Explanation with Real Examples

Think of a proxy as a middleman between two computers communicating over a network.

The difference is who the middleman works for.

- Forward proxy: works on behalf of the client.
- Reverse proxy: works on behalf of the server.

## 1. Forward proxy — Example of an office network

Imagine you work in a company where employees are not allowed to access certain websites directly.

Your laptop

You want to open github.com

Forward proxy

Checks company internet policies

GitHub server

Returns the requested webpage

### What happens step by step?

1. You enter `https://github.com` in your browser.
2. Your browser sends the request to the configured forward proxy.
3. The proxy checks whether access to GitHub is allowed.
4. If allowed, the proxy forwards the request to GitHub.
5. GitHub sends the response back through the proxy.
6. You see GitHub in your browser.

Why use a forward proxy?

The company can control which websites employees access, log outbound connections, and apply security policies.

Think of it as a company security guard who checks where employees want to go before allowing them out.

## 2. Reverse proxy — Example of your .NET Web API

Now imagine you've built an Employee Management application using Angular and ASP.NET Core Web API.

Your API is running on three servers to handle more users.

Angular application

Calls api.example.com/api/employees

Reverse proxy

Public entry point · Routes requests

API 1

API 2

API 3

The proxy forwards each request to a suitable backend instance.

### What happens step by step?

Suppose your Angular application sends this request:

```
GET https://api.example.com/api/employees
```

1. The browser connects to the public endpoint.
2. The reverse proxy receives the request.
3. It checks its routing rules and selects a backend server.
4. The selected .NET API executes the request and fetches employee records.
5. The API returns the response to the reverse proxy.
6. The reverse proxy forwards the response to Angular.

The Angular application uses one public URL. It doesn't need to know the private IP address of each API server.

Why use a reverse proxy?

- Distribute requests across multiple API servers.
- Hide backend network details.
- Manage HTTPS termination and routing.
- Help protect and monitor incoming traffic.

Think of it as a hotel receptionist who receives visitors and directs each visitor to the correct room.

## 3. Compare them side by side

[System Design Interview: 12 Templates](https://images.openai.com/static-rsc-4/2OUg5G65N2s39VSTCShUKNui51T0I6pQZFNsguA8Y38ENUKgwg6rp54rc-W_Sj2oS3IRyXfqXnYzeRksiRibpGuDC69o0z1dkUkFhPT3sZqb1Z875eQrNvBPi-CT2m-FbDmNy_obtmkm9FPZGA3l4nMvpUNMC_Jvt957nhZqE2Y?purpose=inline)

Forward proxy

Client → Proxy → Website

The client uses the proxy to access another server.

[What is Network Traffic Analysis (NTA)? | IBM](https://images.openai.com/static-rsc-4/zgSIhvITmWvgMZ4MoEESZ3J2iztYR2ZSpVOA9KSuMYQogISMm06l0qIev4mRANRkua-CfigUusGVp55VD-GUP_dRCFdHZrt4Pp-U8xcG6QranQYIDitwqyeryzFbqnC_lQ2WP1cRf4CSV0GxeA4nMfzfh8SC9JH2R0vkP79gSr0?purpose=inline)

Reverse proxy

Client → Proxy → Backend API

The server owner uses the proxy to receive and route client requests.

| Question          | Forward                 | Reverse                     |
| ----------------- | ----------------------- | --------------------------- |
| Represents        | Client                  | Server                      |
| Traffic direction | Outbound access         | Inbound access              |
| Typical purpose   | Internet access control | Application delivery        |
| Example           | Corporate web proxy     | Nginx in front of .NET APIs |

## 4. Where do these fit in Azure?

For your .NET and Azure architecture, common examples include:

- Azure Application Gateway: can act as a reverse proxy and Layer 7 load balancer for web applications.
- Azure API Management: acts as an API gateway, receiving API calls and forwarding them to backend services.
- Azure Front Door: provides a global application entry point with routing, acceleration, and edge security capabilities.

These products have overlapping capabilities, but they aren't identical.

## 5. Check your understanding

Scenario: Your company has 10 .NET API servers. Users call a single public URL, and incoming requests are distributed across those servers.

Which proxy is most appropriate?

Forward proxy

Reverse proxy

SOCKS proxy

&#x20;Correct!

A reverse proxy receives incoming requests on behalf of the backend servers and can distribute them across the 10 API instances.

Remember this interview rule: If the proxy helps a client reach external websites, think forward proxy. If it receives requests for your application servers, think reverse proxy.
