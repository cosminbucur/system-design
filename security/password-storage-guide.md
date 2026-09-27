# Password and Secrets Management in Microservices Architecture

In a microservices architecture, storing and managing passwords and sensitive credentials requires a strict separation of concerns. **User account passwords** (end-user identity) and **system passwords/secrets** (machine-to-machine credentials, API keys, database connection strings) are handled using completely different tools and security patterns.

---

## 1. Storing User Account Passwords

User account passwords should **never** be stored as plain text or simple reversible encryption. Instead, user credentials should be managed by a dedicated **Identity and Access Management (IAM)** service or an isolated **Auth Microservice**.

### Common Storage Locations & Tools
* **Identity Providers (IdP) / IAM Tools:** Dedicated tools like **Keycloak**, **Auth0**, **Okta**, **Firebase Auth**, or **AWS Cognito**.
* **Isolated Auth Database:** If built in-house, user credentials should reside in a single, dedicated database attached *only* to the Authentication Microservice. No other microservice should have direct database access to user password records.

### How Passwords Are Stored
1. **Strong Hashing Functions:** Passwords must be hashed using slow, memory-hard cryptographic algorithms designed to mitigate brute-force and GPU-based attacks:
   * **Argon2** (Argon2id recommended)
   * **bcrypt**
   * **scrypt**
   * **PBKDF2**
2. **Salting:** Each password is paired with a unique, randomly generated cryptographic salt before hashing to prevent rainbow table attacks.
3. **Pepper (Optional):** An additional secret key stored outside the primary database (e.g., in a Secrets Manager) combined with the password prior to hashing.

---

## 2. Storing System Passwords and Secrets

System passwords encompass database credentials, internal service-to-service API keys, message broker tokens, and TLS certificates. These are managed dynamically using centralized **Secret Management Systems**.

### Standard Secret Managers
* **HashiCorp Vault:** Industry-standard central repository for secrets, dynamic credentials, and key management.
* **Cloud Native Secret Managers:**
  * **AWS Secrets Manager** / **AWS Systems Manager Parameter Store**
  * **Azure Key Vault**
  * **Google Cloud Secret Manager**
* **Kubernetes Secrets:** Used natively within Kubernetes clusters (preferably encrypted at rest via KMS integration or sealed using tools like Bitnami Sealed Secrets / External Secrets Operator).

### Key Architectural Patterns for System Passwords

#### A. Dynamic Credentials
Rather than storing static database passwords in microservice configuration files, microservices fetch short-lived, dynamic credentials directly from a secrets manager (e.g., HashiCorp Vault generates database credentials that auto-expire after an hour).

#### B. Injection at Runtime (Environment Variables / Volume Mounts)
Secrets are retrieved from the vault during container startup or deployment and injected into memory:
* **As Environment Variables:** Loaded into container environments dynamically by orchestrators (e.g., Kubernetes Secrets injected into Pod environment).
* **As Mounted Volumes:** Written into an in-memory temporary filesystem (`tmpfs`) as files that the application reads upon startup.

#### C. Centralized Service-to-Service Authentication (mTLS & IAM)
Instead of relying on hardcoded system passwords for microservice communication, modern architectures use:
* **mTLS (Mutual TLS):** Managed via a Service Mesh (e.g., Istio, Linkerd) where services authenticate using short-lived X.509 certificates.
* **IAM Roles / Workload Identity:** Platforms like AWS (IRSA - IAM Roles for Service Accounts) or GCP (Workload Identity) allow pod service accounts to authenticate directly with cloud services without storing static passwords.

---

## Summary Comparison

| Aspect | User Account Passwords | System Passwords & Secrets |
| :--- | :--- | :--- |
| **Primary Location** | Auth Service Database or External IdP (Keycloak, Auth0, AWS Cognito) | Central Secrets Manager (HashiCorp Vault, AWS Secrets Manager, K8s Secrets) |
| **Format** | Salted & Hashed (Argon2, bcrypt) | Encrypted at rest (AES-256) |
| **Access Control** | Accessible only by the Auth Microservice | Injected into microservices at runtime via environment or volume |
| **Rotation** | Triggered by user reset or security policy | Automated key rotation via Secrets Manager or Service Mesh |