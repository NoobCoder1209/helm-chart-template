# `helm-chart-template` — Execution Plan

> Self-contained build plan. Inherits shared standards from the master plan.

## Goal

A production-shaped Helm chart starter for a generic web service. The kind
of chart someone could fork on Monday and deploy on Tuesday. Demonstrates
real K8s + Helm depth without exposing any SAP-internal patterns.

**Sells:** Helm, Kubernetes, GitOps, Production-ready Manifests, GitHub Actions,
Chart Testing.

## Scope (must-haves)

The chart deploys a **stateless HTTP service** (anchor it on the public
[`hashicorp/http-echo`](https://hub.docker.com/r/hashicorp/http-echo) image so
the smoke test has something real to hit). It includes:

1. `Deployment` with sane defaults: liveness + readiness probes, resource
   requests/limits, securityContext (non-root, read-only fs), revision history,
   minReplicas via `replicaCount`.
2. `Service` (ClusterIP).
3. `Ingress` (toggleable, with TLS hooks).
4. `HorizontalPodAutoscaler` (toggleable, CPU + memory thresholds).
5. `PodDisruptionBudget` (toggleable, default `maxUnavailable: 1`).
6. `ServiceAccount` (toggleable, with annotations support for IRSA / Workload Identity).
7. `ConfigMap` for non-secret env vars.
8. `Secret` (plain) — but with an `externalSecrets.enabled` toggle that swaps
   to an `ExternalSecret` resource using a `SecretStore` ref. Defaults off so
   `helm template` works without ESO installed.
9. `NetworkPolicy` (toggleable, default deny + allow-namespace).
10. `values.schema.json` — JSON Schema describing every value, derived from
    the `values.yaml` defaults. Lets `helm install` fail fast on bad input.
11. **CI lint + smoke test**: `helm lint`, `helm template`, then `kind` cluster
    spin-up + `helm install` + `kubectl wait` + curl probe.
12. README with values table (auto-generated via `helm-docs`).

## Out of scope

- No StatefulSet / DaemonSet variants.
- No Istio VirtualService / Gateway.
- No service mesh.
- No multi-environment overlays (chart is generic; env-specific values live
  outside the chart in real consumers).
- No multi-container patterns / sidecars beyond what's needed for the smoke test.
- No second deliverable (e.g. an "umbrella chart" — out).

## Tech stack

- **Helm:** v3.13+
- **Kubernetes:** v1.28+ (CI uses `kind` with v1.28 node image)
- **CI:** GitHub Actions
- **Linting:** `helm lint`, `kubeconform`, `kube-linter`
- **Smoke test:** `kind` + `helm install` + `kubectl wait` + curl
- **Docs:** `helm-docs` for the values table in README
- **Pre-commit:** optional `.pre-commit-config.yaml` calling helm-docs

## File tree

```
helm-chart-template/
  README.md                       ← chart README (helm-docs auto-section)
  PLAN.md
  LICENSE
  .gitignore                      ← *.tgz, /bin
  Chart.yaml
  values.yaml                     ← defaults
  values.schema.json              ← derived schema
  templates/
    _helpers.tpl
    deployment.yaml
    service.yaml
    ingress.yaml
    hpa.yaml
    pdb.yaml
    serviceaccount.yaml
    configmap.yaml
    secret.yaml
    externalsecret.yaml           ← gated on .Values.externalSecrets.enabled
    networkpolicy.yaml
    NOTES.txt
  ci/                             ← extra values files used in CI matrix
    minimal-values.yaml
    full-values.yaml
    externalsecrets-values.yaml   ← lint-only (no ESO in CI cluster)
  tests/                          ← Helm test hooks (real, not /test)
    test-connection.yaml
  .github/
    workflows/
      lint.yml                    ← helm lint + kubeconform + kube-linter
      smoke.yml                   ← kind + install + curl
  README.md.gotmpl                ← helm-docs template
  docs/
    screenshots/
      ci-passing.png
```

## Step-by-step build

### 1. Bootstrap

```bash
helm create http-echo
# rename or move into the repo root, keep Chart.yaml + templates + values.yaml
```

Then strip the boilerplate down — `helm create` ships extras (tests folder,
`hpa.yaml` already, etc.) that need editing for a "production" feel.

`Chart.yaml`:
```yaml
apiVersion: v2
name: http-echo
description: Production-shaped Helm chart for a stateless HTTP service.
type: application
version: 0.1.0
appVersion: "0.2.3"
maintainers:
  - name: Aleksandar Chapkanov
    url: https://github.com/NoobCoder1209
```

### 2. `values.yaml` (top-level keys)

```yaml
image:
  repository: hashicorp/http-echo
  tag: "0.2.3"
  pullPolicy: IfNotPresent

replicaCount: 2

service:
  type: ClusterIP
  port: 80
  targetPort: 5678

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: chart-example.local
      paths: [{ path: /, pathType: Prefix }]
  tls: []

resources:
  requests: { cpu: 50m, memory: 64Mi }
  limits:   { cpu: 200m, memory: 128Mi }

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

podDisruptionBudget:
  enabled: true
  maxUnavailable: 1

serviceAccount:
  create: true
  annotations: {}
  name: ""

configMap:
  data: {}

secret:
  data: {}

externalSecrets:
  enabled: false
  secretStoreRef:
    kind: SecretStore
    name: example-store
  remoteRefs: []   # list of { localKey, remoteKey, property? }

networkPolicy:
  enabled: false

probes:
  liveness:
    httpGet: { path: /, port: http }
    initialDelaySeconds: 5
    periodSeconds: 10
  readiness:
    httpGet: { path: /, port: http }
    initialDelaySeconds: 2
    periodSeconds: 5

securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities: { drop: ["ALL"] }

podAnnotations: {}
nodeSelector: {}
tolerations: []
affinity: {}
```

Add a starter command-line for `http-echo`:
```yaml
args: ["-text=hello from helm-chart-template"]
```

### 3. Templates

Each template pulls values + uses the standard `_helpers.tpl` for labels,
selector labels, full name, and chart-version annotation. Key things
to get right:

- **`deployment.yaml`** — probes wired from `.Values.probes`, securityContext
  applied at pod and container level, image pull secrets templated, env from
  ConfigMap + Secret via `envFrom`, restart annotations on configmap checksum
  (so config change → rollout).
- **`externalsecret.yaml`** — `apiVersion: external-secrets.io/v1beta1`, only
  rendered when `.Values.externalSecrets.enabled`. Iterates `remoteRefs`.
- **`networkpolicy.yaml`** — default deny ingress + egress, allow same-namespace
  on the configured port.
- **`NOTES.txt`** — helpful `kubectl port-forward` instructions.

### 4. `values.schema.json`

Hand-write or generate from `values.yaml` using
[`helm schema`](https://github.com/karuppiah7890/helm-schema-gen) tool. Make
sure required fields are flagged. Schema enables `helm install --validate`
(implicit in 3.13+).

### 5. `helm-docs` template

`README.md.gotmpl`:
```
# {{ template "chart.header" . }}
{{ template "chart.description" . }}

## Values

{{ template "chart.valuesTable" . }}
```

Add `helm-docs` step locally (`brew install norwoodj/tap/helm-docs` then
`helm-docs`) so the README values table stays in sync.

### 6. Helm test hooks (`tests/test-connection.yaml`)

A `Pod` with annotation `"helm.sh/hook": test` that runs `wget -O- http://service`
and exits 0 on 200. This is the canonical Helm test pattern; `helm test` will
run it.

### 7. CI workflows

**`lint.yml`** triggers on push + PR:
- Setup Helm
- `helm lint .` against `values.yaml` and each `ci/*-values.yaml`
- `helm template .` and pipe into `kubeconform -strict`
- `kube-linter lint .`

**`smoke.yml`** triggers on push + PR:
- Setup Helm
- Spin up `kind` (Kubernetes 1.28)
- `helm install http-echo . --wait --timeout 90s`
- `kubectl wait --for=condition=available deploy/http-echo --timeout=60s`
- `kubectl run curl --image=curlimages/curl --rm -it --restart=Never -- curl -sS http://http-echo`
- Assert response includes `"hello from helm-chart-template"`
- `helm test http-echo`

### 8. README

1. **Title** — *helm-chart-template — Production-shaped Helm chart starter*
2. **Demo** — `docs/screenshots/ci-passing.png` (or a small terminal cast of `helm install` + curl)
3. **What it shows**:
   - Real probes, resource limits, securityContext, PDB, HPA toggles
   - ExternalSecrets-ready secret pattern (SecretStore-scoped)
   - `values.schema.json` for fail-fast input validation
   - kind-based smoke test in CI
4. **Skills demonstrated** — Helm, Kubernetes, GitOps, Production manifests, GitHub Actions, Chart Testing, kubeconform, kube-linter
5. **Quick start**:
   ```bash
   helm install demo . --set ingress.enabled=true --set ingress.hosts[0].host=demo.local
   helm test demo
   ```
6. **Values** — auto-generated table (helm-docs)
7. **External secrets section** — short snippet showing how to flip `externalSecrets.enabled=true` and provide `remoteRefs`
8. **License** — MIT

### 9. Polish + flip public

Topics: `helm`, `helm-chart`, `kubernetes`, `gitops`, `devops`, `chart-testing`,
`github-actions`. Flip public.

## Verification

- [ ] `helm lint .` passes against all `ci/*-values.yaml`
- [ ] `helm template . | kubeconform -strict` passes
- [ ] `kube-linter lint .` produces no errors (warnings acceptable; document any waivers)
- [ ] CI smoke test passes (kind cluster spins up, deploy responds, helm test passes)
- [ ] `values.schema.json` rejects an obviously bad input (e.g. `--set replicaCount=foo`)
- [ ] README values table is up to date (`helm-docs` produces no diff)
- [ ] No SAP-specific values, ingress hosts, or registry references
- [ ] Topics + description set
- [ ] Repo is consumable as a Helm OCI registry source (optional bonus — push to GHCR with `helm push`; document but do not require)

## Stretch (defer)

- ApplicationSet / ArgoCD example consumer chart
- Renovate config for image tag updates
- Fuzz `values.yaml` keys with `chart-testing` (`ct lint --all`)
- ServiceMonitor for Prometheus

v2 — out of v1 scope.
