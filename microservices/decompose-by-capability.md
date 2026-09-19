Decompose by business capability splits services along what the organization *does* — the stable, named functions a business performs regardless of how its internal teams or org chart happen to be arranged today. A capability is something like "manage orders," "process payments," or "track inventory" — durable verbs describing the business itself, not implementation detail. This is usually the first, most intuitive way people decompose a monolith, precisely because these boundaries are easy to name and tend to map cleanly onto how a business already talks about itself.

## 1. What a Business Capability Looks Like

```
Order Management     → OrderService
Inventory Tracking   → InventoryService
Payment Processing   → PaymentService
Customer Management  → CustomerService
Shipping/Fulfillment → ShippingService
```

Each capability becomes a service that owns the data and logic needed to perform that function end to end. The test for a good capability boundary: could you describe it to a non-technical stakeholder in one sentence, and would they immediately recognize it as "a thing the business does"? "We process payments" passes that test; "we manage the `payment_audit_log` table" does not — that's an implementation detail, not a capability.

## 2. Where Capabilities Come From

Business capabilities are usually discovered, not invented — they already exist informally in how an organization describes itself: in org charts, in how departments are named, in the vocabulary used in planning meetings. Decomposing along them is largely a matter of making an already-existing structure explicit in software, rather than designing a novel one from scratch.

| Signal | Example |
| --- | --- |
| A department or team already owns this end to end | A "Payments" team that owns payment processing, reconciliation, and refunds |
| It appears as a noun in business strategy documents | "Order Management," "Fulfillment," "Customer Support" |
| It has its own success metrics distinct from other capabilities | Inventory has its own accuracy/turnover metrics separate from Order Management's throughput metrics |
| Stakeholders would say "that's not really our job, that's theirs" | Order Management doesn't own how payments are authorized — that's a different capability |

## 3. Decompose by Capability vs. Decompose by Subdomain

Both are legitimate decomposition strategies and often arrive at similar-looking boundaries, but they ask different questions to get there.

| | Decompose by capability | Decompose by subdomain |
| --- | --- | --- |
| Question asked | "What does the business do?" | "Where does the meaning of a concept/model diverge?" |
| Source of the boundary | Org structure, business function | Domain modeling, ubiquitous language |
| Typical granularity | Coarser — aligned to a whole business function | Can be finer — two subdomains might sit inside what looks like one capability |
| Best suited for | An initial, pragmatic first decomposition of a monolith | Refining boundaries once the domain model reveals real conceptual splits |

In practice these frequently converge — "Order Management" as a capability and an "Order Management" bounded context often describe the same service — but capability decomposition is the faster, more approachable starting point, while subdomain decomposition provides a sharper, model-driven way to validate or refine those same boundaries once you look closer at how concepts actually diverge inside them.

## 4. A Capability Can Still Contain Multiple Subdomains

A single named capability sometimes turns out to hide more than one underlying subdomain once modeled closely, which is a signal the initial capability-level boundary may be too coarse.

```
Capability: "Order Management"
  → Order Placement subdomain (validating and accepting a new order)
  → Order Fulfillment subdomain (picking, packing, shipping)
```

If these two turn out to have genuinely different rates of change, different scaling needs, or different teams wanting ownership, splitting the one capability-level service into two subdomain-aligned services is a natural evolution — decomposing by capability first doesn't lock you out of refining further by subdomain later.

## 5. The Trap: Decomposing by Technical Layer Instead

A common mistake that looks superficially similar to capability decomposition but isn't: splitting services by technical layer (a `PresentationService`, a `BusinessLogicService`, a `DataAccessService`) rather than by business function. This fails because a single business change (say, a new order validation rule) now requires coordinated deployment across all three layer-services, which is exactly the tight coupling microservices are meant to remove. A capability boundary should let one business change be made, tested, and deployed within a single service — if a typical change requires touching multiple services in lockstep, the decomposition axis is probably wrong.

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Name capabilities the way the business already names them | If a non-technical stakeholder wouldn't recognize the name, it's likely a technical concept, not a capability. |
| Start decomposition here before refining by subdomain | Capability boundaries are faster to identify and give a reasonable first cut at service boundaries. |
| Watch for a capability hiding multiple subdomains | If a capability's model keeps developing internal seams (different rules, different rates of change), consider splitting further. |
| Never decompose by technical layer | A presentation/logic/data split forces every business change to touch multiple services in lockstep — the opposite of independent deployability. |
| Validate a capability boundary against real business change patterns | If most feature work naturally stays within one capability's service, the boundary is good; frequent cross-service coordination is a signal it's wrong. |
| Let ownership follow the capability | A capability-aligned service is easiest to give to a single team that can own it end to end, including its data. |
