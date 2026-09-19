gRPC is a binary RPC (Remote Procedure Call) framework built on HTTP/2 and Protocol Buffers, designed for fast, strongly-typed communication between services — most commonly internal service-to-service calls within a system, rather than a public-facing API. Where REST models an API as resources and GraphQL as a queryable graph, gRPC models it as a set of remote functions with a strict, compiled contract: you call a method, not a URL.

## 1. Protocol Buffers: The Contract and the Wire Format

Every gRPC service is defined in a `.proto` file — a language-neutral schema that describes the service's methods and message types. This same file is compiled into client and server code in whatever languages you need, so the client and server are always working from an identical, generated contract.

```protobuf
syntax = "proto3";

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc PlaceOrder (PlaceOrderRequest) returns (Order);
  rpc StreamOrderUpdates (OrderUpdatesRequest) returns (stream OrderUpdate);
}

message GetOrderRequest {
  string order_id = 1;
}

message Order {
  string id = 1;
  string status = 2;
  repeated OrderItem items = 3;
}

message OrderItem {
  string sku = 1;
  int32 quantity = 2;
}
```

The generated Java server implementation and client stub both come from compiling this same file — there's no hand-written serialization/deserialization code to keep in sync, and no risk of the client and server silently disagreeing about a field's type, unlike a hand-maintained REST/JSON contract.

```java
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {
    @Override
    public void getOrder(GetOrderRequest request, StreamObserver<Order> responseObserver) {
        Order order = orderRepository.findById(request.getOrderId());
        responseObserver.onNext(order);
        responseObserver.onCompleted();
    }
}
```

## 2. Why It's Fast: Binary Encoding + HTTP/2

Two things combine to make gRPC significantly faster and smaller on the wire than typical JSON-over-HTTP/1.1:

| Factor | REST + JSON | gRPC + Protobuf |
| --- | --- | --- |
| Payload format | Text (JSON) — verbose, field names repeated in every message | Binary (Protobuf) — compact, field numbers replace names |
| Transport | Usually HTTP/1.1 — one request per connection at a time (per browser connection limits) | HTTP/2 — multiplexed: many concurrent requests over a single connection |
| Parsing cost | Text parsing (JSON) | Direct binary deserialization, no text parsing |

Protobuf's binary format alone is typically several times smaller than the equivalent JSON, and HTTP/2 multiplexing means many concurrent gRPC calls share one TCP connection without head-of-line blocking at the connection level — both matter most under high call volume between internal services, less so for a browser-facing API where JSON's human-readability and universal tooling support outweigh the size difference.

## 3. The Four RPC Types

gRPC supports more than simple request/response — HTTP/2's native support for streaming lets it express four distinct communication patterns from one framework.

| Type | Shape | Example use case |
| --- | --- | --- |
| Unary | One request, one response | `GetOrder(id) → Order` — the common case, equivalent to a typical REST call |
| Server streaming | One request, a stream of responses | `StreamOrderUpdates(orderId) → stream OrderUpdate` — subscribe once, receive many updates over time |
| Client streaming | A stream of requests, one response | Uploading a large file in chunks, server acknowledges once at the end |
| Bidirectional streaming | Both sides stream independently | A live chat, or a continuous telemetry feed with two-way acknowledgment |

```java
@Override
public void streamOrderUpdates(OrderUpdatesRequest request, StreamObserver<OrderUpdate> responseObserver) {
    orderEventBus.subscribe(request.getOrderId(), update -> {
        responseObserver.onNext(update); // push each update as it happens, no client polling
    });
}
```

Server streaming in particular is a common fit for internal service-to-service "notify me of ongoing changes" needs, without the client having to poll or maintain its own separate WebSocket infrastructure.

## 4. gRPC vs. REST vs. GraphQL

| | REST | GraphQL | gRPC |
| --- | --- | --- | --- |
| Contract | Loose (OpenAPI is optional, not enforced) | Strict, schema-first, introspectable | Strict, compiled from `.proto`, code-generated |
| Payload | Text (JSON), human-readable | Text (JSON), human-readable | Binary (Protobuf), not human-readable |
| Typical audience | Public APIs, browser clients | Public/internal APIs serving varied client shapes | Internal service-to-service calls |
| Streaming | Not native (needs SSE/WebSockets separately) | Subscriptions (typically over WebSockets) | Native, all four RPC types built in |
| Browser support | Native | Native | Needs gRPC-Web (a proxying layer) — not natively callable from a browser |
| Tooling/debuggability | `curl`, browser devtools, universal | GraphQL Playground/GraphiQL | Requires gRPC-aware tooling (`grpcurl`, BloomRPC) — not readable in a plain browser network tab |

gRPC's binary format and mandatory schema are exactly the tradeoff: a REST or GraphQL request can be read and debugged directly in a browser's network tab or with `curl`; a gRPC request cannot, without dedicated tooling. This is a real cost, and part of why gRPC is much more common for internal service-to-service traffic than for a public-facing API.

## 5. Why gRPC Fits Internal Microservices Communication So Well

Internal calls between a company's own services are exactly the setting where gRPC's tradeoffs pay off cleanly: both sides are controlled by teams who can regenerate client/server code from the same `.proto` file, the audience is never a browser needing native fetch/JSON support, and the traffic volume between services (especially high-fan-out calls, like a gateway calling several backend services per incoming request) benefits the most from Protobuf's smaller payloads and HTTP/2's multiplexing. Public APIs, by contrast, need to support arbitrary external clients (including browsers) with minimal friction — which is exactly where REST or GraphQL's plain-text, universally-tooled nature wins out.

## 6. Deadlines and Cancellation

gRPC has first-class support for propagating a deadline (an absolute point in time by which the call must complete) across a chain of calls, rather than each service independently guessing a reasonable timeout.

```java
OrderServiceGrpc.OrderServiceBlockingStub stub = OrderServiceGrpc.newBlockingStub(channel)
    .withDeadlineAfter(2, TimeUnit.SECONDS);

Order order = stub.getOrder(request); // fails with DEADLINE_EXCEEDED if not completed in time
```

If `ServiceA` calls `ServiceB` which calls `ServiceC`, a deadline set at `ServiceA` propagates through the whole chain — `ServiceC` can see how much time is actually left for the overall request, rather than each hop applying its own disconnected timeout that adds up to far more total latency than the original caller was willing to wait.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Reserve gRPC for internal service-to-service calls | Its binary format and lack of native browser support make it a poor fit for a public-facing/browser-facing API. |
| Version `.proto` messages additively | Add new fields with new numbers; never reuse or renumber an existing field number, since old and new binaries must stay wire-compatible. |
| Set and propagate deadlines on every call | Prevents a slow chain of internal calls from silently exceeding what the original caller was willing to wait for. |
| Use server streaming for "notify me of ongoing changes" needs | Avoids client-side polling or standing up separate WebSocket infrastructure just for internal service notifications. |
| Keep `.proto` files as the single source of truth | Regenerate client/server code from them rather than hand-maintaining serialization logic that can drift out of sync. |
| Use gRPC-Web only when a browser genuinely must call a gRPC service directly | Adds a proxying layer and complexity — reconsider whether REST/GraphQL is simply the better fit for that specific audience. |
| Don't remove or renumber Protobuf fields, only deprecate them | A removed/reused field number can silently corrupt data for any client still running older generated code. |
