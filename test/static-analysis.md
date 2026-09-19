Static Application Security Testing (SAST) analyzes source code without running it, looking for security vulnerabilities, bugs, and quality issues directly in the syntax/structure of the code — catching problems at commit/PR time, before they ever reach a running system. SonarQube is the dominant all-in-one SAST/code-quality platform in the Java ecosystem, and Snyk is the dominant developer-first tool for dependency (SCA) and container/IaC security — most mature pipelines run both, since they overlap only partially. This note uses both as the concrete examples, but the underlying practice (shift security/quality checks left, gate the pipeline on them) applies to any SAST tool.

## 1. SAST vs. SCA vs. DAST — Know Which Problem Each Solves

These three are often mentioned together and easy to conflate, but they analyze fundamentally different things.

| Type                                | Analyzes                                   | Catches                                                                                                   | Example tool                                         |
| ----------------------------------- | ------------------------------------------ | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| SAST (Static)                       | Your own source code, without executing it | Insecure code patterns, bugs, code smells                                                                 | SonarQube, Snyk Code, Semgrep, SpotBugs              |
| SCA (Software Composition Analysis) | Your third-party dependencies              | Known CVEs in libraries you depend on                                                                     | Snyk Open Source, OWASP Dependency-Check, Dependabot |
| DAST (Dynamic)                      | A running instance of the application      | Vulnerabilities only visible at runtime (misconfigured headers, actual injection against a live endpoint) | OWASP ZAP, Burp Suite                                |

A mature pipeline runs all three — they find different, non-overlapping classes of problems. This note covers SAST and SCA/container/IaC scanning; DAST is a distinct, complementary practice worth its own tooling even though it's often bundled into the same "AppSec" conversation.

## 2. SonarQube's Finding Categories

SonarQube classifies every finding into one of a few types, which matters for how a team should triage and prioritize them.

| Category         | Meaning                                                                                                           | Example                                                                                                                                |
| ---------------- | ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Bug              | Code that will misbehave — a real correctness defect                                                              | `equals()` overridden without `hashCode()`                                                                                             |
| Vulnerability    | An exploitable security weakness with clear evidence                                                              | Building a SQL query via string concatenation with user input                                                                          |
| Security Hotspot | Security-sensitive code that _might_ be a problem — needs a human to review the context, not an automatic verdict | Use of a cryptographic API that's fine in one context and dangerous in another (e.g., a fast hash used for a checksum, not a password) |
| Code Smell       | Maintainability issue, not a bug or vulnerability                                                                 | A method with excessive cognitive complexity, duplicated code blocks                                                                   |

The distinction between **Vulnerability** and **Security Hotspot** matters operationally: a Vulnerability is treated as a confirmed defect to fix; a Security Hotspot requires a developer to actually look at the code and mark it "Safe" or "Fixed" — SonarQube deliberately doesn't auto-resolve these, because the same API call can be safe or dangerous depending on surrounding context that only a human can judge.

## 3. Real Examples of What SonarQube Catches in Java

```java
// VULNERABILITY: SQL injection — user input concatenated directly into a query
String query = "SELECT * FROM accounts WHERE owner_name = '" + ownerName + "'";
statement.executeQuery(query);

// FIX: parameterized query
PreparedStatement ps = connection.prepareStatement("SELECT * FROM accounts WHERE owner_name = ?");
ps.setString(1, ownerName);
```

```java
// VULNERABILITY: hardcoded credentials committed directly into source
String password = "admin123"; // SonarQube flags string literals matching credential-like patterns

// FIX: externalize via a secrets manager
String password = secretsClient.getSecret("db-password");
```

```java
// SECURITY HOTSPOT: weak/broken hashing algorithm
MessageDigest md = MessageDigest.getInstance("MD5"); // flagged — context determines if this is actually dangerous

// If this MD5 use was for password hashing, the fix is BCrypt/Argon2.
// If it was genuinely just a non-security checksum (e.g., detecting accidental file corruption),
// marking the Hotspot "Safe" with a comment explaining why is the correct resolution — not a code change.
```

```java
// BUG: equals() without hashCode() — breaks HashMap/HashSet lookups
public class OrderId {
    @Override
    public boolean equals(Object o) { ... }
    // missing hashCode() override — SonarQube flags this as a correctness bug, not just style
}
```

## 4. The SonarQube Pipeline: Scanner, Server, Quality Gate

```
Developer commits code
   → SonarScanner runs (Maven/Gradle plugin or standalone CLI) during CI, analyzing the codebase
   → Results uploaded to the SonarQube server
   → Quality Gate evaluates the results against defined conditions
   → Pass: CI proceeds. Fail: the build (or PR merge) is blocked.
```

```xml
<!-- Maven -->
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
</plugin>
```

```bash
mvn verify sonar:sonar \
    -Dsonar.projectKey=order-service \
    -Dsonar.host.url=https://sonarqube.example.com \
    -Dsonar.token=$SONAR_TOKEN
```

A **Quality Profile** defines _which rules_ are checked (and at what severity) for a given language; a **Quality Gate** defines _pass/fail conditions_ evaluated against a scan's results (e.g., "zero new Vulnerabilities," "code coverage on new code ≥ 80%," "zero new Bugs rated Blocker/Critical"). The Quality Gate is what actually gates CI/CD — a scan can run and report findings without blocking anything unless a Quality Gate condition is attached and wired into the build.

## 5. "Clean as You Code" — New Code vs. Overall Code

A large, pre-existing codebase can easily have thousands of historical findings — treating all of them as build-blocking on day one would make adopting SonarQube in an existing project practically impossible. The standard, recommended approach is the **New Code** period: the Quality Gate evaluates only code changed since a defined baseline (a specific date, a release tag, or "since the previous version"), not the entire codebase's accumulated history.

```
Quality Gate condition (typical default):
  - New code: 0 new Bugs, 0 new Vulnerabilities, coverage on new code ≥ 80%, duplicated lines on new code < 3%
  - Overall code: reported for visibility, but doesn't block the build
```

This means: legacy code with pre-existing issues doesn't block anyone, but every _new_ line of code is held to a real standard — the codebase's overall health improves incrementally as it's naturally touched over time, rather than requiring a dedicated, separately-prioritized remediation project. This is the SonarQube-specific instantiation of the Boy Scout Rule: leave the code you touch better, enforced automatically rather than left to individual discipline.

## 6. Snyk: Developer-First Security Across the Stack

Where SonarQube's center of gravity is code quality plus in-house code vulnerabilities, Snyk's is dependency vulnerabilities (SCA), with container image and Infrastructure-as-Code scanning as first-class citizens alongside its own SAST offering (Snyk Code) — most teams end up running Snyk specifically _because_ CVEs in third-party libraries are a different, more constantly-shifting problem than in-house code quality: a dependency can become vulnerable overnight with no code change on your side at all, the moment a new CVE is published against it.

```bash
# SCA: scan project dependencies for known vulnerabilities
snyk test

# SAST: scan your own source code (Snyk Code — separate engine from the SCA scan above)
snyk code test

# Container: scan a built image for OS-package and base-image CVEs
snyk container test myregistry.example.com/order-service:1.4.2

# IaC: scan Terraform/Kubernetes manifests for misconfigurations
snyk iac test ./k8s/deployment.yaml

# Continuous monitoring: snapshot the current dependency tree so Snyk alerts you
# the moment a NEW CVE is published against something you already depend on —
# not just at the moment you happen to re-scan
snyk monitor
```

```yaml
# CI integration (GitHub Actions example) — fails the build on new high-severity findings
- name: Run Snyk to check for vulnerabilities
  uses: snyk/actions/maven@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

Two things Snyk emphasizes that a typical SCA tool doesn't do as automatically:

- **Auto-remediation pull requests**: for a vulnerable dependency, Snyk can open a PR bumping it to the minimum non-vulnerable version automatically, rather than just reporting "you have a CVE" and leaving the fix as manual work.
- **`snyk monitor` for drift over time**: a one-time `snyk test` only reflects vulnerabilities known _at scan time_ — `snyk monitor` keeps watching a snapshot of your dependency tree so you're alerted the moment a new CVE lands against something you already shipped, without needing to re-run CI to find out.

### SonarQube vs. Snyk — Overlapping, Not Redundant

Both offer a "SAST" capability (SonarQube's Vulnerability findings, Snyk Code), which raises the fair question of whether running both is redundant. In practice it isn't, for two reasons: their in-house code rule sets and detection engines differ (each catches some things the other misses), and SonarQube has essentially no meaningful SCA/container/IaC coverage — that's Snyk's real strength and the more common reason teams adopt it. A typical split: SonarQube gates code quality and in-house vulnerabilities/bugs on every PR; Snyk gates dependency, container, and IaC vulnerabilities on every PR and continues monitoring post-deploy via `snyk monitor`.

## 7. Handling False Positives — Suppression With Discipline

No static analyzer is perfect; a finding is sometimes genuinely a false positive, or a deliberate, justified exception. SonarQube supports marking a finding's resolution directly (in the UI or via API): **False Positive** (the tool is wrong), **Won't Fix** (real, but accepted risk), or **Fixed**. Snyk supports the equivalent via `.snyk` policy files, which record an explicit, reviewable ignore with an expiry date rather than a silent, permanent suppression.

```java
@SuppressWarnings("java:S1192") // "define a constant instead of duplicating this string literal"
private static final String STATUS_ACTIVE = "ACTIVE"; // suppressing a specific, understood rule ID
```

```yaml
# .snyk policy file — an explicit, time-boxed, reviewable exception, not a silent permanent suppression
ignore:
  SNYK-JAVA-EXAMPLE-1234567:
    - "*":
        reason: Vulnerable code path is unreachable in our usage — confirmed 2026-02-01, revisit at next major upgrade
        expires: 2026-08-01T00:00:00.000Z
```

The discipline that actually matters here: every suppression should reference the _specific_ rule/vulnerability ID being suppressed (never a blanket suppress-everything), state _why_, and — ideally, as Snyk's policy format enforces — expire, so a temporarily-accepted risk doesn't quietly become permanent by default.

## 8. Other Complementary Static Analysis Tools

| Tool                             | Focus                                                                                                                         |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| SpotBugs (successor to FindBugs) | Bytecode-level bug detection — catches some issues source-level tools miss (e.g., certain null-dereference patterns)          |
| PMD                              | Source-level code quality/style rules, highly configurable rule sets                                                          |
| Checkstyle                       | Enforces formatting/style conventions (import order, naming, brace placement)                                                 |
| Semgrep                          | Fast, pattern-based static analysis using a simple rule DSL — popular for writing custom, org-specific security rules quickly |

## 9. Best Practices

| Practice                                                                                           | Recommendation                                                                                                                                                           |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Gate the build on New Code, not the entire legacy codebase                                         | Makes SonarQube adoptable on an existing project — "clean as you code" approach.                                                                                         |
| Treat Security Hotspots as requiring human review, not auto-resolution                             | The same API can be safe or dangerous depending on context — that's exactly why SonarQube doesn't auto-clear them.                                                       |
| Run SCA continuously, not just at build time                                                       | A dependency can become vulnerable overnight with no code change on your side — use `snyk monitor` (or an equivalent) rather than relying solely on point-in-time scans. |
| Never suppress a finding without a specific ID, a stated reason, and — where supported — an expiry | An unexplained, permanent suppression is indistinguishable from silencing a real problem.                                                                                |
| Run SAST, SCA, and DAST as distinct, complementary practices                                       | Each catches a different class of problem; none of the three substitutes for the others.                                                                                 |
| Wire the Quality Gate into the actual CI/CD pipeline, not just a dashboard                         | A scan that reports findings without blocking a merge is easy to ignore.                                                                                                 |
| Prefer auto-remediation PRs for dependency upgrades where available                                | Turns "you have a CVE" into a reviewable, mergeable fix instead of open-ended manual work — a real Snyk advantage over point-in-time-only SCA tools.                     |
| Review flagged crypto/hashing usage in context, not by pattern alone                               | The fix for a password-hashing MD5 use (BCrypt/Argon2) is very different from the correct resolution for a non-security checksum use of the same algorithm.              |
