# CI/CD Pipeline Guide: Java Microservice on Kubernetes (AWS)

This guide outlines the end-to-end flow of building, testing, containerizing, and releasing a Java microservice (e.g., Spring Boot) to **Amazon Elastic Kubernetes Service (EKS)** using modern **DevOps** and **GitOps** practices.

---

## 1. High-Level Architecture & Stack

A robust Kubernetes deployment pipeline separates concerns into two distinct workflows:
1. **Continuous Integration (CI):** Triggered by code updates; validates code quality, runs tests, creates a container image, and updates manifest repositories.
2. **Continuous Delivery (CD):** Triggered by manifest updates; pulls state changes from Git and synchronizes them to the Amazon EKS cluster via GitOps.

```
+------------------+       +-------------------+       +--------------------+
|  Java App Code   | ----> | CI Pipeline       | ----> | Amazon ECR         |
|  (App Repo)      |       | (GitHub Actions)  |       | (Container Reg)    |
+------------------+       +-------------------+       +--------------------+
                                     |                           |
                                     v                           |
                           +-------------------+                 |
                           | Manifest Repo     |                 |
                           | (Helm / Kustomize)|                 |
                           +-------------------+                 |
                                     |                           |
                                     v                           v
                           +--------------------------------------------+
                           | CD Engine (ArgoCD inside Amazon EKS)       |
                           +--------------------------------------------+
```

### Recommended Tooling Matrix

| Domain | Selected Tool / Service |
| :--- | :--- |
| **Source Control** | GitHub / GitLab / AWS CodeCommit |
| **CI Engine** | GitHub Actions or AWS CodeBuild |
| **Artifact Registry** | Amazon Elastic Container Registry (ECR) |
| **CD Engine (GitOps)**| ArgoCD or Flux CD |
| **Target Infrastructure**| Amazon Elastic Kubernetes Service (EKS) |
| **Configuration** | Helm Charts or Kustomize |

---

## 2. Continuous Integration (CI) Pipeline Detail

The CI pipeline runs automated checks before producing an immutable deployment artifact.

### Pipeline Steps

1. **Source Code & Trigger**
   * Developer pushes code or opens a Pull Request (PR) against the `main` branch.
2. **Static Application Security Testing (SAST) & Quality Gate**
   * Code is scanned using **SonarQube** or **SpotBugs** for code smells, bugs, and security vulnerabilities.
3. **Build & Test (Java)**
   * **Unit Tests:** Run `./mvnw clean test` or `./gradlew test`.
   * **Integration Tests:** Use **Testcontainers** to spin up lightweight, real dependencies (e.g., PostgreSQL, Redis) in Docker during integration tests.
4. **Containerization & Image Security**
   * Build container image using a multi-stage `Dockerfile` (using distroless runtime images like `eclipse-temurin`).
   * **Vulnerability Scanning:** Scan container layers and Java dependencies (`pom.xml` / `build.gradle`) using **Trivy** or **Snyk**.
5. **Publish Artifact to Amazon ECR**
   * Authenticate with AWS via OIDC (OpenID Connect) and IAM Roles.
   * Push image tagged with Git Commit SHA (e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com/order-service:sha-a1b2c3d`).
6. **GitOps Trigger (Manifest Update)**
   * Update the container image tag in the deployment repository (or Helm values file) via an automated bot commit.

---

## 3. Continuous Delivery (CD) Pipeline Detail

Rather than giving the CI tool direct cluster admin access, the **GitOps pattern** uses an agent inside the cluster (**ArgoCD**) to pull changes.

### Pipeline Steps

1. **Reconciliation & Drift Detection**
   * ArgoCD monitors the Git manifest repository.
   * Detecting a new image tag commit in Git marks the EKS cluster state as `OutOfSync`.
2. **Automated Deployment to Staging**
   * ArgoCD applies updated manifests to the staging namespace on EKS.
   * Kubernetes pulls the new image from Amazon ECR.
3. **Health & Readiness Verification**
   * Kubernetes checks container readiness via probes (`/actuator/health/readiness`).
4. **Production Deployment & Progressive Delivery**
   * **Canary / Blue-Green Rollout:** Managed via **Argo Rollouts** or AWS Load Balancer Controller.
   * 10% of user traffic is routed to the new release.
   * Automated monitoring checks error rates (via CloudWatch / Prometheus) before scaling to 100%.

---

## 4. Production Readiness & Java-Specific Configuration

### A. Memory Management on Kubernetes
Always set container resource limits and instruct the JVM to respect cgroup limits:

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

Configure JVM arguments in Java 11/17/21 to avoid Out-Of-Memory (OOM) kills:
```bash
JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
```

### B. AWS Security (IRSA)
Avoid storing static AWS keys in Kubernetes secrets. Use **IAM Roles for Service Accounts (IRSA)** or **EKS Pod Identity** so your Java microservice securely accesses AWS services (e.g., DynamoDB, S3) using native IAM roles.

### C. Zero-Downtime Graceful Shutdown
Configure Spring Boot and Kubernetes to allow active requests to finish during a pod termination:

```yaml
# Spring Boot application.yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

```yaml
# Kubernetes Deployment Spec
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 45
```