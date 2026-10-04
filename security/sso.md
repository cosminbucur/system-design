# Single Sign-On (SSO) & Spring Security Guide

Single Sign-On (SSO) is an authentication mechanism that allows a user to log in with a single set of credentials (username and password) to access multiple independent applications.

## Core Concepts of SSO

1. **Identity Provider (IdP):**
   The central trusted system that handles user authentication and maintains user credentials (e.g., Keycloak, Okta, Auth0, Google, Azure AD/Entra ID).

2. **Service Provider (SP):**
   The target application or resource server that the user wants to access (your Spring Boot application).

3. **Protocols (OIDC / OAuth 2.0 / SAML):**

   * **OAuth 2.0:** An authorization framework that lets applications obtain limited access to user accounts on an HTTP service.

   * **OpenID Connect (OIDC):** Built on top of OAuth 2.0, OIDC adds an **identity layer** specifically designed for authentication, issuing tokens (like JSON Web Tokens or JWTs) containing user profile details.

   * **SAML 2.0:** An older, XML-based standard commonly used in enterprise/B2B environments.

4. **The SSO Flow (OIDC/OAuth2 Example):**

   * The user attempts to access a protected page on the Service Provider (your app).

   * The app recognizes that the user is unauthenticated and redirects them to the IdP.

   * The user logs in successfully at the IdP (if not already logged in).

   * The IdP redirects the user back to the application with an authorization code.

   * The application exchanges this code behind-the-scenes for an **Access Token** and an **ID Token**, then establishes a local user session.

## Spring Security Code Examples

Modern Spring Security makes implementing OAuth2/OIDC Login exceptionally straightforward using `spring-boot-starter-oauth2-client`.

### 1. Maven Dependencies (`pom.xml`)

```
<dependencies>
    <!-- Spring Boot Web & Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <!-- OAuth2 Client for SSO Support -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>
</dependencies>

```

### 2. Application Configuration (`application.yml`)

Configure your Identity Provider parameters (e.g., Keycloak, Okta, or Google):

```
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: "your-google-client-id"
            client-secret: "your-google-client-secret"
            scope: openid, profile, email

```

### 3. Security Configuration Class (`SecurityConfig.java`)

Using modern Spring Security (Lambda-based configuration style):

```
package com.example.demo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/", "/public/**").permitAll() // Public endpoints
                .anyRequest().authenticated()                   // All other URLs require SSO login
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/oauth2/authorization/google")    // Custom login redirect if needed
            );
            
        return http.build();
    }
}

```

### 4. Accessing Authenticated User Information in a Controller

Once logged in via SSO, Spring Security populates the security context with an `OAuth2User` or `Jwt` token, which you can read inside your controllers:

```
package com.example.demo.controller;

import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
public class UserController {

    @GetMapping("/")
    public String home() {
        return "Welcome! Public home page. Go to /dashboard to test SSO.";
    }

    @GetMapping("/dashboard")
    public Map<String, Object> dashboard(@AuthenticationPrincipal OAuth2User principal) {
        // principal.getAttributes() contains the user claims returned by the IdP (Name, Email, Picture, etc.)
        return principal.getAttributes();
    }
}

```