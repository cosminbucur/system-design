# Core Networking Concepts in Java

Understanding core networking in Java is essential for building networked applications, from simple client-server tools to high-performance distributed systems. Java provides robust networking capabilities built primarily into the `java.net` and `java.nio` packages.

Here is a breakdown of the core networking concepts in Java, organized from low-level sockets to modern non-blocking I/O.

---

## 1. IP Addresses and Domain Names (`InetAddress`)

Before connecting to a network, Java uses the `InetAddress` class to represent an Internet Protocol (IP) address. It handles both IPv4 and IPv6 addresses and performs DNS lookups.

* **Key Class:** `java.net.InetAddress`
* **Common Use Case:** Resolving a hostname to an IP address or getting the local machine's address.

```java
import java.net.InetAddress;

public class NetworkExample {
    public static void main(String[] args) throws Exception {
        InetAddress address = InetAddress.getByName("google.com");
        System.out.println("IP Address: " + address.getHostAddress());
    }
}
```

---

## 2. Sockets and Ports

A **socket** is an endpoint for communication between two machines over a network. Think of an IP address as a building address and a **port number** (0–65535) as the specific apartment or room number within that building.

Java provides different types of sockets depending on the transport layer protocol:

### A. TCP Sockets (`Socket` and `ServerSocket`)

Transmission Control Protocol (TCP) is connection-oriented, reliable, and ensures ordered delivery of data.
* **`ServerSocket`**: Used by servers to listen for incoming client connections on a specific port.
* **`Socket`**: Used by clients to connect to a server, or returned by `ServerSocket.accept()` once a connection is established.

**TCP Server Example:**
```java
import java.io.*;
import java.net.*;

public class SimpleServer {
    public static void main(String[] args) throws IOException {
        try (ServerSocket serverSocket = new ServerSocket(8080)) {
            System.out.println("Server listening on port 8080...");
            Socket clientSocket = serverSocket.accept();
            
            BufferedReader reader = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
            PrintWriter writer = new PrintWriter(clientSocket.getOutputStream(), true);
            
            String message = reader.readLine();
            System.out.println("Received: " + message);
            writer.println("Hello from Java Server!");
        }
    }
}
```

### B. UDP Sockets (`DatagramSocket` and `DatagramPacket`)

User Datagram Protocol (UDP) is connectionless and faster, but unreliable (packets may arrive out of order, get duplicated, or be dropped). It is ideal for streaming, gaming, or DNS lookups.
* **`DatagramSocket`**: Sends and receives UDP datagram packets.
* **`DatagramPacket`**: Wraps the data payload along with the target destination IP and port.

---

## 3. URLs and URIs (`URL` and `URI`)

If you need to interact with web resources rather than raw sockets, Java provides high-level abstractions for Uniform Resource Identifiers and Locators.

* **`URI`**: Identifies a resource (syntax parsing).
* **`URL`**: Identifies a resource *and* provides the means to access it (can open streams to download content).
* **`HttpURLConnection`**: A lightweight class built into the standard library for sending HTTP/HTTPS requests (though modern Java applications often prefer `java.net.http.HttpClient`).

---

## 4. Modern HTTP Client (`java.net.http.HttpClient`)

Introduced in Java 11, the modern HTTP client provides a streamlined, fluent API supporting HTTP/1.1 and HTTP/2, synchronous and asynchronous requests, and WebSockets out of the box.

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class HttpClientExample {
    public static void main(String[] args) throws Exception {
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://api.github.com"))
                .GET()
                .build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
        System.out.println("Status Code: " + response.statusCode());
        System.out.println("Response Body: " + response.body());
    }
}
```

---

## 5. Non-Blocking I/O (`java.nio`)

Traditional sockets (`java.net`) use **blocking I/O**, meaning a thread blocks (waits idly) while reading from or writing to a stream. For high-concurrency applications handling thousands of simultaneous connections, this approach consumes too much memory and thread overhead.

Java NIO (New I/O) introduces non-blocking networking using:
* **`Channel`**: Open connections (like `SocketChannel` or `ServerSocketChannel`).
* **`Buffer`**: Memory blocks where data is read or written.
* **`Selector`**: A component that monitors multiple channels for events (such as "connection ready" or "data available"), allowing a single thread to manage many network connections efficiently.

---

## Summary Checklist for Java Networking

* Use **`InetAddress`** for IP/DNS lookups.
* Use **`Socket` / `ServerSocket`** for traditional TCP client-server apps.
* Use **`DatagramSocket`** for lightweight UDP communication.
* Use **`HttpClient`** for modern REST and HTTP web requests.
* Use **`java.nio` (Channels & Selectors)** when building high-concurrency, non-blocking servers.