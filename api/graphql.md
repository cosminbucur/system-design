GraphQL is a query language for APIs where the client specifies exactly which fields it needs in a single request, and the server returns exactly that shape — no more, no less. It solves two problems that show up repeatedly with REST as an API surface grows: over-fetching (a REST endpoint returns a fixed shape, so a client that only needs a user's name still gets their entire profile) and under-fetching (a screen needing data from three different REST resources needs three round trips, or a bespoke aggregating endpoint).

## 1. The Core Idea: One Endpoint, Client-Shaped Responses

A GraphQL API typically exposes a single endpoint (`/graphql`), and the request body itself describes the shape of the response.

```graphql
query {
  order(id: "1234") {
    id
    status
    customer {
      name
    }
    items {
      sku
      quantity
    }
  }
}
```

```json
{
  "data": {
    "order": {
      "id": "1234",
      "status": "SHIPPED",
      "customer": { "name": "Alice" },
      "items": [{ "sku": "ABC", "quantity": 2 }]
    }
  }
}
```

Compare to REST, where getting this same data typically means either one heavyweight endpoint returning far more than this screen needs, or three separate calls (`/orders/1234`, `/customers/{id}`, `/orders/1234/items`) that the client has to stitch together itself.

## 2. Schema-First: The Contract Is Explicit and Typed

Every GraphQL API is backed by a strongly typed schema that defines every type, field, and relationship available to query — this schema is both documentation and a runtime-enforced contract.

```graphql
type Order {
  id: ID!
  status: OrderStatus!
  customer: Customer!
  items: [OrderItem!]!
}

type Customer {
  id: ID!
  name: String!
  email: String!
}

enum OrderStatus {
  PENDING
  SHIPPED
  DELIVERED
}

type Query {
  order(id: ID!): Order
  orders(customerId: ID!): [Order!]!
}
```

A client can query the schema itself (introspection) to discover exactly what's available, which is what powers tooling like GraphQL Playground/GraphiQL and auto-generated client-side types — the schema being machine-readable, not just documentation prose, is a significant part of GraphQL's appeal for frontend teams.

## 3. Resolvers: Where the Actual Data Comes From

Each field in the schema is backed by a resolver function that knows how to fetch that specific piece of data — the schema describes the shape, resolvers supply the implementation.

```java
@Controller
public class OrderResolver {

    @QueryMapping
    public Order order(@Argument String id) {
        return orderService.findById(id);
    }

    @SchemaMapping(typeName = "Order", field = "customer")
    public Customer customer(Order order) {
        return customerService.findById(order.getCustomerId()); // resolves lazily, only if the client asked for it
    }

    @SchemaMapping(typeName = "Order", field = "items")
    public List<OrderItem> items(Order order) {
        return orderItemService.findByOrderId(order.getId()); // also resolved lazily, only on demand
    }
}
```

Crucially, a field resolver only runs if the client's query actually asked for that field — a query that omits `items` never calls the `items` resolver at all. This is what delivers the "client shapes the response" benefit concretely: unused data is never fetched from its underlying source in the first place.

## 4. The N+1 Problem

The same lazy-resolver design that avoids over-fetching creates a classic performance trap: fetching a list of parent objects, then resolving a per-item nested field, naturally results in one query for the list plus one additional query *per item* for the nested field.

```java
// Naive: one query for all orders, then ONE MORE query per order to get its customer — N+1 queries total
@SchemaMapping(typeName = "Order", field = "customer")
public Customer customer(Order order) {
    return customerRepository.findById(order.getCustomerId()); // fires once per order in the result set
}
```

```java
// Fixed: batch all the pending customer lookups from this request into ONE query
@BatchMapping
public Map<Order, Customer> customer(List<Order> orders) {
    List<String> customerIds = orders.stream().map(Order::getCustomerId).toList();
    Map<String, Customer> customers = customerRepository.findAllById(customerIds)
        .stream().collect(Collectors.toMap(Customer::getId, c -> c));
    return orders.stream().collect(Collectors.toMap(o -> o, o -> customers.get(o.getCustomerId())));
}
```

The general-purpose tool for this is a **DataLoader**: it batches individual lookup requests that occur within the same request/response cycle into a single underlying query, transparently, without the resolver code needing to know it's being batched. This is not an edge case — any GraphQL API with nested, list-producing fields needs to solve N+1 deliberately, or its database will be hit far harder than the equivalent REST implementation would have been.

## 5. Mutations: Writes in GraphQL

Reads are `query`; writes are `mutation` — same schema-driven, client-shaped-response idea, applied to operations that change data.

```graphql
mutation {
  placeOrder(input: { customerId: "42", items: [{ sku: "ABC", quantity: 2 }] }) {
    id
    status
  }
}
```

```java
@Controller
public class OrderMutationResolver {
    @MutationMapping
    public Order placeOrder(@Argument PlaceOrderInput input) {
        return orderService.place(input);
    }
}
```

The mutation still returns exactly the fields the client asks for, same as a query — useful for immediately getting back, say, just the generated `id` and `status` without a follow-up read.

## 6. Subscriptions: Real-Time Updates

A `subscription` lets a client receive a stream of updates over time (typically over WebSockets) rather than a single response, for data that changes and the client wants to be notified about as it happens.

```graphql
subscription {
  orderStatusChanged(orderId: "1234") {
    status
  }
}
```

This is GraphQL's answer to the same problem WebSockets solve more generally — a client subscribes once and receives pushed updates rather than polling.

## 7. GraphQL vs. REST — When Each Fits Better

| | REST | GraphQL |
| --- | --- | --- |
| Response shape | Fixed per endpoint | Client-specified per request |
| Number of round trips for composite data | Often multiple calls, or a bespoke aggregating endpoint | One call, one request shape |
| Caching | Straightforward — HTTP caching works natively per URL | Harder — every query can have a different shape, so a simple URL-based cache doesn't apply directly |
| Versioning | Often via URL/header versioning (`/v2/orders`) | Typically evolved by adding fields, deprecating old ones — same schema, no version in the URL |
| Learning curve / tooling | Simple, universally understood | Requires understanding the schema, resolvers, and N+1 pitfalls |
| Best fit | Simple CRUD-shaped resources, public APIs valuing HTTP-native caching | Complex, deeply nested data graphs; multiple client types (mobile, web) wanting different shapes from the same backend |

GraphQL earns its added complexity specifically when different clients need meaningfully different shapes of the same underlying data, or when the data graph is genuinely nested enough that REST would require either significant over-fetching or a proliferation of custom aggregating endpoints. For a simple, mostly-flat resource model, REST's simplicity and native HTTP caching are usually the better trade.

## 8. Errors in GraphQL Are Different from REST

A GraphQL response almost always returns HTTP `200`, even when part of the query failed — errors are reported in a separate `errors` array alongside whatever `data` could still be resolved, rather than via the HTTP status code.

```json
{
  "data": {
    "order": null
  },
  "errors": [
    { "message": "Order not found", "path": ["order"], "extensions": { "code": "NOT_FOUND" } }
  ]
}
```

This means client error handling has to check the `errors` array explicitly rather than relying on HTTP status codes the way a REST client typically would — a common integration mistake is treating every `200` as an unambiguous success.

## 9. Best Practices

| Practice | Recommendation |
| --- | --- |
| Design the schema around the client's needs, not the database's shape | The schema is the actual contract clients depend on — model it for how it will be queried, not as a 1:1 mirror of internal tables. |
| Solve N+1 deliberately with batching (DataLoader) | Any nested, list-producing field will otherwise fire one query per parent row under real data volumes. |
| Set query depth/complexity limits | An unbounded, deeply nested client query can otherwise force the server to do unbounded work per request. |
| Check the `errors` array, not just the HTTP status | A `200` response can still carry partial or complete failure inside `data`/`errors`. |
| Evolve the schema by adding and deprecating fields, not versioning the endpoint | GraphQL's single-endpoint model is designed around additive, in-place schema evolution. |
| Cache deliberately, since HTTP caching doesn't apply automatically | Consider persisted queries or field-level caching rather than assuming URL-based caching works as it would for REST. |
| Don't adopt GraphQL just because it's popular | REST's simplicity and native caching are the better fit for straightforward, mostly-flat resource APIs. |
