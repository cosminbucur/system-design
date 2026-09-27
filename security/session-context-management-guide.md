# Java Session & Authentication Context Management (Non-Spring)

This document provides a comprehensive guide on managing authenticated user sessions and context propagation in plain Java applications, covering standard Servlet Sessions, Stateless Bearer Tokens (JWT), thread-safe `ThreadLocal` patterns, and modern Java 21 Scoped Values.

---

## 1. State Management Strategies

When dealing with authenticated users in non-Spring Java applications, authentication states generally fall into two categories:

| Strategy | Storage Location | Scalability | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **Stateful (Session-based)** | Server Memory / Redis (`HttpSession`) | Requires session stickiness or distributed storage | Traditional server-rendered web applications (JSP, Thymeleaf, WebFilters) |
| **Stateless (Token-based)** | Client-side (JWT / OAuth2 Bearer Token) | High / Horizontally Scalable | REST APIs, Single Page Applications (SPAs), Mobile Backends |

---

## 2. Stateful Management: `HttpSession`

In traditional Java Web applications running on standard Servlet Containers (Tomcat, Jetty, WildFly), user sessions are managed via the container's built-in `HttpSession`.

### Writing to and Reading from `HttpSession`

```java
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;
import java.io.IOException;

@WebServlet("/login")
public class LoginServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) 
            throws IOException {
        
        // Authenticate credentials against database
        User user = authenticate(request.getParameter("username"), request.getParameter("password"));

        if (user != null) {
            // Create a new session (or get existing)
            HttpSession session = request.getSession(true);
            
            // Prevent Session Fixation attacks by invalidating old session ID if present
            request.changeSessionId();
            
            // Store user object or principal in session
            session.setAttribute("user", user);
            
            response.sendRedirect("/dashboard");
        } else {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid credentials");
        }
    }
}
```

### Retrieving Authenticated User in Later Requests

```java
@WebServlet("/dashboard")
public class DashboardServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) 
            throws IOException {
        
        HttpSession session = request.getSession(false); // Do not create if it doesn't exist

        if (session != null && session.getAttribute("user") != null) {
            User user = (User) session.getAttribute("user");
            response.getWriter().write("Welcome back, " + user.getUsername());
        } else {
            response.sendRedirect("/login");
        }
    }
}
```

---

## 3. Context Propagation Across Layers

Passing `HttpServletRequest` or `HttpSession` through every service, repository, and domain layer creates strong coupling with the Web API. To keep business logic independent, applications decouple user context using execution-scoped holders.

### Safety Comparison: `ThreadLocal` vs. `ScopedValue`

| Feature | `ThreadLocal` | `ScopedValue` (Java 21+) |
| :--- | :--- | :--- |
| **Mutability** | Mutable (`set()`, `remove()`) | Immutable (Value fixed once bound) |
| **Lifetime** | Unbounded (Risk of memory leak if not removed) | Bounded (Strictly tied to execution scope) |
| **Thread Pool Safety** | High Risk (Reused worker threads retain state) | Zero Risk (Automatically unbinds when scope ends) |
| **Virtual Threads** | High Overhead | Native Optimization |

---

## 4. Implementation: Safe `ThreadLocal` Pattern

To prevent memory leaks and user cross-contamination in pooled environments (such as Servlet containers), `ThreadLocal` cleaning **must** be enforced within a `try-finally` block inside a Servlet Filter.

### The Context Holder

```java
public final class UserContext {

    private static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();

    private UserContext() {}

    public static void setCurrentUser(User user) {
        CURRENT_USER.set(user);
    }

    public static User getCurrentUser() {
        return CURRENT_USER.get();
    }

    public static void clear() {
        CURRENT_USER.remove(); // Removes the key/value pair from ThreadLocalMap
    }
}
```

### The Enforcing Filter

```java
import jakarta.servlet.*;
import jakarta.servlet.annotation.WebFilter;
import jakarta.servlet.http.HttpServletRequest;
import java.io.IOException;

@WebFilter("/*")
public class UserContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        
        HttpServletRequest request = (HttpServletRequest) req;

        try {
            // Extract user from Session or Token
            User user = resolveUser(request);
            if (user != null) {
                UserContext.setCurrentUser(user);
            }

            // Continue processing request
            chain.doFilter(req, res);
            
        } finally {
            // CRITICAL: Always clear context to avoid leaking user data to reused pool threads
            UserContext.clear();
        }
    }

    private User resolveUser(HttpServletRequest request) {
        // Implementation checking Session or Authorization Header
        return (User) request.getSession(false) != null ? 
               (User) request.getSession(false).getAttribute("user") : null;
    }
}
```

---

## 5. Modern Implementation: Scoped Values (Java 21+)

For modern applications running Java 21 or later, `ScopedValue` replaces `ThreadLocal` for safer and cleaner immutable context propagation.

```java
import java.lang.ScopedValue;

public final class AppContext {
    public static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();
}
```

### Binding Scope in Filter/Middleware

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    User user = resolveUser((HttpServletRequest) req);

    if (user != null) {
        // Bind the user value for the duration of the run block
        ScopedValue.where(AppContext.CURRENT_USER, user).run(() -> {
            try {
                chain.doFilter(req, res);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }); 
        // Automatically cleared once execution exits this block
    } else {
        chain.doFilter(req, res);
    }
}
```

### Consuming in Deep Business Logic

```java
public class OrderService {

    public void createOrder() {
        if (AppContext.CURRENT_USER.isBound()) {
            User user = AppContext.CURRENT_USER.get();
            System.out.println("Processing order for: " + user.getUsername());
        } else {
            throw new IllegalStateException("Unauthenticated user context");
        }
    }
}
```