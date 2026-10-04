# Understanding Passkeys and Implementing Them in Spring Security

Passkeys represent a modern, phishing-resistant authentication standard developed by the FIDO Alliance and the W3C. They completely replace traditional passwords with cryptographic key pairs tied directly to user devices (such as a smartphone, laptop, or hardware security key).

---

## Part 1: Core Concepts of Passkeys

At the heart of passkeys is **Public-Key Cryptography (Asymmetric Cryptography)** combined with platform authenticators (like Touch ID, Face ID, Windows Hello, or a security PIN). 

### 1. The Cryptographic Key Pair
When a user registers a passkey with a service, their device's authenticator generates a unique key pair:
* **Private Key:** Stored securely and encrypted on the user's local hardware device. It never leaves the device and cannot be exported or stolen via a server breach.
* **Public Key:** Sent to and stored by the service provider's server, associated with the user's account.

### 2. Challenge-Response Authentication
Authentication relies on cryptographic signing rather than transmitting secrets:
* **Registration Ceremony:** The server sends a random challenge to the client device. The device signs this challenge using its newly generated private key and sends the signature back along with the public key. The server stores the public key.
* **Authentication Ceremony:** When logging in, the server generates a new random challenge. The client device prompts the user for local biometric verification (Face ID/Fingerprint). Once verified, the device signs the challenge with the private key and returns the signature. The server verifies this signature using the stored public key.

### 3. Domain Binding (Phishing Resistance)
Passkeys are inextricably bound to the exact origin/domain (e.g., `example.com`) where they were created. Even if a user visits a convincing phishing site (e.g., `examp1e.com`), the browser and operating system will refuse to provide the signature because the domain name does not match the stored registration data.

### 4. Ecosystem Syncing
Passkeys can sync securely across a user’s devices via cloud providers (e.g., iCloud Keychain, Google Password Manager, 1Password), allowing a passkey created on an iPhone to be used on a Mac or PC (often via a secure Bluetooth-proximated QR code scan).

---

## Part 2: Spring Security Implementation

Starting from Spring Security 6.1+, native support for WebAuthn and passkeys has been integrated. Below is an architectural overview of how to configure and handle WebAuthn in a Spring Boot application.

### Step 1: Add Dependencies
Ensure you have the Spring Security Web and WebAuthn support modules in your build configuration:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-security'
implementation 'org.springframework.boot:spring-boot-starter-web'
```

### Step 2: Configure Spring Security Filter Chain
Enable WebAuthn login within your `SecurityFilterChain` bean:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/register", "/login", "/webauthn/**").permitAll()
                .anyRequest().authenticated()
            )
            .webAuthn(webAuthn -> webAuthn
                .rpName("My Application")
                .rpId("localhost") // Set your production domain here
                .allowedOrigins("http://localhost:8080")
            )
            .formLogin(form -> form
                .loginPage("/login").permitAll()
            );

        return http.build();
    }
}
```

### Step 3: Implement the Credential Repository
Spring Security abstracts data persistence through the `WebAuthnCredentialRepository` interface. You must back this with a database (e.g., via Spring Data JPA) to store credential IDs, public keys, and user associations:

```java
import org.springframework.security.web.credential.WebAuthnCredentialRecord;
import org.springframework.security.web.credential.WebAuthnCredentialRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public class JpaWebAuthnCredentialRepository implements WebAuthnCredentialRepository {

    // Inject your Spring Data JPA repository here
    // private final SpringDataWebAuthnRepository repository;

    @Override
    public Optional<WebAuthnCredentialRecord> findByCredentialId(byte[] credentialId) {
        // Query database by credential ID byte array
        return Optional.empty();
    }

    @Override
    public void save(WebAuthnCredentialRecord credentialRecord) {
        // Persist the public key record to the database
    }

    @Override
    public void delete(byte[] credentialId) {
        // Delete the credential record
    }
}
```

### Step 4: Frontend Integration (WebAuthn API)
On the frontend client, use the browser's native `navigator.credentials` API to trigger authenticator prompts:

```javascript
// Example: Initiating Passkey Authentication
async function loginWithPasskey() {
    try {
        // 1. Fetch challenge options from Spring Boot backend
        const response = await fetch('/webauthn/authenticate/options');
        const options = await response.json();

        // 2. Trigger biometric/hardware prompt via browser WebAuthn API
        const credential = await navigator.credentials.get({ publicKey: options });

        // 3. Send signature assertion back to the server for verification
        const result = await fetch('/login/webauthn', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(credential)
        });

        if (result.ok) {
            window.location.href = '/dashboard';
        } else {
            console.error('Authentication failed');
        }
    } catch (err) {
        console.error('Error during passkey login:', err);
    }
}
```

---

## Summary
Passkeys eliminate the vulnerabilities associated with human-created passwords by leveraging secure hardware elements and public-key cryptography. Spring Security simplifies server-side validation via its built-in WebAuthn configuration layer, making it straightforward to implement passwordless experiences.