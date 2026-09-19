Session management is how a system remembers who a caller is across multiple requests, since HTTP itself is stateless — every request arrives with no memory of any previous one unless something explicitly carries that context forward. The core design decision is where that "who is this and what do they currently have access to" state actually lives: on the server, or entirely inside something the client holds and presents each time. That choice has outsized consequences once a service runs as multiple replicas behind a load balancer.

## 1. Stateful Sessions

The server creates a session on first contact (typically at login), stores it in server-side memory or a shared store, and hands the client a small opaque session ID (usually via a cookie). Every subsequent request carries just that ID; the server looks up the actual session data behind it.

```java
@PostMapping("/login")
public ResponseEntity<Void> login(@RequestBody Credentials credentials, HttpSession session) {
    User user = authenticate(credentials);
    session.setAttribute("userId", user.getId()); // stored server-side, keyed by the session ID
    return ResponseEntity.ok().build(); // client just gets a JSESSIONID cookie back
}

@GetMapping("/orders")
public List<Order> getOrders(HttpSession session) {
    Long userId = (Long) session.getAttribute("userId"); // server looks it up by ID on every request
    return orderService.findByUserId(userId);
}
```

The client's cookie is meaningless on its own — it's just a lookup key. All the actual state (who they are, roles, cart contents, whatever the session holds) lives entirely on the server side.

## 2. Stateless Sessions

The server creates no session record at all. Instead, everything needed to authorize a request is encoded directly into a signed token the client holds and sends with every request — typically a JWT.

```java
public String issueToken(User user) {
    return Jwts.builder()
        .subject(user.getId().toString())
        .claim("roles", user.getRoles())
        .expiration(Date.from(Instant.now().plus(Duration.ofMinutes(15))))
        .signWith(signingKey)
        .compact(); // this token itself IS the session — nothing stored server-side
}

@GetMapping("/orders")
public List<Order> getOrders(@RequestHeader("Authorization") String bearerToken) {
    Claims claims = jwtValidator.validate(bearerToken); // verify signature + expiry, no lookup needed
    return orderService.findByUserId(Long.parseLong(claims.getSubject()));
}
```

Any server instance that has the signing key's public counterpart can validate the token and reconstruct who the caller is, entirely from the token itself — no shared state, no lookup, no dependency on which instance handled the previous request.

## 3. Stateful vs. Stateless, Side by Side

| | Stateful | Stateless |
| --- | --- | --- |
| Where session data lives | Server (in-memory or a shared store) | Entirely inside the token the client holds |
| What the client holds | An opaque, meaningless ID | The actual claims, signed |
| Horizontal scaling | Needs sticky sessions or a shared session store | Any instance can handle any request — nothing to share |
| Revoking access immediately | Trivial — delete the session server-side | Hard — a signed token is valid until it expires, by design |
| Token/session size | Small (just an ID) | Larger (carries all claims on every request) |
| What happens if the signing key or session store is compromised | Attacker needs access to the session store itself | Attacker who steals the signing key can forge arbitrary sessions |

The single biggest practical tradeoff: stateless sessions scale horizontally for free, but immediate revocation (forcing a logout, banning a compromised account right now) becomes genuinely hard, because the whole point of a self-contained token is that no server lookup is required to trust it.

## 4. Why This Matters More in Microservices

A monolith with one server instance can keep sessions in local memory without much consequence. Once a service runs as multiple replicas behind a load balancer, stateful sessions create a real problem: if `Instance A` created the session but the load balancer routes the next request to `Instance B`, that instance has no idea who the caller is.

```
Client → Load Balancer → Instance A (creates session, stores in local memory)
Client → Load Balancer → Instance B (next request lands here — has no idea this session exists)
```

Two ways to fix that for a stateful approach: **sticky sessions** (pin a client to the same instance, at the cost of uneven load distribution and losing sessions when that instance is scaled down or restarted) or a **shared session store** (Redis, so any instance can look up any session — this restores true statelessness at the instance level, at the cost of a network hop and a new shared dependency). Stateless (JWT) sessions sidestep the problem entirely, since any instance can validate the token without needing to see any other instance's state.

## 5. The Revocation Problem, and How to Mitigate It

Because a stateless token is valid purely by virtue of its signature and expiry, there's no natural way to say "actually, invalidate this one specific token right now" without reintroducing some form of server-side state — which starts to erode the "no shared state" benefit that made the approach attractive in the first place. Common mitigations, each a different point on the same tradeoff:

| Mitigation | How it works | Cost |
| --- | --- | --- |
| Short expiry + refresh tokens | Access token is valid for minutes; a longer-lived refresh token (which _can_ be revoked server-side) is used to get new ones | Revocation still isn't instant — a stolen access token stays valid until it naturally expires |
| Token blocklist | Maintain a small shared store of explicitly revoked token IDs, checked on validation | Reintroduces a shared lookup on every request — partially undoes the statelessness benefit |
| Short-lived tokens + re-check on sensitive actions | Accept the small revocation delay for routine requests, but re-verify status server-side before high-risk operations | Adds complexity only where it's actually justified, rather than everywhere |

There's no way to get instant revocation with zero server-side state — every mitigation trades back a little bit of the stateless model's core benefit to regain some ability to say "no" immediately.

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Default to stateless (JWT) sessions for microservices behind a load balancer | Avoids sticky sessions and shared session-store dependencies as the fleet scales up and down. |
| Keep access tokens short-lived | Minutes, not hours — bounds how long a compromised or stale token can be misused before it naturally expires. |
| Use a refresh token (revocable server-side) for long-lived sign-in | Combines mostly-stateless request handling with a real revocation point that doesn't require checking every request. |
| Use a shared session store (Redis) if you must stay stateful | Restores instance-level statelessness without giving up server-side session data or immediate revocation. |
| Avoid sticky sessions as a first resort | They undermine even load distribution and lose sessions when an instance is scaled down — treat them as a last resort, not a default. |
| Add a revocation mechanism deliberately if instant logout/ban is a hard requirement | A pure JWT approach can't do this natively — decide explicitly whether that gap is acceptable for your use case. |
| Never put sensitive data directly in a JWT's claims | Anything decodable from the token (even if signed, not encrypted) should be treated as visible to whoever holds it. |
