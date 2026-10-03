# Implementing a Custom REST Client in Modern Java

Implementing a custom REST client in modern Java (Java 11+) is straightforward using the built-in `java.net.http.HttpClient`. It eliminates the need for heavy external libraries like Apache HttpClient or OkHttp for most standard use cases.

Below is a clean, reusable implementation of a lightweight REST client that handles common HTTP methods, headers, timeouts, and error handling, followed by a look at how you can structure it with a fluent builder pattern.

---

## 1. Core Implementation (`HttpClient`-Based)

This implementation provides a base client wrapper that you can easily extend or inject into your services.

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.Map;
import java.util.concurrent.CompletableFuture;

public class CustomRestClient {
    
    private final HttpClient httpClient;
    private final String baseUrl;
    private final Map<String, String> defaultHeaders;

    public CustomRestClient(String baseUrl, Map<String, String> defaultHeaders) {
        this.baseUrl = baseUrl;
        this.defaultHeaders = defaultHeaders != null ? defaultHeaders : Map.of();
        
        // Configure the underlying HTTP client (supports HTTP/2 by default)
        this.httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .followRedirects(HttpClient.Redirect.NORMAL)
                .build();
    }

    // Synchronous GET
    public String get(String endpoint) {
        try {
            HttpRequest.Builder requestBuilder = HttpRequest.newBuilder()
                    .uri(URI.create(baseUrl + endpoint))
                    .GET();
            
            addHeaders(requestBuilder);
            
            HttpResponse<String> response = httpClient.send(
                    requestBuilder.build(), 
                    HttpResponse.BodyHandlers.ofString()
            );
            
            validateResponse(response);
            return response.body();
        } catch (Exception e) {
            throw new RuntimeException("GET request failed for " + endpoint, e);
        }
    }

    // Synchronous POST with JSON payload
    public String post(String endpoint, String jsonBody) {
        try {
            HttpRequest.Builder requestBuilder = HttpRequest.newBuilder()
                    .uri(URI.create(baseUrl + endpoint))
                    .header("Content-Type", "application/json")
                    .POST(HttpRequest.BodyPublishers.ofString(jsonBody));
            
            addHeaders(requestBuilder);
            
            HttpResponse<String> response = httpClient.send(
                    requestBuilder.build(), 
                    HttpResponse.BodyHandlers.ofString()
            );
            
            validateResponse(response);
            return response.body();
        } catch (Exception e) {
            throw new RuntimeException("POST request failed for " + endpoint, e);
        }
    }

    // Asynchronous GET (utilizing CompletableFuture)
    public CompletableFuture<String> getAsync(String endpoint) {
        HttpRequest.Builder requestBuilder = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + endpoint))
                .GET();
        
        addHeaders(requestBuilder);

        return httpClient.sendAsync(requestBuilder.build(), HttpResponse.BodyHandlers.ofString())
                .thenApply(response -> {
                    validateResponse(response);
                    return response.body();
                });
    }

    private void addHeaders(HttpRequest.Builder builder) {
        defaultHeaders.forEach(builder::header);
    }

    private void validateResponse(HttpResponse<String> response) {
        int status = response.statusCode();
        if (status < 200 || status >= 300) {
            throw new RuntimeException("HTTP Error Status: " + status + ", Body: " + response.body());
        }
    }
}
```

---

## 2. Adding a Fluent Builder Pattern (Optional Enhancement)

If you want a more expressive, chainable syntax for building individual requests (similar to modern fluent APIs), you can layer a request builder on top:

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.HashMap;
import java.util.Map;

public class FluentRestClient {
    
    private static final HttpClient SHARED_CLIENT = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .build();

    public static RequestBuilder builder() {
        return new RequestBuilder();
    }

    public static class RequestBuilder {
        private String uri;
        private String method = "GET";
        private String body = "";
        private final Map<String, String> headers = new HashMap<>();

        public RequestBuilder url(String uri) {
            this.uri = uri;
            return this;
        }

        public RequestBuilder get() {
            this.method = "GET";
            return this;
        }

        public RequestBuilder post(String body) {
            this.method = "POST";
            this.body = body;
            this.headers.put("Content-Type", "application/json");
            return this;
        }

        public RequestBuilder header(String name, String value) {
            this.headers.put(name, value);
            return this;
        }

        public HttpResponse<String> execute() {
            try {
                HttpRequest.Builder reqBuilder = HttpRequest.newBuilder()
                        .uri(URI.create(uri));

                headers.forEach(reqBuilder::header);

                if ("POST".equalsIgnoreCase(method)) {
                    reqBuilder.POST(HttpRequest.BodyPublishers.ofString(body));
                } else {
                    reqBuilder.GET();
                }

                return SHARED_CLIENT.send(reqBuilder.build(), HttpResponse.BodyHandlers.ofString());
            } catch (Exception e) {
                throw new RuntimeException("Request execution failed", e);
            }
        }
    }
}
```

### Usage Example:
```java
HttpResponse<String> response = FluentRestClient.builder()
    .url("https://api.example.com/data")
    .header("Authorization", "Bearer token_abc123")
    .get()
    .execute();

System.out.println(response.body());
```

---

## 3. Key Design Considerations for Production

* **JSON Mapping:** The examples above return raw `String` bodies. In a real-world client, you would typically integrate a JSON serialization library like Jackson (`ObjectMapper`) or Gson to automatically map request objects to JSON and parse responses into typed DTO records/classes.
* **Connection Pooling:** `HttpClient` manages its own internal connection pool and thread pool for async requests. Keep your `HttpClient` instance as a singleton across your application rather than instantiating a new one per request.
* **Resilience:** For enterprise-grade reliability, wrap your client execution calls with a resilience library (like Resilience4j) to handle retries, circuit breaking, and rate limiting.