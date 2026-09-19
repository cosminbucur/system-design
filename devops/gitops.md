GitOps is an operating model for deployment: Git holds the declarative, version-controlled description of what should be running, and an automated controller continuously reconciles the live system to match it — rather than deployments happening as one-off imperative commands (`kubectl apply`, a deploy script) run by a CI pipeline or a person. ArgoCD is the most widely used tool that implements this model for Kubernetes, but GitOps is the pattern; ArgoCD is one implementation of it.

## 1. The Core Idea: Pull, Not Push

Traditional CI/CD is a **push** model: the pipeline itself has credentials to the target environment and pushes changes into it directly at the end of a build.

```
Push model (traditional CI/CD):
  CI pipeline: build → test → push image → kubectl apply / deploy script (pipeline pushes into the cluster)
```

GitOps flips this into a **pull** model: CI still builds and tests, but its only output is a Git commit updating a manifest (a new image tag, a config change). A controller running *inside* the target environment watches that Git repo and pulls the change in on its own.

```
Pull model (GitOps):
  CI pipeline:   build → test → push image → update image tag in a Git manifest repo (CI's job ends here)
  GitOps agent:  watches the manifest repo → detects the change → applies it to the cluster (agent pulls it in)
```

This single inversion is what the rest of GitOps's guarantees are built on: the pipeline never holds direct write credentials to production, and "what should be running" (Git, reviewable via PR, with full history) is cleanly separated from "how it actually gets applied" (the controller, automatic and auditable).

## 2. The Four Principles

GitOps as a term (coined by Weaveworks) is usually defined by four properties a system must have to qualify, not just "we keep our YAML in Git":

| Principle | Meaning |
| --- | --- |
| Declarative | The desired state is described declaratively (what should exist), not as a sequence of imperative steps (how to get there) |
| Versioned and immutable | The desired state is stored in Git — every change is a commit, fully auditable, and trivially revertible with `git revert` |
| Pulled automatically | An agent in the target environment pulls approved changes, rather than external tooling pushing changes into it |
| Continuously reconciled | The agent doesn't just apply once — it continuously compares live state to declared state and corrects drift automatically |

The fourth principle is the one most often missed by teams who think they're "doing GitOps" just because their manifests live in a repo: without continuous reconciliation, a manual `kubectl edit` against production silently drifts the cluster away from Git and nobody notices until the next deploy overwrites it unpredictably. A real GitOps controller catches and reverts that drift immediately.

## 3. Why This Matters: What GitOps Buys You

- **Git history becomes deployment history**: `git log` on the manifest repo *is* the audit trail of every production change — who changed what, when, and (via the PR) why — without needing a separate deployment log or ticketing system to reconstruct it.
- **Rollback is `git revert`**: no separate rollback tooling or runbook — reverting a bad deploy is the same operation as reverting a bad code change, run through the same review process.
- **No standing deploy credentials outside the cluster**: the pipeline never holds a production kubeconfig or cloud credential capable of writing to the cluster directly — only the in-cluster controller does, shrinking the credential-leak blast radius considerably compared to a push-based pipeline.
- **Self-healing against drift**: a manual, undocumented change made directly against a live resource (during an incident, by mistake, or maliciously) gets detected and reverted automatically rather than silently persisting until someone notices the cluster no longer matches what's documented.

## 4. ArgoCD — A GitOps Implementation for Kubernetes

ArgoCD is the reference implementation most teams reach for: it runs as a controller inside the cluster, watches one or more Git repos of Kubernetes manifests, and continuously reconciles live cluster state to match them.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example-org/k8s-manifests.git
    targetRevision: main
    path: apps/order-service
  destination:
    server: https://kubernetes.default.svc
    namespace: order-service
  syncPolicy:
    automated:
      prune: true # remove resources deleted from Git
      selfHeal: true # revert manual cluster changes back to match Git
```

`selfHeal: true` is principle 4 (continuous reconciliation) made concrete: if someone manually `kubectl edit`s a live Deployment, ArgoCD detects the drift and reverts it back to match Git within seconds — the cluster genuinely cannot silently diverge from what's declared, rather than just being discouraged from doing so by convention.

## 5. ArgoCD Sync Strategies

| Setting | Behavior |
| --- | --- |
| Manual sync | Changes are detected but require a human to click "Sync" — useful for production wanting a deliberate gate before principle 3 (auto-pull) takes effect |
| Automated sync | Applies changes as soon as they're detected in Git — typical for lower environments (dev/staging) |
| `prune: true` | Deletes cluster resources whose manifests were removed from Git — without it, deleted YAML leaves orphaned resources running |
| `selfHeal: true` | Reverts manual out-of-band cluster changes to match Git |

## 6. Other GitOps Implementations, Briefly

ArgoCD isn't the only agent implementing this pattern — the model itself is tool-agnostic:

| Tool | Notes |
| --- | --- |
| ArgoCD | The most widely adopted; includes a UI for visualizing sync status/drift per application, multi-cluster support |
| Flux | CNCF's other major GitOps controller; more modular/composable pieces (source, kustomize, helm controllers) versus ArgoCD's more opinionated all-in-one Application model |
| Terraform (with a GitOps wrapper, e.g. Atlantis) | Applies the same pull-based, PR-triggered reconciliation idea to infrastructure provisioning rather than just Kubernetes manifests |

The choice between ArgoCD and Flux is mostly about ecosystem fit and preferred operating style (one opinionated tool vs. composable controllers) — both satisfy all four GitOps principles.

## 7. Progressive Delivery on Top of GitOps

ArgoCD (or Flux) is often paired with a rollout controller (Argo Rollouts is the common pairing) for deployment strategies beyond a basic rolling update — GitOps decides *what* should be running, progressive delivery decides *how carefully* to shift traffic toward it.

| Strategy | Behavior |
| --- | --- |
| Rolling update (Kubernetes default) | Gradually replaces old Pods with new ones |
| Blue-Green | Full new version deployed alongside the old; traffic switched all at once after validation |
| Canary | A small percentage of traffic routed to the new version first, gradually increased if healthy |

Canary deployments pair naturally with observability metrics — an automated canary analysis watches error rate/latency on the new version's small traffic slice and auto-rolls-back if it regresses, before the full fleet is affected. This rollback is still fundamentally a GitOps operation underneath: the rollout controller adjusts what's declared (or reverts to what was previously declared), and the same reconciliation loop applies it.

## 8. Best Practices

| Practice | Recommendation |
| --- | --- |
| Treat Git as the source of truth, not the cluster | Never `kubectl edit` a production resource directly — change the manifest in Git and let the controller apply it, or `selfHeal` will just revert your manual change anyway. |
| Separate the application code repo from the manifest repo | Keeps the CI pipeline (build/test) cleanly decoupled from the deploy trigger (a manifest commit) — a common and recommended GitOps repo layout. |
| Use manual sync (or a canary gate) for production | Automated sync is fine for dev/staging; production benefits from a deliberate approval step or automated canary analysis before full rollout. |
| Never commit real secrets to the manifest repo | Use a secrets manager or sealed-secrets approach — a Git-committed plaintext (or even base64) secret is a leaked secret the moment the repo is cloned. |
| Enable `prune` deliberately, understanding its effect | Without it, resources removed from Git are silently left running, orphaned and untracked. |
| Verify all four GitOps principles, not just "manifests are in Git" | Declarative, versioned, pulled, and continuously reconciled — missing continuous reconciliation specifically means drift goes undetected. |
| Pair canary rollouts with real observability signals | An automated canary analysis needs error rate/latency metrics to actually decide whether to proceed or roll back. |
