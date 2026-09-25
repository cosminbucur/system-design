# Architectural Layers Reference

Modern system architectures are conceptually divided into four primary layers: the **Client Layer**, the **Edge/Ingress Layer**, the **Core Application Layer**, and the **Core Infrastructure & Control Plane Layer**.

---

## 1. The Core Infrastructure & Control Plane Layer

This layer is the **brain and nervous system** of the architecture. It does not handle user data directly; instead, it manages, configures, coordinates, and monitors the systems that do.

- **Service Discovery & Registries:** Tracks the IP addresses and health of all active services (e.g., Consul, Eureka).
- **Container Orchestration & Scheduling:** Manages the deployment, scaling, and lifecycle of containers (e.g., Kubernetes Control Plane, including `kube-apiserver` and `etcd`).
- **Configuration Management:** Stores and distributes runtime settings to applications (e.g., Vault, Spring Cloud Config).
- **Distributed Consensus & State Storage:** Maintains the "source of truth" configuration for the cluster (e.g., Zookeeper, etcd).
- **Observability & Monitoring Backends:** Collects, aggregates, and stores metrics, logs, and traces (e.g., Prometheus, Grafana Loki, Jaeger).
- **Internal Secret Management:** Securely stores and injects API keys, database credentials, and SSL certificates (e.g., HashiCorp Vault).

---

## 2. The Edge & Ingress Layer

This is the **secure gateway** and entryway into the system. It sits at the perimeter to interface between external clients and internal services.

- **API Gateways:** Handles request routing, authentication, rate limiting, and protocol translation (e.g., Kong, Apigee).
- **Load Balancers (External):** Distributes incoming traffic across multiple edge servers or availability zones (e.g., AWS ALB, NGINX).
- **Content Delivery Networks (CDNs):** Caches static assets physically closer to the user to reduce latency (e.g., Cloudflare, Akamai).
- **Web Application Firewalls (WAF):** Inspects incoming traffic to block malicious exploits like SQL injections or DDoS attacks.
- **Edge Compute / Serverless:** Executes lightweight code directly at edge locations (e.g., Cloudflare Workers).

---

## 3. The Core Application Layer (Data Plane)

This is where the **business logic lives**. It executes the actual code required to fulfill user requests and process data.

- **Microservices & Monoliths:** The actual backend application code (e.g., a Checkout Service, a User Profile Service).
- **Internal Load Balancers:** Distributes traffic between internal services (East-West traffic).
- **Service Meshes (Data Plane):** Sidecar proxies that handle secure service-to-service communication, retries, and circuit breaking (e.g., Istio Envoy sidecars).
- **Asynchronous Workers:** Background processes that handle long-running tasks, like generating PDFs or processing images.

---

## 4. The Data & Storage Layer

Often categorized as part of the core infrastructure, this layer is dedicated to **data persistence and state**.

- **Primary Databases:** Relational and non-relational storage (e.g., PostgreSQL, MongoDB).
- **Caching Layers:** In-memory data structures used to speed up database queries (e.g., Redis, Memcached).
- **Message Brokers & Event Streaming:** Handles asynchronous communication between microservices (e.g., Apache Kafka, RabbitMQ).

---

## Layer Comparison Matrix

| Layer                        | Primary Responsibility                           | Key Example Component                       |
| :--------------------------- | :----------------------------------------------- | :------------------------------------------ |
| **Edge / Ingress**           | Perimeter security, routing, and traffic shaping | API Gateway, CDN, WAF                       |
| **Core Application**         | Executing business logic and user requests       | Payment Microservice, Internal Proxies      |
| **Data & Storage**           | Persisting state and handling async data streams | PostgreSQL, Kafka, Redis                    |
| **Infrastructure & Control** | Coordinating, scaling, and monitoring the system | Kubernetes Control Plane, Service Discovery |
