# Comprehensive Guide to Authorization and Spring Security Implementations

Authorization is the process of giving a user, system, or service permission to access specific resources or perform specific actions. Unlike *authentication* (which verifies *who* you are), authorization determines *what* you are allowed to do.

---

## Part 1: Types of Authorization Models

### 1. Role-Based Access Control (RBAC)
* **How it works:** Permissions are assigned to specific **roles** (e.g., `Admin`, `Editor`, `Viewer`), and users are assigned to those roles. A user inherits all the permissions associated with their assigned role.
* **Best for:** Applications with structured, predictable user hierarchies where permissions rarely need to be tailored to individual users.
* **Example:** In a blog platform, an `Admin` can delete any post, an `Editor` can publish posts, and a `Viewer` can only read posts.

### 2. Attribute-Based Access Control (ABAC)
* **How it works:** Access decisions are made dynamically by evaluating a combination of **attributes** (or policies)—including user attributes, resource attributes, action attributes, and environmental context (such as time of day, IP address, or location).
* **Best for:** Complex systems requiring fine-grained, context-aware security policies.
* **Example:** A doctor can view a patient's medical record *only if* the doctor is on duty, the patient is assigned to their ward, and the access request happens during working hours.

### 3. Discretionary Access Control (DAC)
* **How it works:** The **owner** of a resource has full control over it and can decide at their own discretion who is allowed to access it and what permissions they get.
* **Best for:** Operating systems and file-sharing environments (like Google Drive or standard Unix file permissions) where individual users manage their own files.
* **Example:** You create a document and explicitly share it with a coworker, giving them "Editor" access while keeping it private from everyone else.

### 4. Mandatory Access Control (MAC)
* **How it works:** Users do not have control over resource permissions. Instead, access is strictly regulated by a central authority or operating system based on a **security labeling system** (clearance levels). 
* **Best for:** Highly secure environments, such as military, government, or defense systems.
* **Example:** A user with a "Secret" security clearance cannot access a document labeled "Top Secret," regardless of who created it.

### 5. Rule-Based Access Control (RuBAC)
* **How it works:** Access is granted or denied based on a set of **system-wide rules** defined by an administrator, often looking at network or environmental conditions rather than individual user attributes or roles.
* **Best for:** Network security, firewalls, and routing policies.
* **Example:** All traffic originating from a specific IP address range outside the company's country is automatically blocked from accessing internal admin tools.

### 6. Token-Based Authorization (OAuth / JWT)
* **How it works:** Instead of checking a database for every request, the user authenticates once and receives a cryptographically signed token (like a **JSON Web Token / JWT**) or an OAuth access token. This token securely carries the user's identity and claims/permissions, which downstream services verify instantly.
* **Best for:** Modern distributed systems, microservices, and single-page applications (SPAs).
* **Example:** Logging into a mobile app via Google; Google issues an OAuth token that grants the app permission to read your basic profile without sharing your password.

---

### Quick Comparison Table

| Authorization Type | Primary Decision Factor | Flexibility | Complexity | Common Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **RBAC** | User roles | Moderate | Low | Enterprise software, SaaS dashboards |
| **ABAC** | Dynamic attributes & context | Very High | High | Healthcare, finance, cloud security policies |
| **DAC** | Resource owner's choice | High | Low | File systems, personal cloud storage |
| **MAC** | Central security labels/clearance | Low | High | Government, defense, classified systems |
| **RuBAC** | System-wide rules & conditions | Moderate | Medium | Firewalls, API gateways, routing |
| **Token-Based** | Cryptographic claims & scopes | High | Medium | Microservices, APIs, third-party integrations |

---

## Part 2: Spring Security Implementations

Spring Security provides powerful, flexible mechanisms to implement different authorization models. Below are code implementations using modern Spring Boot / Spring Security 6+ syntax.

### 1. HTTP Endpoint Security (Filter Chain DSL)
The most common approach is configuring URL-based access control inside your `SecurityFilterChain` bean. This intercepts incoming HTTP requests and matches them against roles or authorities.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                // Public endpoints that require no authentication
                .requestMatchers("/api/public/**", "/login", "/css/**").permitAll()
                
                // Role-based matching
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                
                // Multiple options or fine-grained authority matching
                .requestMatchers(HttpMethod.POST, "/api/posts/**").hasAnyAuthority("WRITE_POST", "ROLE_ADMIN")
                
                // Any other request must be authenticated
                .anyRequest().authenticated()
            )
            .httpBasic(httpBasic -> {}); // Or formLogin(), oauth2ResourceServer(), etc.
            
        return http.build();
    }
}
```

### 2. Method-Level Security (RBAC / ABAC via Annotations)
Instead of restricting entire URLs, you can secure individual service or repository methods. This is ideal for domain-driven design, ensuring that rules are enforced regardless of *how* the service method is invoked.

First, enable method security on a configuration class:
```java
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

@Configuration
@EnableMethodSecurity // Enables @PreAuthorize, @PostAuthorize, etc.
public class MethodSecurityConfig {
}
```

Then, use annotations like `@PreAuthorize` combined with Spring Expression Language (SpEL) to check roles, authorities, or even method parameters:

```java
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;

@Service
public class DocumentService {

    // Restrict strictly by role
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAllDocuments() {
        // ... deletion logic
    }

    // Combine roles and evaluate method arguments (ABAC-style check)
    @PreAuthorize("hasRole('ADMIN') or #username == authentication.name")
    public UserProfile updateProfile(String username, UserProfile profile) {
        // Users can update their own profile; admins can update anyone's
        return profile;
    }
}
```

### 3. Custom Security Expressions (Advanced ABAC / Context-Aware)
When your authorization rules depend on complex business logic (e.g., *"Does this user belong to the same organization as the requested resource?"*), you can write custom evaluator beans.

**Step 1: Define a custom evaluator component**
```java
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;

@Component("customSecurity")
public class CustomSecurityExpressions {

    public boolean canAccessResource(Authentication authentication, Long resourceId) {
        String currentUsername = authentication.getName();
        // Perform a database lookup or business check here
        boolean isOwner = checkUserOwnsResource(currentUsername, resourceId);
        boolean isAdmin = authentication.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
                
        return isOwner || isAdmin;
    }

    private boolean checkUserOwnsResource(String username, Long resourceId) {
        // mock logic
        return true; 
    }
}
```

**Step 2: Reference it inside `@PreAuthorize`**
```java
@Service
public class ResourceService {

    @PreAuthorize("@customSecurity.canAccessResource(authentication, #resourceId)")
    public ResourceDTO getResourceDetails(Long resourceId) {
        return new ResourceDTO(resourceId, "Confidential Data");
    }
}
```

### 4. Stateless JWT Token Authorization (Microservices / APIs)
For modern token-based architectures (like SPAs or distributed microservices), you parse incoming JSON Web Tokens (JWTs) and map their claims directly into Spring Security's `GrantedAuthority` collection using an `JwtAuthenticationConverter`.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class JwtSecurityConfig {

    @Bean
    public SecurityFilterChain jwtFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/public/auth/**").permitAll()
                .anyRequest().hasRole("USER")
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
            );

        return http.build();
    }

    // Customizes how JWT claims (e.g., "scopes" or "roles") map to Spring roles
    private JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthoritiesClaimName("roles"); // JSON claim name in token
        authoritiesConverter.setAuthorityPrefix("ROLE_");     // Prefix expected by hasRole()

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return converter;
    }
}