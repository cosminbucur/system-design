A socket is the fundamental abstraction underneath nearly every network call in this entire set of notes — a TCP connection identified by an IP address and port, through which bytes flow in both directions. HTTP (and therefore every REST API) is built on top of sockets, opening one per request/response and closing it afterward. WebSockets exist specifically because that "open, one request/response, close" model doesn't fit every use case — some interactions genuinely need a connection that stays open, with either side able to send a message at any time.

## 1. Raw Sockets: The Foundation

A socket in Java is a plain, low-level TCP endpoint — `Socket` for a client connection, `ServerSocket` for something listening and accepting connections.

```java
// Server
try (ServerSocket serverSocket = new ServerSocket(8080)) {
    while (true) {
        Socket clientSocket = serverSocket.accept(); // blocks until a client connects
        // handle clientSocket — typically handed off to a thread pool,
        // since a naive accept-loop handling one client at a time can't serve concurrent clients
        new Thread(() -> handleClient(clientSocket)).start();
    }
}

// Client
try (Socket socket = new Socket("localhost", 8080);
     PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
     BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()))) {
    out.println("hello");
    String response = in.readLine();
}
```

Almost nobody writes raw sockets directly in application code — HTTP clients/servers, JDBC drivers, and message broker clients all use sockets internally, wrapped in a much higher-level protocol. Understanding this layer matters mainly for reasoning about what's actually happening underneath: a "connection" in any of those higher-level tools is, physically, one of these TCP sockets, with its own connect/read/write timeouts and its own failure modes.

## 2. TCP vs. UDP, Briefly

Nearly everything in this note (and everything else in these notes involving networking) sits on top of TCP, not UDP — worth knowing the distinction exists even though UDP rarely appears in typical application code.

|             | TCP                                                                                    | UDP                                                                                                  |
| ----------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Connection  | Connection-oriented — a handshake establishes the connection before data flows         | Connectionless — packets (datagrams) are just sent, no setup                                         |
| Reliability | Guaranteed delivery and ordering (retransmits lost packets)                            | Best-effort — packets can be lost, duplicated, or arrive out of order, and nothing retransmits them  |
| Use case    | Anything needing guaranteed, ordered delivery — HTTP, database connections, WebSockets | Latency-sensitive, loss-tolerant traffic — video/audio streaming, DNS lookups, some real-time gaming |

TCP's guarantees are exactly why HTTP and WebSocket are built on it — losing or reordering a byte of an API response or a chat message isn't acceptable, so the retransmission/ordering cost is worth paying.

## 3. Why WebSockets Exist

Plain HTTP is fundamentally request-driven: the client always initiates, the server always responds, and the connection typically closes (or is reused for the _next_ client-initiated request) once that response is sent. This is a poor fit for anything where the _server_ needs to push data to the client the moment something happens — a live balance update, a fraud alert, a chat message from another user — without the client repeatedly asking "anything new?".

A WebSocket connection starts as an HTTP request but then upgrades into a persistent, full-duplex TCP connection: once established, either side can send a message at any time, with no new HTTP request/response cycle needed per message.

```
HTTP:        Client → request → Server → response → (connection closes or is reused for the NEXT unrelated request)
WebSocket:   Client → upgrade request → Server → upgrade response → [connection STAYS OPEN]
             Client ⇄ message ⇄ Server ⇄ message ⇄ Client ⇄ ...  (either side, any time, same connection)
```

## 4. The WebSocket Handshake

The connection starts as a normal HTTP request with an `Upgrade` header — if the server agrees, the HTTP exchange completes and the _same underlying TCP socket_ is repurposed to carry WebSocket frames instead of further HTTP messages.

```
GET /ws/chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

`Sec-WebSocket-Key`/`Accept` exist to confirm the server genuinely understood and intended to handle a WebSocket upgrade (rather than, say, a misconfigured proxy blindly forwarding the request) — after the `101 Switching Protocols` response, no more HTTP requests/responses happen on this connection; it's now a raw, framed, bidirectional message channel over the one TCP socket that was opened for the original HTTP request.

## 5. WebSockets in Java/Spring

```java
// Jakarta WebSocket API — a plain endpoint, no Spring required
@ServerEndpoint("/ws/chat")
public class ChatEndpoint {

    @OnOpen
    public void onOpen(Session session) {
        sessions.add(session);
    }

    @OnMessage
    public void onMessage(String message, Session session) {
        sessions.forEach(s -> s.getAsyncRemote().sendText(message)); // broadcast to everyone connected
    }

    @OnClose
    public void onClose(Session session) {
        sessions.remove(session);
    }
}
```

```java
// Spring's WebSocketHandler — the equivalent, integrated with the rest of a Spring app
@Component
public class NotificationWebSocketHandler extends TextWebSocketHandler {

    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message) {
        // handle an inbound message from this specific client
    }

    public void sendNotification(WebSocketSession session, String payload) throws IOException {
        session.sendMessage(new TextMessage(payload)); // server-initiated push, at any time
    }
}
```

For richer pub/sub semantics (topics, subscriptions, broadcasting to groups of clients rather than raw message handling), Spring layers STOMP (a simple messaging protocol) on top of WebSocket:

```java
@Controller
public class NotificationController {

    @MessageMapping("/accounts/{accountId}/watch") // client subscribes here
    @SendTo("/topic/accounts/{accountId}")
    public AccountUpdate onSubscribe(@DestinationVariable String accountId) {
        return accountService.currentState(accountId);
    }
}
```

```java
@Autowired
private SimpMessagingTemplate messagingTemplate;

public void publishBalanceUpdate(String accountId, AccountUpdate update) {
    messagingTemplate.convertAndSend("/topic/accounts/" + accountId, update); // pushes to every subscriber
}
```

This STOMP-over-WebSocket shape looks a lot like the pub/sub pattern, deliberately — it's the same publish/subscribe concept, just terminating at a browser/client connection instead of another backend service.

## 6. Browser Compatibility: SockJS

Some corporate proxies, older browsers, or restrictive network environments don't support the WebSocket upgrade at all. SockJS provides a client library and server-side fallback chain (falling back to HTTP long-polling or streaming) that presents the same WebSocket-like API to application code, regardless of which underlying transport actually got through.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS(); // falls back automatically if a real WebSocket can't connect
    }
}
```

## 7. Scaling WebSockets Across Multiple Instances

This is the sharpest operational difference from a stateless REST API: a WebSocket connection is inherently stateful and pinned to whichever specific server instance accepted it — unlike a stateless HTTP request, which any instance behind a load balancer can serve interchangeably. If a client connects to instance A, a message meant for that client but published from instance B has nowhere to go unless something bridges the two.

```
Client ⇄ [Instance A] — client's WebSocket connection lives HERE, and only here

Instance B publishes an update meant for that client
   → without a broker relay, Instance B has no way to reach a connection it doesn't hold
   → Instance A needs to be told, so it can push down the connection IT holds
```

The standard fix is a message broker relay: instead of each instance trying to push directly, every instance publishes updates to a shared broker (Redis Pub/Sub, RabbitMQ, or a full STOMP broker relay), and each instance subscribes to the topics its own connected clients care about, relaying matching messages down the specific WebSocket connections it holds locally.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableStompBrokerRelay("/topic")
            .setRelayHost("rabbitmq-host")
            .setRelayPort(61613); // every instance relays through the SAME external broker
    }
}
```

This is also why a load balancer in front of WebSocket-serving instances typically needs session affinity (sticky sessions) — a client's _reconnection_ attempts should ideally land back on an instance that can resume cleanly, and a load balancer that doesn't understand the connection is long-lived can otherwise round-robin a single logical session across instances in ways that break it.

## 8. Backpressure Over a WebSocket

A slow client (a mobile device on a poor connection, a browser tab that's backgrounded and throttled) can't keep up with a fast-publishing server — messages queue up in the connection's send buffer, and an unbounded buffer here is exactly the same failure mode.

```java
@Override
public void handleTransportError(WebSocketSession session, Throwable exception) {
    if (session.getTextMessageSizeLimit() > 0 && /* buffer exceeded */ true) {
        // Spring throws SessionLimitExceededException once configured buffer limits are hit —
        // handle it explicitly rather than letting the buffer grow without bound
    }
}
```

```java
container.setMaxSessionIdleTimeout(60000L);
container.setMaxTextMessageBufferSize(8192); // bounded — the backpressure decision
```

Once a client is confirmed too far behind to catch up, the practical options are exactly the same menu from backpressure: drop older, less-relevant messages (send only the latest account balance, not every intermediate update), disconnect the slow client and let it reconnect and resync from current state, or — for genuinely critical messages that can't be dropped — fall back to a different, more reliable delivery mechanism (a webhook or a durable queue) rather than a best-effort push over a live socket.

## 9. Detecting a Dead Connection: Heartbeats

TCP alone doesn't reliably tell either side that the other end silently vanished (a client's laptop closed without a clean disconnect, a mobile network dropped without notice) — the connection can appear open indefinitely with no data flowing, consuming server resources for a client that's actually long gone.

```java
config.enableStompBrokerRelay("/topic")
    .setHeartbeat(10000, 10000); // send/expect a heartbeat frame every 10s in both directions
```

```javascript
// Client-side ping/pong — if no pong arrives within the timeout, treat the connection as dead and reconnect
setInterval(() => socket.send(JSON.stringify({ type: "ping" })), 15000);
```

A missed heartbeat is the practical signal to close the connection server-side (freeing its resources) and for the client to attempt reconnection — without heartbeats, a "half-open" connection can sit around consuming a server thread/resources for a client that's never coming back.

## 10. Security

- **`wss://` (WebSocket Secure), not `ws://`**, in any production deployment — the same TLS the rest of an application already requires for HTTPS, applied to the WebSocket upgrade and everything sent afterward.
- **Validate the `Origin` header** on the server during the handshake — without this, a malicious page on an unrelated site could open a WebSocket connection to your server from a victim's browser (Cross-Site WebSocket Hijacking), riding the victim's existing session cookie the same way CSRF exploits a plain HTTP endpoint.
- **Authentication is awkward by design**: browsers' WebSocket API can't set arbitrary headers (like `Authorization: Bearer ...`) on the initial handshake request — the common workarounds are passing a short-lived token as a query parameter (`wss://.../ws?token=...`, logged carefully — see the URL-logging caution implicit in security's general credential-handling guidance) or via a WebSocket subprotocol header, then validating it server-side during the handshake before accepting the upgrade.

## 11. WebSocket vs. Server-Sent Events vs. Polling

WebSocket isn't the only option for server-to-client push — it's the right one specifically when the client also needs to send messages back over the same connection.

| Mechanism                 | Direction                                       | Fits when                                                                                                                                                                                                           |
| ------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Polling                   | Client-initiated, repeated                      | Real-time isn't critical; simplest to implement on both sides                                                                                                                                                       |
| Server-Sent Events (SSE)  | Server → client only, over plain HTTP           | One-way streaming (live feed, notifications) where the client never needs to push data back over the same channel; simpler infrastructure (works over standard HTTP/1.1, browsers auto-reconnect via `EventSource`) |
| WebSocket                 | Bidirectional                                   | Genuine two-way, low-latency interaction — chat, collaborative editing, trading interfaces                                                                                                                          |
| Reactive `Flux` streaming | Server → client, often over SSE or chunked HTTP | The backend implementation technique for a one-way stream — pairs naturally with SSE on the wire                                                                                                                    |

Choosing WebSocket for a purely one-directional server-push feed (e.g., a live dashboard the user never sends data back through) adds connection-management complexity for a bidirectional capability that's never actually used — SSE is the simpler, more appropriate tool for that specific shape.

## 12. Client-Side Reconnection

A dropped WebSocket connection (network blip, server restart during a deploy) should be retried with backoff, not hammered immediately — the same exponential-backoff-with-jitter pattern already covered for webhook delivery.

```javascript
let attempt = 0;
function connect() {
  const socket = new WebSocket("wss://example.com/ws");
  socket.onclose = () => {
    const delay = Math.min(1000 * 2 ** attempt++, 30000); // exponential backoff, capped
    setTimeout(connect, delay);
  };
  socket.onopen = () => {
    attempt = 0;
  }; // reset backoff once successfully reconnected
}
```

## 13. Best Practices

| Practice                                                             | Recommendation                                                                                                                                                        |
| -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Use WebSocket only when the client genuinely needs to push data back | For one-way server-to-client streaming, prefer SSE — simpler infrastructure, no unused bidirectional capability.                                                      |
| Always use `wss://` and validate the `Origin` header                 | Prevents both plaintext interception and Cross-Site WebSocket Hijacking.                                                                                              |
| Never rely on a header-based auth token for the handshake            | Browsers can't set custom headers on the WebSocket upgrade — pass a short-lived token via query param or subprotocol and validate server-side.                        |
| Bound message buffers and handle backpressure explicitly             | An unbounded per-connection send buffer for a slow client is the same failure mode as any other unbounded queue.                                                      |
| Configure heartbeats and idle timeouts                               | TCP alone won't reliably tell you a client silently vanished — a missed heartbeat is the signal to free that connection's resources.                                  |
| Use a broker relay to scale across multiple instances                | A WebSocket connection is pinned to the instance that accepted it — publishing from a different instance needs a shared broker (Redis/RabbitMQ) to actually reach it. |
| Reconnect with exponential backoff, not immediately                  | Mirrors the retry discipline in webhooks — an immediate, unthrottled reconnect storm after a server restart can overwhelm the very instance that just came back up.   |
