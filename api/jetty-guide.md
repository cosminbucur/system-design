# Eclipse Jetty: Core Concepts, Architecture, and Best Practices

**Eclipse Jetty** is an open-source, highly scalable, Java-based web server and servlet container. It is widely used because it is lightweight, fast, standards-compliant, and uniquely designed to be **embedded** directly into applications (like Spring Boot, microservices, or custom Java programs) rather than acting solely as a heavy standalone application server.

---

## 1. Core Concepts

Jetty's architecture is structured as a tree of modular Java components, centered around a few main building blocks:

*   **The Server (`org.eclipse.jetty.server.Server`):** 
    The root component of any Jetty instance. It coordinates the thread pool, lifecycle, connectors, and handlers. 
*   **Connectors:** 
    Responsible for accepting incoming network traffic (HTTP, HTTPS, HTTP/2, WebSockets) from clients. Connectors listen on specific network ports and pass raw connection bytes to Jetty's I/O layer.
*   **Handlers:** 
    Components that process incoming requests. Jetty uses a flexible handler chain pattern (`HandlerCollection`, `ContextHandler`, etc.). A request passes sequentially through handlers until one handles it and generates a response.
*   **The Servlet Container & Contexts:** 
    Jetty includes a full Jakarta EE / Java EE Servlet engine. Components like `WebAppContext` map URL paths to specific servlets, filters, and standard web application packaging (`WAR` files or directories).
*   **Jetty I/O (Endpoints and Connections):** 
    At the low level, Jetty utilizes Java NIO to perform non-blocking I/O. An `EndPoint` manages the raw socket abstraction, while a `Connection` deserializes byte streams into high-level protocol objects (like HTTP requests or WebSocket frames).

---

## 2. How Jetty Works

Jetty follows a clean lifecycle and event-driven request-response model:

1.  **Network Ingestion:** A client sends an HTTP request. A **Connector** accepts the socket connection using non-blocking I/O (NIO) selectors.
2.  **Parsing:** The connection layer reads raw bytes via `EndPoint` and parses them into a structured request object without blocking precious threads.
3.  **Handler Routing:** The request is handed over to the **Server**, which passes it through the configured **Handlers** (e.g., security checks, context routing).
4.  **Servlet Execution:** If it matches a web app context, the request enters the servlet container, invoking custom business logic, filters, or REST endpoints.
5.  **Response Generation:** The application writes output back through Jetty's asynchronous output streams, and the connection layer flushes the bytes back to the client.

---

## 3. Best Practices for Eclipse Jetty

### Architecture & Deployment
*   **Prefer Embedded Mode for Microservices:** Leverage Jetty's programmatic API to embed it directly within your application JAR. This eliminates external container management and aligns well with cloud-native or containerized workflows.
*   **Use Modular Configuration (Standalone Mode):** If running Jetty standalone, take advantage of its module system (`$JETTY_HOME/modules`). Only enable the modules you explicitly require (e.g., HTTP/2, security, deployment) to keep the memory footprint low and attack surface minimal.
*   **Separate `JETTY_HOME` and `JETTY_BASE`:** Never modify files directly inside `JETTY_HOME`. Keep binaries separate from your application configurations (`JETTY_BASE`), making upgrades smooth and preventing custom configurations from being overwritten.

### Concurrency & Performance Tuning
*   **Tune Thread Pools Carefully:** Jetty manages its own thread pools for handling tasks. Monitor thread exhaustion under load and tune maximum/minimum thread limits to match your hardware and workload constraints.
*   **Leverage Asynchronous Requests:** For long-polling, heavy I/O, or streaming endpoints, use Jetty's asynchronous request processing (e.g., Servlet 3.1+ async features or Jetty's native `AsyncContext`). This frees up handler threads to process other incoming traffic instead of blocking.
*   **Proper Resource Consumption:** When dealing with low-level Jetty I/O or custom content sources (`Content.Source`), always ensure streams are fully consumed or explicitly failed to prevent memory leaks and dangling resources.

### Security & Operations
*   **Enforce TLS/HTTPS by Default:** Never run production endpoints unencrypted. Configure SSL/TLS connectors properly via Jetty's `ssl` modules, and enforce secure cipher suites.
*   **Isolate Classpaths:** Avoid dumping loose third-party libraries into global extension folders (`lib/ext`) unless necessary. Keep dependencies scoped locally to your application to avoid version conflicts and classpath pollution.
*   **Keep Jetty Updated:** Track stable patch releases of your chosen Jetty version branch to ensure you receive up-to-date security patches and performance fixes.