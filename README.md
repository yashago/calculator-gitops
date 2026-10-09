# calculator-gitops

Desired state of the calculator in Kubernetes. Argo CD watches this repo and makes the
cluster match it: deploying means committing here, rolling back means `git revert`.

The app and the pipeline that publishes images live in
[calculator-ci-test](https://github.com/yashago/calculator-ci-test).

| Path | What |
|---|---|
| `apps/` | ApplicationSet: one Argo CD Application per environment |
| `base/` | Rollout (canary), Services, ALB Ingress, AnalysisTemplate, ServiceMonitor, HPA, PDB |
| `overlays/dev/` | `calculator-dev` namespace. **CI updates the image tag** after every publish to main |
| `overlays/prod/` | `calculator-prod` namespace, 2–5 replicas. **Changed only by a reviewed PR** |

## How a release flows

1. A merge to `main` in calculator-ci-test builds, scans, signs and pushes `sha-<commit>` to ECR.
2. CI commits that tag to `overlays/dev` through a GitHub App that can write only to this repo.
3. Argo CD syncs `calculator-dev`. Argo Rollouts shifts ALB traffic 10% → 30% → 60% → 100%,
   two minutes per step, while Prometheus checks the canary's success rate (≥ 99%) and
   p95 latency (< 300 ms). A failing check aborts the rollout and sends all traffic back to stable.
4. To promote, open a PR here that copies the dev tag into `overlays/prod`. Merging it runs
   the same canary in `calculator-prod`.

## Useful commands

```sh
kubectl kustomize overlays/dev                       # render what Argo CD will apply
kubectl argo rollouts get rollout calculator -n calculator-dev -w
kubectl get ingress calculator -n calculator-dev     # ADDRESS is the ALB hostname
```
