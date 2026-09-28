# PKCS and Using It in Spring Boot

## What is PKCS (Pixie)?

**PKCS** stands for **Public-Key Cryptography Standards**. Developed by RSA Laboratories in the early 1990s in collaboration with industry leaders (such as Apple, Microsoft, and Sun), PKCS is a suite of specifications designed to standardise and deploy public-key infrastructure (PKI) across disparate software applications and operating systems.

Rather than being a single technology, PKCS comprises numbered specifications (PKCS #1 through PKCS #15). The most prominent standards encountered in modern Java and Spring applications include:

- **PKCS #1:** Specifies the mathematical and formatting standards for RSA encryption, decryption, and digital signatures.
- **PKCS #7 / CMS (Cryptographic Message Syntax):** Standardises signed, enveloped, and encrypted data structures (commonly used in S/MIME email security and digital certificates).
- **PKCS #8:** Specifies the standard format for storing **private key information** (either encrypted or unencrypted).
- **PKCS #12:** Defines a portable, password-protected archive format for storing a **private key alongside its associated public key certificate chain**. In Java, this standard is represented by `.p12` or `.pfx` extensions and serves as a KeyStore.

---

## Using PKCS in Spring Applications

In Spring and Spring Boot applications, PKCS standards are most commonly used for two purposes:

1. **Setting up HTTPS/TLS using PKCS #12 (`.p12`) archives.**
2. **Loading PKCS #8 formatted private keys for JWT/OAuth2 signing.**

---

### 1. Configuring HTTPS in Spring Boot (PKCS #12)

Java applications historically used the proprietary Java KeyStore (`JKS`) format. Modern Spring Boot applications standardise on **PKCS #12** (`.p12` or `.pfx`) for SSL/TLS configuration because it is an open, cross-language industry standard.

#### Step 1: Generate a PKCS #12 KeyStore

You can create a self-signed PKCS #12 keystore using Java's `keytool`:

```bash
keytool -genkeypair \
  -alias spring-app \
  -keyalg RSA \
  -keysize 2048 \
  -storetype PKCS12 \
  -keystore keystore.p12 \
  -validity 365
```

#### Step 2: Configure Spring Boot Properties

Place `keystore.p12` in your `src/main/resources/` directory and add the configuration to `application.yml` (or `application.properties`):

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store-type: PKCS12
    key-store: classpath:keystore.p12
    key-store-password: your_secure_password
    key-alias: spring-app
```

---

### 2. Loading PKCS #8 Private Keys in Spring Security

When implementing custom OAuth2 Authorization Servers, JWT signing, or asymmetric encryption, private keys are typically provided in **PKCS #8** format (often encoded as Base64 PEM files starting with `-----BEGIN PRIVATE KEY-----`).

#### Programmatic Key Loading Example

```java
package com.example.security.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.nio.file.Files;
import java.nio.file.Paths;
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.spec.PKCS8EncodedKeySpec;
import java.util.Base64;

@Configuration
public class SecurityKeyConfig {

    @Bean
    public PrivateKey privateKey() throws Exception {
        // Read the PKCS #8 PEM file content
        String keyPem = Files.readString(Paths.get("src/main/resources/private_key.pem"));

        // Strip PEM headers/footers and whitespace
        String privateKeyPEM = keyPem
                .replace("-----BEGIN PRIVATE KEY-----", "")
                .replace("-----END PRIVATE KEY-----", "")
                .replaceAll("\\s", "");

        // Base64 decode to get raw PKCS #8 bytes
        byte[] encoded = Base64.getDecoder().decode(privateKeyPEM);

        // Parse key using Java's PKCS8EncodedKeySpec
        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(encoded);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");

        return keyFactory.generatePrivate(keySpec);
    }
}
```

---

## PKCS Quick Reference for Spring Developers

| Standard     | File Extensions          | Primary Spring Usage                                                                      |
| :----------- | :----------------------- | :---------------------------------------------------------------------------------------- |
| **PKCS #8**  | `.pem`, `.key`, `.pkcs8` | Loading asymmetric private keys into `KeyFactory` for JWT/OAuth2 signing.                 |
| **PKCS #12** | `.p12`, `.pfx`           | Configuring SSL/TLS keystores (`server.ssl.key-store-type=PKCS12`) and mutual TLS (mTLS). |
