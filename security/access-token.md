The Access Token pattern solves a specific problem that shows up once authentication happens at the edge (an API Gateway) but authorization decisions still need to be made deep inside the system: how does a downstream service, several hops away from the original client request, know who's calling and what they're allowed to do, without re-running a full login check itself? The answer is to carry a token — the access token — through every hop, so each service can verify identity and permissions locally, from the token alone, without a network call back to an authentication server on every single request.

## 1. The Problem It Solves

A client authenticates once, typically at the gateway. Everything after that point is a chain of internal service-to-service calls that all need to agree on who the original caller is and what they're allowed to do.

```
Client → API Gateway (authenticates, obtains/validates access token)
            → OrderService (needs to know: which customer, what scopes)
                → InventoryService (needs the same identity/permission context)
```

Without a shared token, each service downstream would either have to trust the previous hop blindly (no real authorization) or call back to a central auth service on every request (a synchronous dependency and a latency/availability tax on every hop). The access token is what lets each service verify authorization locally and independently.

## 2. What's Inside an Access Token

Two structurally different implementations, with real tradeoffs between them:

| Token type | How it works | Verification |
| --- | --- | --- |
| Opaque/reference token | A random string with no embedded meaning — just a lookup key | Requires a call (or cache lookup) to the issuing authorization server to resolve it to actual claims |
| Self-contained (JWT) | Claims (subject, scopes, expiry, issuer) are encoded directly in the token, signed | Any service holding the signing key's public counterpart can verify it locally, no network call needed |

```java
// A JWT access token's payload, decoded — this is what a service reads to make its authorization decision
{
  "sub": "customer-4821",
  "scope": "orders:read orders:write",
  "iss": "https://auth.example.com",
  "aud": "internal-services",
  "exp": 1735689600
}
```

A self-contained JWT is what makes the pattern scale across many internal hops without turning the authorization server into a bottleneck every service has to call synchronously — this is usually the deciding factor for choosing JWTs over opaque tokens in a microservices context specifically, even though opaque tokens have their own advantages (instant revocation, no token size limit) in a simpler, single-service setup.

## 3. Propagating the Token Across Hops

Each service in the call chain forwards the access token (or a derived, internally-scoped equivalent) to the next hop, and validates it before doing any work.

```java
@Component
public class TokenPropagationFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // The gateway validated this token already — forward it unchanged to the backend service
        return chain.filter(exchange);
    }
}

@Service
public class InventoryClient {
    public StockLevel getStock(String sku, String accessToken) {
        return restClient.get()
            .uri("http://inventory-service/api/inventory/{sku}", sku)
            .header("Authorization", "Bearer " + accessToken) // propagate it downstream, don't drop it
            .retrieve()
            .body(StockLevel.class);
    }
}
```

Every service that receives the token independently validates its signature, expiry, issuer, and audience before trusting anything in it — trusting the previous hop's word that "this request is authenticated" without validating the token itself defeats the entire point of the pattern.

## 4. Scoped-Down Internal Tokens

Forwarding the exact same client-facing access token unchanged to every internal hop works, but it means every downstream service sees the full scope the client was granted, even for calls that only need a narrow subset of it. A more defensive variant has each service exchange the incoming token for a new, more narrowly scoped token before calling the next service — the Phantom Token pattern is one specific implementation of this, where the gateway swaps an opaque client-facing token for a JWT with only the claims that specific internal call chain actually needs.

```
Client's token: scope = "orders:read orders:write profile:read payment:write"
                                    ↓ (gateway exchanges/narrows before forwarding)
InventoryService receives:  scope = "orders:read" only — nothing it doesn't need
```

This limits the blast radius if a downstream service is ever compromised or has a bug — it can only misuse the permissions it was actually handed, not the full breadth of what the original client was allowed to do.

## 5. Token Expiry and Long-Running Chains

An access token is deliberately short-lived (minutes), which is fine for a single request/response chain but creates a real problem for a long-running internal workflow (a saga, an async job) that outlives the token's validity window. There's no clean universal answer here — options include re-authenticating for each new leg of a long-running process, using a service-to-service credential (client credentials grant) for background work instead of forwarding a user's original token at all, or accepting that a long-running process needs its own longer-lived, more narrowly scoped machine credential rather than trying to stretch a user-facing access token's lifetime to fit.

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Validate the token at every hop, not just at the gateway | Trusting an upstream service's word without independent validation defeats the pattern's purpose. |
| Prefer self-contained (JWT) tokens for internal propagation | Avoids every downstream service needing a synchronous call back to the authorization server just to verify a request. |
| Keep access tokens short-lived | Minutes, not hours — limits the damage window if a token is ever leaked or intercepted. |
| Scope tokens down before forwarding to less-trusted or narrower-purpose services | A downstream service should only receive the permissions it actually needs for that specific call. |
| Never let a service log or persist a full access token | Treat it as a credential — log only a token ID/reference if traceability is needed, never the token itself. |
| Use a machine credential (client credentials grant) for background/long-running work | Don't try to stretch a short-lived, user-facing access token's lifetime to cover a process that outlives it. |
| Verify signature, issuer, audience, and expiry on every validation, not just presence of a token | A syntactically valid but wrong-audience or expired token must be rejected exactly like a missing one. |
