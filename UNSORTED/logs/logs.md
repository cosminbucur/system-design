stdin - 0
stdout - 1
sterr - 2

```sh
# separate files
python script.py > output.log 2> error.log

# combined files
python script.py &> combined.log
python script.py > combined.log 2>&1
```

# Main Types of Logs

- Application Logs: Track software runtime events, custom messages, and internal states. Developers use them for debugging, performance tracking, and operational insights.
- Audit Logs: Record critical user actions or security changes (like permission updates or record deletions) to ensure accountability and meet regulatory compliance.
- System Logs: Capture core operating system and infrastructure events such as hardware statuses, driver loads, and server reboots.
- Security Logs: Monitor authentication attempts, successful or failed logins, privilege escalations, and potential cyber threats.
- Access Logs: Log every inbound request or resource interaction, tracking who accessed a specific file, URL, or database and when.
- Error Logs: Highlight software faults, application crashes, unhandled exceptions, and system failures.

| Log Type         | Primary Source                            | Architecture Layer          | Example Events                                                         |
| ---------------- | ----------------------------------------- | --------------------------- | ---------------------------------------------------------------------- |
| Access Logs      | API Gateway, Reverse Proxy, Load Balancer | Edge Layer / Infrastructure | HTTP 200/404, request path, IP addresses, response times.              |
| Audit Logs       | Microservice (Business Logic) or IAM      | Application / Service Layer | "User X changed their email," "Admin Y changed permissions."           |
| Application Logs | Microservice                              | Application / Service Layer | "Database connection established," "Payment processed successfully."   |
| Error Logs       | Microservice, Backend Workers, Database   | All Layers                  | Unhandled exceptions, database timeouts, out-of-memory faults.         |
| Security Logs    | Identity Provider (IdP), Firewalls, WAF   | Edge / Identity Layer       | Failed login attempts, brute-force detections, blocked SQL injections. |
| System Logs      | OS, Containers (K8s), VM Hosts            | Infrastructure Layer        | Container restarts, high CPU alerts, disk space exhaustion.            |
