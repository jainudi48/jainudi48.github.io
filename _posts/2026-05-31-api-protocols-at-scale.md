---
layout: post
title: "API Protocols at Scale — A Deep Dive for Engineers"
date: 2026-05-31
categories: [system-design, backend, api]
tags: [rest, graphql, grpc, sse, websocket, system-design, api-design, backend, protocols]
---

# API Protocols at Scale — A Deep Dive for Engineers

## The question nobody asks first

When engineers design APIs they jump straight to implementation. REST because everyone uses it. GraphQL because it is modern. gRPC because it is fast. This is backwards thinking.

Every protocol is an answer to a set of questions. Before picking one, ask:

- Who is consuming this API? Do you know them?
- Do you control both sides — client and server?
- What matters more — flexibility or performance?
- Does the client need to receive data, send data, or both?
- How important is caching to your scale strategy?

Your protocol choice is your answer to these questions. Let us build the understanding from the ground up.

---

## The communication model underneath everything

All five protocols we cover here solve the same fundamental problem — two systems communicating over a network. But they make different assumptions about the shape of that communication.

There are really only two shapes:

```
Request-response:   client asks → server answers → done
                    connection closes or reused for next request

Persistent push:    connection stays open
                    server sends data as it happens
                    client does not need to ask repeatedly
```

REST, GraphQL, and gRPC are all request-response at their core. SSE and WebSocket are persistent push. Understanding this distinction first makes everything else fall into place.

---

## REST — What it actually is

Most engineers think REST means HTTP with JSON. That is not REST. That is an HTTP API. REST — Representational State Transfer — is an architectural style defined by Roy Fielding in his 2000 doctoral dissertation. It has six constraints:

1. **Client-Server** — concerns are separated. Client handles UI, server handles data.
2. **Stateless** — each request contains all context needed. Server stores no session state between requests.
3. **Cacheable** — responses must define whether they can be cached.
4. **Uniform Interface** — resources identified by URIs, manipulated through representations, self-descriptive messages, and HATEOAS.
5. **Layered System** — client cannot tell if it is talking to an origin server or an intermediary.
6. **Code on Demand** — optional. Server can send executable code.

The constraint most people miss is HATEOAS — Hypermedia as the Engine of Application State. A truly RESTful response contains links to next possible actions:

```json
{
  "order_id": "456",
  "status": "confirmed",
  "_links": {
    "cancel": "/orders/456/cancel",
    "track": "/orders/456/tracking",
    "invoice": "/orders/456/invoice"
  }
}
```

The client does not need to know URLs in advance. The server tells it what is possible next. Almost nobody implements this. Most APIs called REST are technically HTTP APIs — which is fine, but knowing the difference is what separates a Senior from a Staff engineer in interviews.

### Where REST genuinely wins

REST wins when your consumers are unknown and diverse. If you are building a public API that third-party developers will consume, REST is the right choice. Anyone with an HTTP client — curl, Postman, a Python script — can talk to it. No special tooling, no generated code, no proto files.

REST also wins when caching matters. Because each resource has a URL, CDNs and HTTP caches can cache responses using standard Cache-Control headers. At scale, a well-configured CDN can absorb 60-80% of traffic before it reaches your origin servers. No other protocol gives you this for free.

Statelessness is another underrated advantage. Because the server stores no session state, any server can handle any request. Horizontal scaling becomes trivial. Load balancers just round-robin across instances with no sticky session complexity.

### Where REST breaks at scale

**Over-fetching** is the first problem. A mobile client needs three fields — name, avatar, status. Your REST endpoint returns forty fields. You are shipping thirty-seven fields of wasted bandwidth on every request. At millions of requests per day this is not a minor inefficiency.

**Under-fetching and the N+1 problem** is the second. To render an order summary page:

```
GET /orders              → returns 20 orders
GET /orders/1/items      → need items for order 1
GET /orders/2/items      → need items for order 2
... 20 more requests
```

One screen, twenty-one HTTP round trips. Each round trip carries network latency. On mobile this is catastrophic.

**Weak contracts** is the third and most dangerous problem in production. REST has no enforced schema by default. A field gets renamed. A type changes from string to integer. A required field gets removed. Your client discovers this as a runtime error in production — not as a build failure. At scale, with dozens of teams consuming your API, this creates a constant stream of incidents.

---

## GraphQL — The problem it was actually built to solve

Facebook built GraphQL in 2012. Their mobile app was making dozens of REST calls per screen on slow 3G networks, fetching enormous amounts of data it did not need. The team needed a way to let clients ask for exactly what they needed in a single request.

The core insight is a fundamental inversion:

```
REST:      server defines the shape of the response
GraphQL:   client defines the shape of the response
```

```graphql
query {
  user(id: "123") {
    name
    email
    orders(last: 5) {
      id
      total
    }
  }
}
```

One request. Exactly those fields. No over-fetching. No N+1.

GraphQL also gives you a strongly typed schema as the contract between client and server. Every field has a type. Required versus optional is explicit. Clients can query the schema itself — full introspection — and tooling like autocomplete and compile-time validation comes for free.

### Where GraphQL genuinely wins

GraphQL shines when you have multiple clients with genuinely different data needs. Your mobile app needs three fields. Your web app needs fifteen. Your smart TV app needs eight. With REST you either build three endpoints or all three clients over-fetch. With GraphQL you build one schema and each client asks for what it needs.

It also dramatically improves frontend velocity. A frontend team can add fields to their queries without waiting for a backend engineer to create a new endpoint. In fast-moving product teams this compounds into a significant competitive advantage.

### The problems nobody talks about until production

**Caching is fundamentally broken.** REST caches at the URL level. GET /users/123 is cached by URL — CDNs, HTTP caches, browser caches all understand this natively. GraphQL sends everything as POST /graphql. POST is not cacheable. The query is in the request body, not the URL. Your CDN cannot cache it. You lose HTTP caching entirely unless you implement persisted queries — a significant additional complexity.

**N+1 does not disappear — it moves to the server.** Consider this query:

```graphql
query {
  users {
    orders {
      items { }
    }
  }
}
```

A naive resolver fetches all users in one query, then for each user fetches their orders, then for each order fetches items. The N+1 problem has moved from the client to the server. You need DataLoader — a batching and caching layer — to solve it. Most teams discover this in production.

**Unbounded query complexity is a security problem.** A client can construct a deeply nested query that is technically valid but computationally catastrophic:

```graphql
query {
  users {
    friends {
      friends {
        friends {
          orders { items { reviews { author { friends } } } }
        }
      }
    }
  }
}
```

This is a denial of service attack via a valid query. You need query depth limiting, complexity analysis, and cost estimation — all custom built. REST never had this problem.

**Error handling is non-standard.** GraphQL returns HTTP 200 OK even when operations fail. Errors live inside the response body. Your monitoring, alerting, and logging tools all assume HTTP status codes encode success and failure. They do not in GraphQL. Standard HTTP tooling breaks silently.

**Observability is harder.** With REST, each endpoint is a distinct URL — you measure latency, error rate, and throughput per endpoint trivially. With GraphQL everything goes to one endpoint. How do you measure latency per operation type? How do you rate limit a specific query? These are solved problems in REST. They require custom instrumentation in GraphQL.

---

## gRPC — When you control both sides

gRPC was built by Google to solve internal service-to-service communication. The mental model shift from REST is significant:

```
REST:   you are manipulating resources
        GET /orders/123

gRPC:   you are calling functions on a remote machine
        orderService.GetOrder(GetOrderRequest{id: "123"})
```

gRPC uses two technologies together: **Protocol Buffers** (Protobuf) as the interface definition language and binary serialization format, and **HTTP/2** as the transport.

### Protobuf — why it changes everything

You define your API in a `.proto` file:

```protobuf
message Order {
    string id = 1;
    string user_id = 2;
    repeated OrderItem items = 3;
    int64 created_at = 4;
}

service OrderService {
    rpc GetOrder(GetOrderRequest) returns (Order);
    rpc StreamOrders(StreamRequest) returns (stream Order);
}
```

From this single file, gRPC generates client code, server stub code, serialization code, and type-safe request and response objects — in any language you target. Python, Go, Java, C++, all from one source of truth.

The wire format is binary. The same data that JSON encodes in roughly forty bytes, Protobuf encodes in roughly fifteen — about sixty percent smaller. Serialization and deserialization is five to ten times faster. At millions of requests per second between internal services, this compounds into enormous infrastructure savings.

Type mismatches are caught at compile time, not in production. A field type changes in the proto file — the build fails. Every consuming service must update. This discipline is what prevents the silent runtime failures that plague REST at scale.

### HTTP/2 and what it actually gives you

gRPC requires HTTP/2. Understanding why helps you understand gRPC's performance characteristics.

HTTP/1.1 opens one request at a time per connection. To parallelize, browsers open multiple connections — typically six per domain. Each connection requires a TCP handshake. This is expensive.

HTTP/2 multiplexes multiple requests over a single TCP connection simultaneously. No head-of-line blocking. No per-request TCP handshake overhead. For microservices making hundreds of calls per second between each other, this is a significant throughput improvement.

HTTP/2 also supports server push and header compression — both of which gRPC leverages.

### Four streaming modes

gRPC supports four communication patterns — this is where it goes well beyond REST:

```
Unary:                  one request → one response (like REST)
Server streaming:       one request → stream of responses
Client streaming:       stream of requests → one response
Bidirectional:          both sides stream simultaneously
```

Bidirectional streaming is genuinely unique. A persistent connection where both client and server push data independently. REST has no equivalent. WebSocket achieves this at the application layer but without gRPC's type safety and code generation.

### Where gRPC breaks

**Browsers cannot use gRPC natively.** Browsers do not expose HTTP/2 framing at the level gRPC requires. gRPC-Web is a workaround but it is limited — it does not support client streaming or bidirectional streaming. For APIs consumed directly by browsers, gRPC is the wrong choice.

**The binary format is not human readable.** You cannot curl a gRPC endpoint and inspect the response. Debugging requires special tooling. In a REST world, a developer can open a terminal and introspect the API immediately. In gRPC they need the proto files and a gRPC client.

**Protobuf field numbers are permanent.** This is the subtlest failure mode. In Protobuf, each field has a number — `string id = 1`. That number is baked into the binary encoding. If you remove field 1 and add a new field with number 1, clients and servers on different versions will silently misread each other's data. Not an error — silent corruption. You can only add fields, never reuse numbers. This discipline requires rigorous schema review processes at scale.

---

## The architecture that uses all three

In practice, mature engineering organizations do not choose one protocol. They use each where it fits:

```
Third-party developers  →  REST API
                           unknown consumers, no tooling requirement,
                           CDN caching, stable versioned surface

Internal microservices  →  gRPC
                           owned both sides, performance matters,
                           compile-time contracts, native streaming

Mobile and web apps     →  GraphQL BFF (Backend for Frontend)
                           different clients need different data shapes,
                           aggregates multiple gRPC services,
                           one optimized layer per client type
```

The critical architectural detail is where GraphQL lives. It does not sit on individual microservices — that would couple your internal service design to client needs. It sits as a BFF layer: a separate service whose only job is to aggregate data from internal gRPC services and present a client-optimized graph.

```
Mobile App   ──→  GraphQL BFF  ──→  gRPC services
Web App      ──→  GraphQL BFF  ──→  gRPC services
3rd party    ──→  REST API     ──→  gRPC services
                      ↑
              API Gateway handles:
              REST → gRPC transcoding
              auth, rate limiting, observability
```

The REST public API does not directly talk to your microservices. An API gateway — using a tool like gRPC-Gateway or Envoy's transcoding filter — maps REST endpoints to gRPC methods using annotations in the proto file. The proto definition remains the single source of truth. The REST API is derived from it.

---

## SSE — Server-sent events over REST

Once you need the server to push data to clients without clients polling, request-response protocols stop being the right tool. The first question to ask is: does the client need to send data back on the same connection?

If the answer is no — the server just needs to push events — SSE is the right choice.

SSE is not a new protocol. It is a long-lived HTTP response that never closes, with a specific content type:

```
GET /order-status/456 HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

id: 1
data: {"status": "confirmed"}

id: 2
data: {"status": "preparing"}

id: 3
data: {"status": "out_for_delivery"}
```

The server keeps writing to the response stream. The client reads events as they arrive. The HTTP connection stays open. That is the entire mechanism.

### Why SSE is underrated

Because it is plain HTTP, SSE works through every proxy, CDN, and firewall that allows HTTP traffic. No protocol upgrade negotiation, no special headers, no compatibility issues.

The reconnection behavior is the most underrated feature. The server assigns an ID to each event. If the client loses network — phone switches from WiFi to 4G, tunnel kills connectivity — the browser automatically reconnects and sends a `Last-Event-ID` header. The server replays all events from that ID onwards. The client never misses an event. This is built into the browser spec. You do not write this logic yourself. WebSocket has no equivalent — you build reconnection and replay entirely from scratch.

The browser's `EventSource` API is native — no library required. Named event types let you route different events to different handlers on the client. It is simpler to implement correctly than WebSocket for the vast majority of push use cases.

Every LLM streaming API — ChatGPT, Claude, Gemini — streams tokens to your browser using SSE. Not WebSocket.

### Where SSE falls short

SSE is receive-only for the client. If you need the client to send data on the same connection — chat, collaborative editing, live location sharing — SSE is insufficient.

On HTTP/1.1, browsers limit connections to six per domain. Each open SSE stream holds one. Multiple SSE streams on the same page can starve other requests. HTTP/2 solves this via multiplexing — all streams share one connection. This is an important operational detail when deploying SSE at scale.

---

## WebSocket — True bidirectional real-time

WebSocket is the right choice when both sides need to push data independently and simultaneously. Chat applications. Collaborative document editing. Multiplayer games. Live trading dashboards where the user interacts while receiving updates.

### The handshake

WebSocket starts as HTTP — deliberately, to traverse firewalls and proxies that only allow HTTP:

```
Client sends:
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZQ==
Sec-WebSocket-Version: 13

Server responds:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the 101 response, HTTP is completely gone. The connection becomes a raw TCP channel with a lightweight framing layer. Both sides can send frames at any time with no headers, no content-type, no cookies — just the frame and the data. This is what makes WebSocket efficient for high-frequency messages.

### The scalability problem

WebSocket's power comes with a fundamental scalability challenge. HTTP is stateless — any server can handle any request, load balancers round-robin freely. WebSocket is stateful — a client is connected to one specific server. That connection lives there for the duration of the session.

This creates the fan-out problem:

```
Client A  →  connected to Server 1
Client B  →  connected to Server 2
Client C  →  connected to Server 1

Message needs to reach all three clients.
Server 2 cannot push to Client A or Client C.
Server 1 cannot push to Client B.
```

The solution is a pub/sub layer — typically Redis Pub/Sub or Kafka — between your WebSocket servers. When Server 2 receives a message, it publishes to Redis. All WebSocket servers subscribe. Each server pushes the message to the clients connected to it. This architecture is non-negotiable at any meaningful scale.

WebSocket also has no built-in reconnection or message replay. If a client disconnects and reconnects, you are responsible for replaying missed messages. If a server crashes, you are responsible for re-establishing the connection. This is significant implementation work that teams frequently underestimate.

---

## The backend trigger problem — gRPC vs message broker

A common architecture question is: when the SSE or WebSocket handler needs to be triggered by a backend service, should that service call the handler directly via gRPC, or should it publish to a message broker?

The answer reveals an important conceptual distinction.

gRPC is request-driven — a consumer asks a producer for data when it needs it. The producer responds. The interaction is initiated by the consumer.

A message broker is event-driven — a producer announces that something happened. It does not know who is listening and does not care. Consumers react independently.

Order status changes are events — things that happened. The OrderService is not responding to a request. It is announcing a fact: "order 456 moved to preparing." That is the event-driven pattern. Publishing to Kafka fits the natural shape of the problem.

gRPC streaming technically works for this — OrderService streams status updates directly to the SSE handler. But it creates tight coupling: the SSE handler is a direct dependent of OrderService. It creates a connection mesh at scale: ten SSE servers and five OrderService instances means fifty persistent gRPC connections to maintain. And it provides no replay capability: if the SSE handler restarts, events during the downtime are gone forever.

With Kafka, OrderService publishes and forgets. SSE handlers consume independently. They scale, deploy, and fail independently. A restarting SSE handler replays from its last offset and catches up on everything it missed.

One important clarification: Kafka protects events after they are published. It does not protect events that were never published. If OrderService goes down before publishing a status change, that event is lost. Protecting the producer side — ensuring the event is published even if the service crashes mid-operation — is a separate problem solved by the Outbox Pattern, which we will cover when we get to distributed systems.

---

## The decision framework

At Staff level, protocol choice is not about preference. It is a structured decision based on the specific constraints of the problem.

**Start with the consumer:**
- Unknown external developers → REST
- Your own clients → GraphQL or gRPC
- Internal services you own → gRPC

**Then ask about directionality:**
- Request-response only → REST, GraphQL, or gRPC
- Server pushes, client listens → SSE
- Both sides push simultaneously → WebSocket

**Then ask about caching:**
- CDN caching critical → REST only
- Caching less important → any protocol fits

**Then ask about performance:**
- High throughput internal services → gRPC
- Standard web traffic → REST or GraphQL fine

**Then ask about streaming:**
- Real-time bidirectional → WebSocket or gRPC bidirectional
- Server-to-client events only → SSE (prefer over WebSocket for simplicity)

When an interviewer asks you to design a system — whether it is an API platform, an order tracking system, a chat application, or a live dashboard — your protocol choices should follow from these questions, not precede them. Stating your reasoning out loud is what signals Staff-level thinking. Anyone can name a protocol. The interview question is always: why this one, and what does it cost you?

---

## Quick reference

| Protocol | HTTP version | Direction | Wire format | Caching | Best for | Breaks when |
|---|---|---|---|---|---|---|
| REST | 1.1 / 2 | req → res | JSON | Native HTTP caching | Public APIs, CRUD, unknown consumers | N+1 on mobile, over-fetching, weak contracts |
| GraphQL | 1.1 / 2 | req → res | JSON | Broken by default (POST) | Multiple clients, BFF layer, graph data | Caching, N+1 server-side, unbounded queries |
| gRPC | HTTP/2 only | req → res + streaming | Protobuf binary | Not cacheable | Internal services, high throughput, polyglot | Browsers, human readability, field number reuse |
| SSE | 1.1 / 2 | server → client | text/event-stream | No | Notifications, feeds, AI streaming, order tracking | Client needs to send data, HTTP/1.1 connection limits |
| WebSocket | 1.1 upgrade → TCP | bidirectional | frames | No | Chat, games, collaborative editing, high-frequency | Horizontal scaling without pub/sub, no built-in replay |

---

<div class="apx">
<style>
@import url('https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,400;12..96,600;12..96,800&family=JetBrains+Mono:wght@400;600&display=swap');
.apx{--ink:#0f131c;--panel:#171c28;--panel2:#1d2330;--line:#2b3343;--tx:#e8ecf4;--mut:#8b95aa;--rest:#f5a623;--gql:#ec4899;--grpc:#22d3ee;--sse:#34d399;--ws:#a78bfa;font-family:'Bricolage Grotesque',ui-sans-serif,system-ui,sans-serif;color:var(--tx);background:var(--ink);border:1px solid var(--line);border-radius:18px;padding:30px 26px 36px;margin:36px 0;line-height:1.5;background-image:radial-gradient(circle at 1px 1px,rgba(255,255,255,.04) 1px,transparent 0);background-size:22px 22px;}
.apx *{box-sizing:border-box;}
.apx .apx-kicker{font-family:'JetBrains Mono',monospace;font-size:11px;letter-spacing:.22em;text-transform:uppercase;color:var(--grpc);margin:0 0 6px;}
.apx h2.apx-title{font-size:30px;line-height:1.05;font-weight:800;margin:0 0 8px;letter-spacing:-.02em;background:linear-gradient(92deg,var(--tx),var(--mut));-webkit-background-clip:text;background-clip:text;color:transparent;}
.apx .apx-sub{color:var(--mut);font-size:14px;margin:0 0 26px;max-width:62ch;}
.apx .apx-block{background:var(--panel);border:1px solid var(--line);border-radius:14px;padding:20px 18px;margin:22px 0;}
.apx .apx-bt{font-family:'JetBrains Mono',monospace;font-size:12px;letter-spacing:.14em;text-transform:uppercase;color:var(--mut);margin:0 0 4px;}
.apx .apx-bh{font-size:18px;font-weight:700;margin:0 0 16px;letter-spacing:-.01em;}
.apx .apx-note{font-size:12.5px;color:var(--mut);margin:14px 0 0;font-style:italic;}
.apx .apx-mono{font-family:'JetBrains Mono',monospace;}
.apx .apx-radio{position:absolute;width:1px;height:1px;opacity:0;pointer-events:none;}
.apx .apx-diagram{display:flex;align-items:center;justify-content:space-between;gap:14px;background:var(--panel2);border:1px solid var(--line);border-radius:12px;padding:18px 16px;position:relative;}
.apx .apx-node{font-family:'JetBrains Mono',monospace;font-size:12px;font-weight:600;padding:12px 8px;border-radius:10px;background:var(--ink);border:1px solid var(--line);text-align:center;min-width:78px;flex:0 0 auto;}
.apx .apx-wire{position:relative;flex:1;height:3px;background:repeating-linear-gradient(90deg,var(--line) 0 7px,transparent 7px 14px);border-radius:2px;}
.apx .apx-pkt{position:absolute;top:50%;width:12px;height:12px;border-radius:50%;margin-top:-6px;left:0;box-shadow:0 0 12px currentColor;}
.apx .apx-lbl{font-family:'JetBrains Mono',monospace;font-size:10.5px;color:var(--mut);text-align:center;margin-top:10px;}
.apx .apx-2col{display:grid;grid-template-columns:1fr 1fr;gap:16px;}
@media(max-width:560px){.apx .apx-2col{grid-template-columns:1fr;}}
.apx .apx-rr .apx-pkt{color:var(--grpc);background:var(--grpc);animation:apxRR 3s linear infinite;}
@keyframes apxRR{0%{left:0;opacity:0;}5%{opacity:1;}45%{left:calc(100% - 12px);opacity:1;}50%{left:calc(100% - 12px);opacity:0;}55%{left:calc(100% - 12px);opacity:0;}60%{opacity:1;}95%{left:0;opacity:1;}100%{left:0;opacity:0;}}
.apx .apx-pp .apx-pkt{color:var(--sse);background:var(--sse);}
.apx .apx-pp .apx-p1{animation:apxPP 2.2s linear infinite;}
.apx .apx-pp .apx-p2{animation:apxPP 2.2s linear infinite;animation-delay:.73s;}
.apx .apx-pp .apx-p3{animation:apxPP 2.2s linear infinite;animation-delay:1.46s;}
@keyframes apxPP{0%{left:calc(100% - 12px);opacity:0;}8%{opacity:1;}92%{opacity:1;}100%{left:0;opacity:0;}}
.apx .apx-tabs{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:16px;}
.apx .apx-tab{font-family:'JetBrains Mono',monospace;font-size:12px;font-weight:600;padding:9px 14px;border-radius:9px;background:var(--panel2);border:1px solid var(--line);color:var(--mut);cursor:pointer;transition:all .15s;user-select:none;}
.apx .apx-tab:hover{color:var(--tx);border-color:var(--mut);}
.apx .apx-pane{display:none;}
.apx #apxp-rest:checked~.apx-tabs label[for=apxp-rest]{background:var(--rest);color:#1a1205;border-color:var(--rest);}
.apx #apxp-gql:checked~.apx-tabs label[for=apxp-gql]{background:var(--gql);color:#fff;border-color:var(--gql);}
.apx #apxp-grpc:checked~.apx-tabs label[for=apxp-grpc]{background:var(--grpc);color:#04282b;border-color:var(--grpc);}
.apx #apxp-sse:checked~.apx-tabs label[for=apxp-sse]{background:var(--sse);color:#052b1c;border-color:var(--sse);}
.apx #apxp-ws:checked~.apx-tabs label[for=apxp-ws]{background:var(--ws);color:#1c0f3a;border-color:var(--ws);}
.apx #apxp-rest:checked~.apx-pane-rest,.apx #apxp-gql:checked~.apx-pane-gql,.apx #apxp-grpc:checked~.apx-pane-grpc,.apx #apxp-sse:checked~.apx-pane-sse,.apx #apxp-ws:checked~.apx-pane-ws{display:block;}
.apx .apx-card{border-left:3px solid var(--line);padding:4px 0 4px 16px;}
.apx .apx-card.r{border-color:var(--rest);}.apx .apx-card.g{border-color:var(--gql);}.apx .apx-card.c{border-color:var(--grpc);}.apx .apx-card.s{border-color:var(--sse);}.apx .apx-card.v{border-color:var(--ws);}
.apx .apx-cn{font-size:20px;font-weight:800;margin:0 0 2px;}
.apx .apx-cd{font-size:13.5px;color:var(--mut);margin:0 0 14px;}
.apx .apx-meta{display:flex;flex-wrap:wrap;gap:7px;margin:0 0 14px;}
.apx .apx-chip{font-family:'JetBrains Mono',monospace;font-size:10.5px;padding:4px 9px;border-radius:20px;background:var(--ink);border:1px solid var(--line);color:var(--mut);}
.apx .apx-ww{display:grid;grid-template-columns:1fr 1fr;gap:14px;}
@media(max-width:560px){.apx .apx-ww{grid-template-columns:1fr;}}
.apx .apx-win,.apx .apx-brk{font-size:12.5px;}
.apx .apx-h{font-family:'JetBrains Mono',monospace;font-size:10.5px;letter-spacing:.1em;text-transform:uppercase;margin:0 0 7px;}
.apx .apx-win .apx-h{color:var(--sse);}.apx .apx-brk .apx-h{color:#fb7185;}
.apx .apx-li{display:flex;gap:8px;margin:0 0 7px;color:#cdd4e1;line-height:1.4;}
.apx .apx-li::before{content:'';flex:0 0 auto;width:6px;height:6px;border-radius:50%;margin-top:6px;}
.apx .apx-win .apx-li::before{background:var(--sse);}.apx .apx-brk .apx-li::before{background:#fb7185;}
.apx .apx-of-wrap{display:grid;grid-template-columns:1fr 1fr;gap:16px;}
@media(max-width:560px){.apx .apx-of-wrap{grid-template-columns:1fr;}}
.apx .apx-payload{font-family:'JetBrains Mono',monospace;font-size:11.5px;background:var(--ink);border:1px solid var(--line);border-radius:10px;padding:14px;min-height:230px;}
.apx .apx-pt{font-size:10px;letter-spacing:.1em;text-transform:uppercase;color:var(--mut);margin:0 0 10px;display:flex;justify-content:space-between;}
.apx .apx-fld{padding:3px 6px;border-radius:4px;margin:2px 0;transition:all .3s;}
.apx .apx-need{color:var(--sse);background:rgba(52,211,153,.1);}
.apx .apx-waste{color:#55607a;}
.apx #apxof:checked~.apx-of-wrap .apx-rest-payload .apx-waste{opacity:.28;text-decoration:line-through;}
.apx .apx-toggle{display:inline-flex;align-items:center;gap:10px;font-family:'JetBrains Mono',monospace;font-size:12px;cursor:pointer;margin-bottom:16px;user-select:none;color:var(--mut);}
.apx .apx-switch{width:42px;height:23px;border-radius:20px;background:var(--line);position:relative;transition:.2s;flex:0 0 auto;}
.apx .apx-switch::after{content:'';position:absolute;top:2px;left:2px;width:19px;height:19px;border-radius:50%;background:var(--tx);transition:.2s;}
.apx #apxof:checked~.apx-toggle .apx-switch{background:var(--sse);}
.apx #apxof:checked~.apx-toggle .apx-switch::after{left:21px;background:#052b1c;}
.apx #apxof:checked~.apx-toggle .apx-on{color:var(--sse);}
.apx .apx-stream{display:grid;gap:12px;}
.apx .apx-srow{display:grid;grid-template-columns:120px 1fr;gap:14px;align-items:center;background:var(--panel2);border:1px solid var(--line);border-radius:10px;padding:12px 14px;}
@media(max-width:560px){.apx .apx-srow{grid-template-columns:1fr;gap:8px;}}
.apx .apx-sname{font-family:'JetBrains Mono',monospace;font-size:12px;font-weight:600;color:var(--grpc);}
.apx .apx-sdesc{font-size:11px;color:var(--mut);margin-top:2px;}
.apx .apx-track{position:relative;height:34px;display:flex;align-items:center;}
.apx .apx-track .apx-wire{background:repeating-linear-gradient(90deg,var(--line) 0 6px,transparent 6px 12px);}
.apx .apx-d{position:absolute;width:10px;height:10px;border-radius:50%;top:50%;margin-top:-5px;background:var(--grpc);color:var(--grpc);box-shadow:0 0 10px currentColor;}
.apx .apx-fwd{animation:apxFwd 2.6s linear infinite;}
.apx .apx-bwd{animation:apxBwd 2.6s linear infinite;background:var(--ws);color:var(--ws);}
@keyframes apxFwd{0%{left:0;opacity:0;}10%{opacity:1;}90%{opacity:1;}100%{left:calc(100% - 10px);opacity:0;}}
@keyframes apxBwd{0%{left:calc(100% - 10px);opacity:0;}10%{opacity:1;}90%{opacity:1;}100%{left:0;opacity:0;}}
.apx .apx-d2{animation-delay:.55s;}.apx .apx-d3{animation-delay:1.1s;}.apx .apx-d4{animation-delay:1.65s;}
.apx .apx-tree .apx-q{font-family:'JetBrains Mono',monospace;font-size:13px;font-weight:600;color:var(--tx);margin:0 0 12px;}
.apx .apx-choices{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:4px;}
.apx .apx-choice{font-size:13px;padding:11px 15px;border-radius:10px;background:var(--panel2);border:1px solid var(--line);color:var(--mut);cursor:pointer;transition:.15s;user-select:none;flex:1 1 160px;text-align:center;}
.apx .apx-choice:hover{color:var(--tx);border-color:var(--mut);transform:translateY(-1px);}
.apx .apx-branch,.apx .apx-leaf{display:none;animation:apxFade .35s ease;}
@keyframes apxFade{from{opacity:0;transform:translateY(6px);}to{opacity:1;transform:none;}}
.apx .apx-branch{margin-top:18px;padding-top:18px;border-top:1px dashed var(--line);}
.apx #apxq1-rr:checked~.apx-b-rr,.apx #apxq1-sse:checked~.apx-b-sse,.apx #apxq1-ws:checked~.apx-b-ws{display:block;}
.apx #apxq1-rr:checked~.apx-choices label[for=apxq1-rr],.apx #apxq1-sse:checked~.apx-choices label[for=apxq1-sse],.apx #apxq1-ws:checked~.apx-choices label[for=apxq1-ws]{background:var(--grpc);color:#04282b;border-color:var(--grpc);}
.apx #apxq2-ext:checked~.apx-l-ext,.apx #apxq2-mul:checked~.apx-l-mul,.apx #apxq2-int:checked~.apx-l-int{display:block;}
.apx #apxq2-ext:checked~.apx-choices label[for=apxq2-ext],.apx #apxq2-mul:checked~.apx-choices label[for=apxq2-mul],.apx #apxq2-int:checked~.apx-choices label[for=apxq2-int]{background:var(--tx);color:var(--ink);border-color:var(--tx);}
.apx .apx-leaf{margin-top:14px;border-radius:10px;padding:16px;background:var(--ink);}
.apx .apx-pick{font-family:'JetBrains Mono',monospace;font-size:11px;letter-spacing:.1em;text-transform:uppercase;color:var(--mut);margin:0 0 4px;}
.apx .apx-pickn{font-size:22px;font-weight:800;margin:0 0 8px;}
.apx .apx-pickr{font-size:13px;color:#cdd4e1;margin:0;line-height:1.45;}
.apx .apx-arch{display:grid;gap:0;}
.apx .apx-tier{display:grid;grid-template-columns:repeat(3,1fr);gap:12px;}
@media(max-width:560px){.apx .apx-tier{grid-template-columns:1fr;}}
.apx .apx-abox{border:1px solid var(--line);border-radius:11px;padding:13px 12px;text-align:center;background:var(--panel2);transition:.18s;}
.apx .apx-abox:hover{transform:translateY(-2px);}
.apx .apx-an{font-family:'JetBrains Mono',monospace;font-size:12px;font-weight:700;margin:0 0 3px;}
.apx .apx-ad{font-size:10.5px;color:var(--mut);line-height:1.35;}
.apx .apx-abox.client{border-color:var(--line);}
.apx .apx-flow{text-align:center;color:var(--mut);font-family:'JetBrains Mono',monospace;font-size:18px;padding:8px 0;}
.apx .apx-gw{border:1px dashed var(--grpc);border-radius:11px;padding:11px;text-align:center;background:rgba(34,211,238,.06);margin:6px 0;}
.apx .apx-gw .apx-an{color:var(--grpc);}
.apx .apx-fan{display:grid;grid-template-columns:1fr 1fr;gap:18px;}
@media(max-width:560px){.apx .apx-fan{grid-template-columns:1fr;}}
.apx .apx-fcol .apx-h{margin-bottom:10px;}
.apx .apx-fcol.bad .apx-h{color:#fb7185;}.apx .apx-fcol.good .apx-h{color:var(--sse);}
.apx .apx-srv{font-family:'JetBrains Mono',monospace;font-size:11px;border:1px solid var(--line);border-radius:8px;padding:9px 10px;margin-bottom:8px;background:var(--ink);}
.apx .apx-cl{display:inline-block;font-family:'JetBrains Mono',monospace;font-size:10px;padding:3px 7px;border-radius:5px;background:var(--panel2);margin:3px 3px 0 0;}
.apx .apx-bus{font-family:'JetBrains Mono',monospace;font-size:11px;text-align:center;border:1px solid var(--sse);color:var(--sse);border-radius:8px;padding:9px;margin:8px 0;background:rgba(52,211,153,.07);}
</style>
<p class="apx-kicker">Interactive Appendix</p>
<h2 class="apx-title">See the protocols in motion</h2>
<p class="apx-sub">Six interactive visuals that turn the ideas above into something you can poke at. Everything below is pure HTML and CSS — tap, toggle, and hover to explore. No two protocols solve the same problem the same way; these make the differences obvious.</p>
<div class="apx-block">
<p class="apx-bt">Visual 01 · Foundations</p>
<p class="apx-bh">The two shapes of communication</p>
<div class="apx-2col">
<div>
<div class="apx-diagram apx-rr">
<div class="apx-node">Client</div>
<div class="apx-wire"><span class="apx-pkt"></span></div>
<div class="apx-node">Server</div>
</div>
<p class="apx-lbl">REQUEST → RESPONSE · ask, answer, done (REST · GraphQL · gRPC unary)</p>
</div>
<div>
<div class="apx-diagram apx-pp">
<div class="apx-node">Client</div>
<div class="apx-wire"><span class="apx-pkt apx-p1"></span><span class="apx-pkt apx-p2"></span><span class="apx-pkt apx-p3"></span></div>
<div class="apx-node">Server</div>
</div>
<p class="apx-lbl">PERSISTENT PUSH · connection stays open, server emits (SSE · WebSocket)</p>
</div>
</div>
<p class="apx-note">Every other decision in this post hangs off which of these two shapes your problem actually is.</p>
</div>
<div class="apx-block">
<p class="apx-bt">Visual 02 · Reference</p>
<p class="apx-bh">Protocol explorer — pick one to compare</p>
<input class="apx-radio" type="radio" name="apxpane" id="apxp-rest" checked>
<input class="apx-radio" type="radio" name="apxpane" id="apxp-gql">
<input class="apx-radio" type="radio" name="apxpane" id="apxp-grpc">
<input class="apx-radio" type="radio" name="apxpane" id="apxp-sse">
<input class="apx-radio" type="radio" name="apxpane" id="apxp-ws">
<div class="apx-tabs">
<label class="apx-tab" for="apxp-rest">REST</label>
<label class="apx-tab" for="apxp-gql">GraphQL</label>
<label class="apx-tab" for="apxp-grpc">gRPC</label>
<label class="apx-tab" for="apxp-sse">SSE</label>
<label class="apx-tab" for="apxp-ws">WebSocket</label>
</div>
<div class="apx-pane apx-pane-rest">
<div class="apx-card r">
<p class="apx-cn">REST</p>
<p class="apx-cd">Manipulate resources by URL. The lingua franca of public APIs.</p>
<div class="apx-meta"><span class="apx-chip">HTTP 1.1 / 2</span><span class="apx-chip">req → res</span><span class="apx-chip">JSON</span><span class="apx-chip">native HTTP caching</span></div>
<div class="apx-ww">
<div class="apx-win"><p class="apx-h">Wins when</p><div class="apx-li">Consumers are unknown &amp; diverse — any HTTP client works</div><div class="apx-li">CDN caching matters; a CDN absorbs 60–80% of traffic</div><div class="apx-li">Stateless scaling — any server handles any request</div></div>
<div class="apx-brk"><p class="apx-h">Breaks when</p><div class="apx-li">Over-fetching — 40 fields shipped, 3 needed</div><div class="apx-li">N+1 round trips wreck mobile latency</div><div class="apx-li">Weak contracts surface as production runtime errors</div></div>
</div>
</div>
</div>
<div class="apx-pane apx-pane-gql">
<div class="apx-card g">
<p class="apx-cn">GraphQL</p>
<p class="apx-cd">Client defines the response shape. Built at Facebook for hungry mobile screens.</p>
<div class="apx-meta"><span class="apx-chip">HTTP 1.1 / 2</span><span class="apx-chip">req → res</span><span class="apx-chip">JSON</span><span class="apx-chip">caching broken (POST)</span></div>
<div class="apx-ww">
<div class="apx-win"><p class="apx-h">Wins when</p><div class="apx-li">Many clients, genuinely different data needs</div><div class="apx-li">One typed schema is the contract for all of them</div><div class="apx-li">Frontend velocity — add fields without new endpoints</div></div>
<div class="apx-brk"><p class="apx-h">Breaks when</p><div class="apx-li">POST kills URL-level CDN caching</div><div class="apx-li">N+1 moves server-side; needs DataLoader</div><div class="apx-li">Unbounded nested queries = DoS via valid query</div></div>
</div>
</div>
</div>
<div class="apx-pane apx-pane-grpc">
<div class="apx-card c">
<p class="apx-cn">gRPC</p>
<p class="apx-cd">Call functions on a remote machine. Protobuf over HTTP/2 for services you own.</p>
<div class="apx-meta"><span class="apx-chip">HTTP/2 only</span><span class="apx-chip">req → res + streaming</span><span class="apx-chip">Protobuf binary</span><span class="apx-chip">not cacheable</span></div>
<div class="apx-ww">
<div class="apx-win"><p class="apx-h">Wins when</p><div class="apx-li">Internal services, both sides owned</div><div class="apx-li">~60% smaller payloads, 5–10× faster (de)serialization</div><div class="apx-li">Compile-time contracts; four streaming modes</div></div>
<div class="apx-brk"><p class="apx-h">Breaks when</p><div class="apx-li">Browsers can't speak it natively</div><div class="apx-li">Binary wire format — not curl-able</div><div class="apx-li">Reusing Protobuf field numbers = silent corruption</div></div>
</div>
</div>
</div>
<div class="apx-pane apx-pane-sse">
<div class="apx-card s">
<p class="apx-cn">SSE</p>
<p class="apx-cd">A long-lived HTTP response that never closes. The quiet workhorse of streaming.</p>
<div class="apx-meta"><span class="apx-chip">HTTP 1.1 / 2</span><span class="apx-chip">server → client</span><span class="apx-chip">text/event-stream</span><span class="apx-chip">no caching</span></div>
<div class="apx-ww">
<div class="apx-win"><p class="apx-h">Wins when</p><div class="apx-li">Plain HTTP — passes every proxy &amp; firewall</div><div class="apx-li">Auto-reconnect + replay via Last-Event-ID, free</div><div class="apx-li">Native EventSource; powers LLM token streaming</div></div>
<div class="apx-brk"><p class="apx-h">Breaks when</p><div class="apx-li">Client also needs to send on the same connection</div><div class="apx-li">HTTP/1.1's six-connection-per-domain limit</div></div>
</div>
</div>
</div>
<div class="apx-pane apx-pane-ws">
<div class="apx-card v">
<p class="apx-cn">WebSocket</p>
<p class="apx-cd">Upgrades HTTP into a raw two-way channel. For when both sides talk at once.</p>
<div class="apx-meta"><span class="apx-chip">HTTP/1.1 upgrade → TCP</span><span class="apx-chip">bidirectional</span><span class="apx-chip">frames</span><span class="apx-chip">no caching</span></div>
<div class="apx-ww">
<div class="apx-win"><p class="apx-h">Wins when</p><div class="apx-li">Chat, multiplayer, collaborative editing</div><div class="apx-li">High-frequency, low-overhead frames both ways</div><div class="apx-li">Live dashboards where users act while receiving</div></div>
<div class="apx-brk"><p class="apx-h">Breaks when</p><div class="apx-li">Stateful — horizontal scaling needs pub/sub</div><div class="apx-li">No built-in reconnect or replay; you build it</div></div>
</div>
</div>
</div>
</div>
<div class="apx-block">
<p class="apx-bt">Visual 03 · The over-fetching problem</p>
<p class="apx-bh">Same screen, two payloads</p>
<input class="apx-radio" type="checkbox" id="apxof">
<label class="apx-toggle" for="apxof"><span class="apx-switch"></span><span>REST response</span><span>→</span><span class="apx-on">dim the wasted fields</span></label>
<div class="apx-of-wrap">
<div class="apx-payload apx-rest-payload">
<p class="apx-pt"><span>GET /users/123 · REST</span><span>40 fields</span></p>
<div class="apx-fld apx-need">name: "Udit"</div>
<div class="apx-fld apx-need">avatar: "/img/u.png"</div>
<div class="apx-fld apx-need">status: "online"</div>
<div class="apx-fld apx-waste">email, phone, address_line_1,</div>
<div class="apx-fld apx-waste">address_line_2, city, region,</div>
<div class="apx-fld apx-waste">postal_code, country, timezone,</div>
<div class="apx-fld apx-waste">locale, created_at, updated_at,</div>
<div class="apx-fld apx-waste">last_login, preferences{...},</div>
<div class="apx-fld apx-waste">billing{...}, +22 more fields</div>
</div>
<div class="apx-payload">
<p class="apx-pt"><span>{ user } · GraphQL</span><span>3 fields</span></p>
<div class="apx-fld apx-need">name: "Udit"</div>
<div class="apx-fld apx-need">avatar: "/img/u.png"</div>
<div class="apx-fld apx-need">status: "online"</div>
<div style="margin-top:14px;color:#55607a;font-size:11px;">The client asked for exactly three<br>fields. Nothing else crosses the wire.</div>
</div>
</div>
<p class="apx-note">Flip the toggle: the mobile client needed three fields, but REST shipped all forty. Multiply by millions of requests a day.</p>
</div>
<div class="apx-block">
<p class="apx-bt">Visual 04 · gRPC streaming</p>
<p class="apx-bh">Four communication patterns</p>
<div class="apx-stream">
<div class="apx-srow">
<div><div class="apx-sname">Unary</div><div class="apx-sdesc">1 req → 1 res</div></div>
<div class="apx-track"><div class="apx-wire"><span class="apx-d apx-fwd"></span></div></div>
</div>
<div class="apx-srow">
<div><div class="apx-sname">Server stream</div><div class="apx-sdesc">1 req → many res</div></div>
<div class="apx-track"><div class="apx-wire"><span class="apx-d apx-bwd"></span><span class="apx-d apx-bwd apx-d2"></span><span class="apx-d apx-bwd apx-d3"></span><span class="apx-d apx-bwd apx-d4"></span></div></div>
</div>
<div class="apx-srow">
<div><div class="apx-sname">Client stream</div><div class="apx-sdesc">many req → 1 res</div></div>
<div class="apx-track"><div class="apx-wire"><span class="apx-d apx-fwd"></span><span class="apx-d apx-fwd apx-d2"></span><span class="apx-d apx-fwd apx-d3"></span><span class="apx-d apx-fwd apx-d4"></span></div></div>
</div>
<div class="apx-srow">
<div><div class="apx-sname">Bidirectional</div><div class="apx-sdesc">both stream at once</div></div>
<div class="apx-track"><div class="apx-wire"><span class="apx-d apx-fwd"></span><span class="apx-d apx-fwd apx-d3"></span><span class="apx-d apx-bwd apx-d2"></span><span class="apx-d apx-bwd apx-d4"></span></div></div>
</div>
</div>
<p class="apx-note">Cyan dots flow client → server, violet dots server → client. Bidirectional is the mode REST simply has no answer for.</p>
</div>
<div class="apx-block">
<p class="apx-bt">Visual 05 · The decision framework</p>
<p class="apx-bh">Build the path — answer to reveal the pick</p>
<div class="apx-tree">
<p class="apx-q">1 · What does the communication look like?</p>
<input class="apx-radio" type="radio" name="apxq1" id="apxq1-rr">
<input class="apx-radio" type="radio" name="apxq1" id="apxq1-sse">
<input class="apx-radio" type="radio" name="apxq1" id="apxq1-ws">
<div class="apx-choices">
<label class="apx-choice" for="apxq1-rr">Ask &amp; answer<br><span style="font-size:11px;color:var(--mut)">request → response</span></label>
<label class="apx-choice" for="apxq1-sse">Server pushes<br><span style="font-size:11px;color:var(--mut)">client only listens</span></label>
<label class="apx-choice" for="apxq1-ws">Both push<br><span style="font-size:11px;color:var(--mut)">simultaneously</span></label>
</div>
<div class="apx-branch apx-b-rr">
<p class="apx-q">2 · Who consumes it?</p>
<input class="apx-radio" type="radio" name="apxq2" id="apxq2-ext">
<input class="apx-radio" type="radio" name="apxq2" id="apxq2-mul">
<input class="apx-radio" type="radio" name="apxq2" id="apxq2-int">
<div class="apx-choices">
<label class="apx-choice" for="apxq2-ext">Unknown external devs</label>
<label class="apx-choice" for="apxq2-mul">Your own apps, varied needs</label>
<label class="apx-choice" for="apxq2-int">Internal services you own</label>
</div>
<div class="apx-leaf apx-l-ext"><p class="apx-pick">Recommended</p><p class="apx-pickn" style="color:var(--rest)">REST</p><p class="apx-pickr">Anyone with an HTTP client can consume it, CDN caching comes for free, and statelessness makes horizontal scaling trivial. The default for a public, versioned surface.</p></div>
<div class="apx-leaf apx-l-mul"><p class="apx-pick">Recommended</p><p class="apx-pickn" style="color:var(--gql)">GraphQL (as a BFF)</p><p class="apx-pickr">One typed schema lets mobile, web, and TV each ask for exactly their fields. Budget for broken caching, server-side N+1 (DataLoader), and query-complexity limits.</p></div>
<div class="apx-leaf apx-l-int"><p class="apx-pick">Recommended</p><p class="apx-pickn" style="color:var(--grpc)">gRPC</p><p class="apx-pickr">You own both sides, so trade human-readability for binary speed, compile-time contracts, and native streaming. Not for direct browser consumption.</p></div>
</div>
<div class="apx-branch apx-b-sse"><div class="apx-leaf" style="display:block"><p class="apx-pick">Recommended</p><p class="apx-pickn" style="color:var(--sse)">SSE</p><p class="apx-pickr">Plain HTTP that streams events one way, with free auto-reconnect and replay via Last-Event-ID. Prefer it over WebSocket whenever the client only needs to listen — notifications, feeds, order tracking, LLM tokens.</p></div></div>
<div class="apx-branch apx-b-ws"><div class="apx-leaf" style="display:block"><p class="apx-pick">Recommended</p><p class="apx-pickn" style="color:var(--ws)">WebSocket</p><p class="apx-pickr">The only fit when both sides push at once — chat, multiplayer, collaborative editing. Accept the cost: a pub/sub layer (Redis/Kafka) for fan-out and hand-rolled reconnection.</p></div></div>
</div>
<p class="apx-note">The point isn't the answer — it's that the answer falls out of the constraints. Naming a protocol is easy; stating why, and what it costs you, is the Staff-level move.</p>
</div>
<div class="apx-block">
<p class="apx-bt">Visual 06 · Putting it together</p>
<p class="apx-bh">The architecture that uses all three</p>
<div class="apx-arch">
<div class="apx-tier">
<div class="apx-abox client"><div class="apx-an" style="color:var(--ws)">Mobile app</div><div class="apx-ad">varied data needs</div></div>
<div class="apx-abox client"><div class="apx-an" style="color:var(--ws)">Web app</div><div class="apx-ad">varied data needs</div></div>
<div class="apx-abox client"><div class="apx-an" style="color:var(--rest)">3rd-party dev</div><div class="apx-ad">unknown consumer</div></div>
</div>
<div class="apx-flow">↓</div>
<div class="apx-tier">
<div class="apx-abox" style="border-color:var(--gql)"><div class="apx-an" style="color:var(--gql)">GraphQL BFF</div><div class="apx-ad">one optimized graph per client</div></div>
<div class="apx-abox" style="border-color:var(--gql)"><div class="apx-an" style="color:var(--gql)">GraphQL BFF</div><div class="apx-ad">aggregates internal services</div></div>
<div class="apx-abox" style="border-color:var(--rest)"><div class="apx-an" style="color:var(--rest)">REST API</div><div class="apx-ad">stable, versioned, cacheable</div></div>
</div>
<div class="apx-flow">↓</div>
<div class="apx-gw"><div class="apx-an">API Gateway</div><div class="apx-ad">REST → gRPC transcoding · auth · rate limiting · observability</div></div>
<div class="apx-flow">↓</div>
<div class="apx-tier">
<div class="apx-abox" style="border-color:var(--grpc)"><div class="apx-an" style="color:var(--grpc)">gRPC service</div><div class="apx-ad">orders</div></div>
<div class="apx-abox" style="border-color:var(--grpc)"><div class="apx-an" style="color:var(--grpc)">gRPC service</div><div class="apx-ad">users</div></div>
<div class="apx-abox" style="border-color:var(--grpc)"><div class="apx-an" style="color:var(--grpc)">gRPC service</div><div class="apx-ad">inventory</div></div>
</div>
</div>
<p class="apx-note">GraphQL lives as a BFF — never bolted onto a microservice. The proto file stays the single source of truth; the REST surface is derived from it.</p>
<div style="margin-top:22px;padding-top:20px;border-top:1px dashed var(--line)">
<p class="apx-bt" style="color:var(--ws)">Bonus · WebSocket at scale</p>
<p class="apx-bh" style="font-size:15px">Why fan-out needs pub/sub</p>
<div class="apx-fan">
<div class="apx-fcol bad">
<p class="apx-h">Without pub/sub — broken</p>
<div class="apx-srv">Server 1 <span class="apx-cl">Client A</span><span class="apx-cl">Client C</span></div>
<div class="apx-srv">Server 2 <span class="apx-cl">Client B</span></div>
<p style="font-size:11px;color:#fb7185;margin:8px 0 0;line-height:1.4">A message at Server 2 can't reach A or C — they live on Server 1.</p>
</div>
<div class="apx-fcol good">
<p class="apx-h">With pub/sub — works</p>
<div class="apx-srv">Server 1 <span class="apx-cl">Client A</span><span class="apx-cl">Client C</span></div>
<div class="apx-bus">↑↓ Redis / Kafka bus ↑↓</div>
<div class="apx-srv">Server 2 <span class="apx-cl">Client B</span></div>
<p style="font-size:11px;color:var(--sse);margin:8px 0 0;line-height:1.4">Server 2 publishes; every server pushes to its own clients.</p>
</div>
</div>
</div>
</div>
</div>
