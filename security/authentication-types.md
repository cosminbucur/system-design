# Authentication Types in Spring Boot

Spring Boot applications handle authentication via **Spring Security**, which offers robust, highly customizable options depending on your application's architecture (monolith vs. microservices, web app vs. REST API). 

---

## 1. Traditional & Server-Driven Authentication

* **HTTP Basic Authentication**: 
  * *How it works:* Credentials (username and password) are sent in the `Authorization` header encoded in Base64 for every request. 
  * *Use case:* Simple API testing, internal machine-to-machine services, or quick setups. **Note:** Insecure unless paired strictly with HTTPS.
* **Form-Based Authentication**: 
  * *How it works:* Provides a traditional HTML login page. Upon successful authentication, the server creates an `HttpSession` and tracks the user via a session cookie (`JSESSIONID`). 
  * *Use case:* Server-rendered web applications (e.g., using Thymeleaf, JSP, or Freemarker).
* **Remember-Me Authentication**: 
  * *How it works:* Extends form-based or session login by allowing users to stay logged in across browser sessions using a secure, persistent cookie or database-backed token.

---

## 2. Token-Based & Stateless Authentication (REST APIs)

* **JWT (JSON Web Tokens) / Bearer Token Authentication**: 
  * *How it works:* The server authenticates user credentials once and returns a signed, self-contained JWT. The client sends this token in the `Authorization: Bearer <token>` header for subsequent requests. Because the token holds user claims, the server can validate it statelessly without looking up an active session in memory.
  * *Use case:* Modern Single Page Applications (SPAs like React, Angular, Vue) and stateless REST APIs.
* **API Key Authentication**: 
  * *How it works:* A unique alphanumeric string is passed via custom headers (e.g., `X-API-KEY`) or query parameters and validated against a database or configuration store.
  * *Use case:* Public developer APIs, third-party integrations, or system-to-system calls.

---

## 3. Federated & Enterprise Identity (OAuth 2.0 / OIDC / SAML)

* **OAuth 2.0 / OpenID Connect (OIDC) Login**: 
  * *How it works:* Allows users to log in via external identity providers like Google, GitHub, Okta, or Keycloak. Spring Security handles the Authorization Code Flow, retrieving an ID token and managing user sessions or tokens.
  * *Use case:* Social logins ("Sign in with Google") and consumer-facing or modern enterprise web apps.
* **OAuth 2.0 Resource Server**: 
  * *How it works:* If your application is a backend API that receives tokens minted by an external authorization server (like Keycloak or Auth0), you configure it as a Resource Server to validate incoming JWTs or opaque tokens.
  * *Use case:* Microservice architectures where authentication is delegated to a dedicated auth server.
* **SAML 2.0 Login**: 
  * *How it works:* Applications integrate with corporate identity providers via SAML assertions using XML protocols.
  * *Use case:* Enterprise B2B software and Single Sign-On (SSO) environments integrated with corporate directories like Azure AD, Okta, or Ping Identity.

---

## 4. Alternative & Specialized Methods

* **LDAP / Active Directory**: Connects Spring Security directly to an enterprise directory service to authenticate corporate users against directory entries.
* **X.509 Client Certificate Authentication**: Uses public key infrastructure (PKI) where clients present a digital SSL/TLS certificate to authenticate. Commonly used for high-security, closed financial or government APIs.
* **Custom / Pre-Authentication**: Allows integration with gateway-level or reverse-proxy authentication setups (e.g., API Gateways, OAuth proxies, or SiteMinder) where authentication happens upstream and Spring Security trusts the incoming headers.

---

## Choosing the Right Approach
* **Monolithic Web App with Server Views:** Form-Based + Remember-Me.
* **Stateless REST API:** JWT / Bearer Token.
* **Enterprise Application / Internal Tools:** LDAP, Active Directory, or SAML 2.0.
* **Modern Cloud-Native / Microservices:** OAuth 2.0 / OIDC (Login + Resource Server).