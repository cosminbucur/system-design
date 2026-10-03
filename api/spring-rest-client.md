# How to Configure `RestClient` in Spring Boot

Introduced in Spring 6.1 and Spring Boot 3.2, `RestClient` is a modern, synchronous, fluent HTTP client designed to replace the legacy `RestTemplate`. 

This guide covers how to set up, customize, and use `RestClient` in a Spring Boot application.

---

## 1. Basic Configuration (Java-Based Bean)

While you can instantiate `RestClient.create()` on the fly, defining it as a `@Bean` allows Spring to manage it, inject default properties, and make it easily injectable into your services.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;

@Configuration
public class RestClientConfig {

    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder
                .baseUrl("https://api.example.com") // Optional: set a default base URL
                .defaultHeader("Content-Type", "application/json")
                .defaultHeader("Accept", "application/json")
                .build();
    }
}
```

---

## 2. Advanced Configuration (Timeouts & Request Factories)

By default, `RestClient` uses the standard Java `HttpURLConnection`. If you need to configure connection or read timeouts, you can utilize Spring Boot's utility classes to customize the underlying request factory.

```java
import org.springframework.boot.web.client.ClientHttpRequestFactories;
import org.springframework.boot.web.client.ClientHttpRequestFactorySettings;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;
import java.time.Duration;

@Configuration
public class AdvancedRestClientConfig {

    @Bean
    public RestClient customRestClient(RestClient.Builder builder) {
        // Configure connection and read timeouts
        ClientHttpRequestFactorySettings settings = ClientHttpRequestFactorySettings.DEFAULTS
                .withConnectTimeout(Duration.ofSeconds(3))
                .withReadTimeout(Duration.ofSeconds(5));

        return builder
                .baseUrl("https://api.example.com")
                .requestFactory(ClientHttpRequestFactories.get(settings))
                .defaultHeader("X-Custom-Header", "value")
                .build();
    }
}
```

---

## 3. Adding Interceptors (Logging & Authentication)

You can easily add request/response interceptors for cross-cutting concerns like logging, metric tracking, or injecting bearer tokens.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;

@Configuration
public class InterceptorConfig {

    @Bean
    public RestClient secureRestClient(RestClient.Builder builder) {
        return builder
                .baseUrl("https://api.example.com")
                .requestInterceptor((request, body, execution) -> {
                    request.getHeaders().add("Authorization", "Bearer YOUR_TOKEN_HERE");
                    return execution.execute(request, body);
                })
                .build();
    }
}
```

---

## 4. How to Use `RestClient` in a Service

Once configured, inject the `RestClient` bean into your services and use its fluent API to perform HTTP operations.

```java
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
public class MyExternalService {

    private final RestClient restClient;

    public MyExternalService(RestClient restClient) {
        this.restClient = restClient;
    }

    public UserDTO getUser(Long id) {
        return restClient.get()
                .uri("/users/{id}", id)
                .retrieve()
                .body(UserDTO.class);
    }

    public void createUser(UserDTO newUser) {
        restClient.post()
                .uri("/users")
                .body(newUser)
                .retrieve()
                .toBodilessEntity();
    }
}
```

---

## 5. Bonus: Using `RestClient` with HTTP Interfaces

Spring allows you to define external APIs as declarative Java interfaces, significantly reducing boilerplate code.

### Step 1: Define the Interface
```java
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.service.annotation.GetExchange;
import org.springframework.web.service.annotation.HttpExchange;

@HttpExchange("/users")
public interface UserClient {

    @GetExchange("/{id}")
    UserDTO getUserById(@PathVariable("id") Long id);
}
```

### Step 2: Register it as a Bean Using `RestClient`
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.support.RestClientAdapter;
import org.springframework.web.service.invoker.HttpServiceProxyFactory;

@Configuration
public class HttpInterfaceConfig {

    @Bean
    public UserClient userClient() {
        RestClient restClient = RestClient.builder().baseUrl("https://api.example.com").build();
        RestClientAdapter adapter = RestClientAdapter.create(restClient);
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();
        
        return factory.createClient(UserClient.class);
    }
}