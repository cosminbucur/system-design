CI/CD is the automated pipeline that takes a code change from commit to running in production, replacing manual builds, manual test runs, and manual deployment steps with a repeatable, versioned process. The three terms in the acronym — Continuous Integration, Continuous Delivery, Continuous Deployment — describe increasing levels of automation, not three separate tools.

## 1. CI vs. CD vs. CD — Three Different Levels of Automation

| Term | What's automated | What still requires a human |
| --- | --- | --- |
| Continuous Integration (CI) | Every commit is automatically built and tested, merged frequently into a shared branch | Deciding when/whether to release; the release itself |
| Continuous Delivery | CI, plus the build is automatically packaged into a deployable artifact and verified as release-ready | A human clicks "deploy" to actually push it to production |
| Continuous Deployment | Continuous Delivery, plus every change that passes all automated checks deploys to production automatically | Nothing — the pipeline itself is the release process, end to end |

The distinction between the two CDs is the one people conflate most: **Continuous Delivery** means you *could* release any passing commit at any time with one click — **Continuous Deployment** means you *do*, automatically, with no click at all. Most teams land on Continuous Delivery in practice, keeping a deliberate human gate before production even when everything upstream is fully automated.

## 2. Continuous Integration: Merge Often, Catch Problems Early

CI's core idea predates most of today's tooling: integrate code changes into a shared branch frequently (multiple times a day, not once a week), with an automated build and test run on every single push, so integration conflicts and regressions surface within minutes of being introduced rather than accumulating for weeks.

```yaml
# GitHub Actions — a minimal CI pipeline triggered on every push and PR
name: CI
on: [push, pull_request]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - run: ./gradlew build test
      - run: ./gradlew jacocoTestReport # code coverage
```

The value compounds because of *frequency*: a merge conflict or a broken integration between two developers' changes is trivial to fix when it's one day's worth of divergence, and genuinely hard to untangle when it's three weeks' worth — this is the same reasoning behind trunk-based development (short-lived feature branches merged frequently) being favored over long-lived branches in a CI-driven workflow.

## 3. Pipeline Stages

A typical pipeline runs a sequence of stages, each gating the next — a failure at any stage stops the pipeline before wasting time (or risk) on later stages.

```
Commit → Build → Unit tests → Static analysis / security scan → Package artifact → Integration tests → Deploy to staging → (gate) → Deploy to production
```

| Stage | Purpose |
| --- | --- |
| Build | Compile the code, resolve dependencies — fails fast on anything that doesn't even compile |
| Unit tests | Fast, isolated tests (already covered in unit testing) — the cheapest possible signal, run first |
| Static analysis / security scan | Linting, SAST tools, dependency vulnerability scanning (e.g. `npm audit`, Snyk, Dependabot) — catches issues without running any code |
| Package artifact | Produce the versioned, deployable output — a container image, a JAR, a deployment package |
| Integration tests | Slower tests exercising real infrastructure (Testcontainers, test slices) — run after packaging since they validate the actual built artifact |
| Deploy to staging | Automated deployment to a non-production environment for final validation |
| Deploy to production | Gated (manual approval, or fully automatic under Continuous Deployment) |

Ordering stages from cheapest/fastest to most expensive/slowest is deliberate: a unit test failure should fail the pipeline in seconds, not after a 10-minute integration test suite has already run — this mirrors the same pyramid-shaped cost tradeoff behind the test pyramid itself.

## 4. Build Once, Promote the Same Artifact

A common and important pipeline principle: build the deployable artifact exactly once, then promote that identical artifact through each environment (dev → staging → production) rather than rebuilding it separately per environment.

```
BAD:  commit → build for dev → deploy to dev
      commit → build for staging → deploy to staging   (different build — what if the build itself is non-deterministic?)
      commit → build for prod → deploy to prod

GOOD: commit → build once → tag artifact (order-service:a3f9c21)
      → deploy order-service:a3f9c21 to dev
      → deploy order-service:a3f9c21 to staging (same artifact, config differs via environment variables)
      → deploy order-service:a3f9c21 to production (same artifact again)
```

This is what actually makes "it worked in staging" a meaningful statement — if production ran a *rebuild* rather than the exact tested artifact, a subtle build-environment difference (a dependency resolving to a different transitive version, a compiler flag) could silently produce a different binary than the one that passed every test. Configuration (database URLs, feature flags) should vary per environment; the artifact itself should not.

## 5. Where CI/CD Hands Off to Deployment and GitOps

CI's output is a versioned, tested artifact (a container image, typically) — what happens next is a separate concern, already covered in more depth elsewhere:

- **Deployment strategy** (how traffic shifts from old to new — rolling, blue-green, canary, A/B) determines the *mechanics* of the switch and its rollback characteristics.
- **GitOps** (an in-cluster controller like ArgoCD pulling a declared version from Git) is one way the "deploy" stage can actually be implemented — instead of the CI pipeline pushing directly into the cluster, its last step becomes committing the new artifact tag to a manifest repo, handing control to the reconciliation loop from there. This is the same push-vs-pull distinction already explored in GitOps: CI's job stops at "the desired state changed in Git," not at "the cluster was directly modified."

## 6. Quality Gates

A gate is an automated pass/fail check the pipeline enforces before allowing progression to the next stage — the entire point of CI/CD is replacing a human's manual judgment call with a consistent, automated one wherever that judgment can be codified.

| Gate | Typical threshold |
| --- | --- |
| Test pass rate | 100% — a single failing test blocks the pipeline, no exceptions |
| Code coverage | A minimum percentage (context-dependent); a coverage *drop* on a PR is often a stronger signal than an absolute number |
| Static analysis / linting | Zero new critical findings introduced by this change |
| Security scan | No new critical/high vulnerabilities in dependencies or the built image |
| Performance test (for critical paths) | p99 latency and error rate within an established threshold, per the same discipline covered in performance testing |

A gate that's routinely overridden ("just this once, force merge") is a signal the gate is either miscalibrated (too strict for real usage) or the team has stopped trusting it — either way, it needs fixing rather than habitual bypassing, since a gate nobody respects provides zero actual protection.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Merge frequently, keep branches short-lived | The value of CI compounds with integration frequency — a day's divergence is trivial to fix, weeks of divergence isn't. |
| Order pipeline stages cheapest-and-fastest first | Fail on a unit test in seconds, not after a slow integration or E2E suite has already run. |
| Build the artifact once, promote it unchanged across environments | Rebuilding per environment risks a subtle build difference between what was tested and what actually ships. |
| Treat gates as non-negotiable, or fix them | A quality gate that's routinely force-bypassed is providing the illusion of protection, not actual protection. |
| Separate CI's job (produce a tested artifact) from deployment's job (get it running safely) | Keeps the pipeline composable — the same artifact can deploy via a rolling update, canary, or GitOps reconciliation without CI needing to know which. |
| Default to Continuous Delivery with a deliberate release gate, not full Continuous Deployment | Most teams benefit from a human decision point before production even when everything upstream is fully automated — reserve full auto-deploy for systems with strong automated rollback/canary analysis already in place. |
