Decompose by subdomain splits services along DDD bounded contexts rather than along business capabilities or technical layers. Instead of asking "what does the business do?" (decompose by capability), you ask "where does the meaning of a word change?" — because that's exactly where a bounded context boundary, and therefore a service boundary, belongs. Two teams can use the word "Product" and mean genuinely different things, and that difference is the signal for where to split.

## 1. Subdomain vs. Business Capability Decomposition

Decomposing by business capability splits along what the organization does (`OrderService`, `BillingService`, `ShippingService`) — those boundaries are usually stable and easy to name. Decomposing by subdomain instead splits along where a model's concepts and rules actually diverge, even when two subdomains superficially deal with "the same" real-world thing.

```
Catalog subdomain:      Product = { name, description, images, category, searchable attributes }
Fulfillment subdomain:  Product = { SKU, weight, dimensions, current stock, warehouse location }
Pricing subdomain:      Product = { base price, discount rules, tax category }
```

All three "own" a concept called Product, but they care about entirely different attributes and enforce entirely different rules on it. Forcing them into one shared `Product` entity/table means every team has to agree on one schema for a concept they don't actually agree on the meaning of — that's the distributed-monolith trap this decomposition avoids.

## 2. Core, Supporting, and Generic Subdomains

DDD distinguishes subdomains by how much competitive value they carry, which should directly influence where you invest engineering effort and how you split services.

| Subdomain type | What it is | Example (retail) | Investment |
| --- | --- | --- | --- |
| Core | The thing that actually differentiates the business | A dynamic pricing/recommendation engine | Build in-house, invest heavily, iterate fast |
| Supporting | Necessary, but not a competitive differentiator | Order fulfillment workflow specific to your warehouse setup | Build in-house, but keep it simple |
| Generic | Solved problems every business needs | Authentication, payment processing, email sending | Buy or use an existing library/SaaS, don't reinvent |

This matters for service boundaries directly: a core subdomain deserves its own dedicated service with a rich model and dedicated team attention; a generic subdomain often shouldn't even be a custom service at all (use an identity provider, a payment gateway) rather than being built and decomposed like the rest of the system.

## 3. Bounded Contexts Are the Service Boundary

A bounded context is the boundary within which a specific model and its ubiquitous language apply consistently. When decomposing by subdomain, each bounded context typically becomes its own service, with its own database, its own version of shared-sounding concepts, and its own team ownership.

```java
// Catalog service's Product — only what the catalog subdomain cares about
public class Product {
    private String sku;
    private String name;
    private String description;
    private List<String> imageUrls;
    private Category category;
}

// Fulfillment service's Product — same real-world thing, deliberately different model
public class Product {
    private String sku; // the only field shared in meaning across contexts
    private Weight weight;
    private Dimensions dimensions;
    private WarehouseLocation location;
}
```

The shared `sku` is a deliberate integration point — the identifier that lets two contexts refer to "the same" product without forcing them to share every attribute of it. Everything else about `Product` is local to its own context and free to evolve independently.

## 4. Context Mapping: How Subdomains Relate

Once services are split by subdomain, you still need to describe how they depend on each other — DDD's context mapping patterns name the relationship explicitly instead of leaving it implicit.

| Pattern | Relationship | Typical use |
| --- | --- | --- |
| Shared Kernel | Two contexts deliberately share a small, jointly-owned model | A common `Money` value object used identically by both Billing and Pricing |
| Customer/Supplier | Upstream context's team accommodates downstream's needs | Inventory (upstream) adjusts its API based on what Fulfillment (downstream) needs |
| Conformist | Downstream just accepts the upstream model as-is, no negotiation | A team integrating with a third-party shipping provider's API |
| Anti-Corruption Layer | Downstream translates the upstream model into its own, isolating itself from upstream's model leaking in | Wrapping a legacy system's ugly data model behind a clean interface before it touches your domain code |

An Anti-Corruption Layer is worth calling out specifically for decomposition work: it's the standard tool for a new subdomain-based service that has to talk to an old, not-yet-decomposed monolith — the new service defines its own clean model and translates at the boundary, rather than letting the monolith's legacy shape spread into new code.

## 5. Signals You've Found a Real Subdomain Boundary

- Two groups use the same word but argue about what it means in meetings — that disagreement is the boundary.
- A single shared model keeps growing fields that only matter to one part of the business, while other parts ignore them entirely.
- One "team" naturally already owns a coherent piece of the domain and rarely needs to coordinate changes with another team over it.
- A rule that's absolute in one part of the business ("price must never go negative") is simply irrelevant or handled completely differently in another part.

## 6. Common Mistake: Decomposing by Database Table Instead of Model

A frequent anti-pattern is mistaking "these two things are stored in related tables" for "these are the same subdomain." Two subdomains can reference the same real-world entity (a customer, a product, an order) while genuinely disagreeing about its shape, lifecycle, and rules. Splitting by subdomain means splitting by where the *model and language* diverge, not by normalizing a shared database schema — if that split still feels wrong, it's usually because the technical schema was decomposed but the underlying bounded contexts were never actually identified.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Split where language and rules diverge, not where tables happen to differ | The test is "would two teams define this concept differently," not "is this normalized data." |
| Identify core vs. supporting vs. generic subdomains before deciding team investment | Don't build a custom, heavily-invested service for a generic problem (auth, payments) that should be bought instead. |
| Let each bounded context own its own version of a shared-sounding concept | A `Product` in Catalog and a `Product` in Fulfillment don't need to share a schema, only a shared identifier. |
| Name the relationship between contexts explicitly | Use context mapping patterns (Shared Kernel, Customer/Supplier, Anti-Corruption Layer) instead of leaving inter-service coupling implicit. |
| Use an Anti-Corruption Layer at the boundary with legacy systems | Keeps a new subdomain-based service's model clean instead of letting a legacy system's shape leak in. |
| Don't force a shared subdomain model just because it looks similar | A false shared abstraction across two genuinely different subdomains recreates the coupling decomposition was meant to remove. |
