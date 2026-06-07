# `helm-chart-template` — Execution Plan

## How to use this plan

You are the build session for this repo. Read this whole file before doing anything else, then start executing immediately — no kickoff prompt needed.

**Working agreement:**

1. **Start without waiting.** Read this file end-to-end, then begin Phase 1 in the *Subagent playbook* below.
2. **Always ask the user about business decisions and business logic.** App image choice (default `hashicorp/http-echo`), README copy, sample value names, screenshot framing.
3. **Ask the user when you are genuinely blocked.**
4. **Do not ask the user about engineering details.** Template structure, label keys, internal helper names, schema details — make the call yourself.
5. **Use subagents aggressively.** Default to the playbook below.
6. **TaskCreate / TaskUpdate everything.**
7. **Pattern 3 only.** No live cluster demo. README ships screenshots of CI green + `helm install` output. Never commit secrets.
8. **Follow shared standards** (MIT, README, CI, topics, private until verified).
9. **All `Agent` tool calls must pass `model: "opus"`.**
10. **Off-limits forever:** SAP-internal Helm patterns (no portal/automaticd specifics), `~/.claude/`, RCA content. The chart must be generic.

## Subagent playbook (this repo)

Helm has lots of moving pieces (templates, schema, ESO, NetworkPolicy, CI). 4 subagents in research, 2 in review.

**Phase 1 — Research (parallel):**
- `Explore` (Opus): "Find current best practices for production-shaped Helm charts: probes, securityContext, PDB, HPA. Cite Bitnami / Grafana / Prometheus chart conventions. Return ≤300 words."
- `Explore` (Opus): "Find the modern External Secrets Operator template pattern (`SecretStore` vs `ClusterSecretStore`, `ExternalSecret` resource shape, gating via `.Values.externalSecrets.enabled`). Return a working template snippet."
- `Explore` (Opus): "Find the canonical `values.schema.json` example that fails fast on bad input, plus the Helm CLI behaviour for schema validation. Return a small working example covering required + enum + integer-min."
- `Explore` (Opus): "Find the canonical kind-based Helm smoke test in GitHub Actions: setup helm + setup kind + helm install + kubectl wait + helm test. Return a complete workflow file."

**Phase 2 — Design (single):**
- `Plan` (Opus): "Given the research and this PLAN.md, propose the exact `values.yaml` layout, template list, and CI matrix. Return as a checklist."

**Phase 3 — Build:** main session writes templates + values + schema + CI.

**Phase 4 — Review (parallel):**
- `code-reviewer` (Opus): "Review templates for: probe wiring, securityContext correctness, configmap-checksum rollout pattern, ExternalSecret gating, NetworkPolicy default-deny correctness, helm-docs sync. High effort."
- `tester` (Opus): "Add `ci/*-values.yaml` matrix files exercising minimal/full/ESO-enabled paths. Verify `helm template . | kubeconform -strict` passes for each."

**Phase 5 — Polish:** capture CI-passing screenshot or terminal cast of `helm install`, ask user before flipping public.

---

## Goal

A production-shaped Helm chart starter for a generic stateless web service.
The kind of chart someone could fork on Monday and deploy on Tuesday.

**Sells:** Helm, Kubernetes, GitOps, Production manifests, GitHub Actions, Chart Testing.

## Business decisions to ask the user about

- **App image to anchor the chart** — recommend `hashicorp/http-echo` (free, tiny, has an HTTP endpoint to probe). Alternatives: `nginxdemos/hello`, `traefik/whoami`.
- **Repo description copy** — keep current ("Production-shaped Helm chart starter…") or rephrase.
- **Whether to push the chart to GHCR as an OCI artifact on tag** — bonus signal for `helm-oci` skill but adds CI weight. Recommend skip for v1, do in v2.

## Scope (must-haves)

The chart deploys a **stateless HTTP service**. It includes:

1. `Deployment` with sane defaults: liveness + readiness probes, resource requests/limits, securityContext (non-root, read-only fs), revision history, `replicaCount`.
2. `Service` (ClusterIP).
3. `Ingress` (toggleable, with TLS hooks).
4. `HorizontalPodAutoscaler` (toggleable, CPU + memory thresholds).
5. `PodDisruptionBudget` (toggleable, default `maxUnavailable: 1`).
6. `ServiceAccount` (toggleable, with annotations support for IRSA / Workload Identity).
7. `ConfigMap` for non-secret env vars.
8. `Secret` (plain) — with `externalSecrets.enabled` toggle that swaps to an `ExternalSecret`. Defaults off.
9. `NetworkPolicy` (toggleable, default deny + allow-namespace).
10. `values.schema.json` covering every value.
11. **CI lint + smoke test:** `helm lint`, `helm template`, then `kind` + `helm install` + `kubectl wait` + curl probe.
12. README with values table (auto-generated via `helm-docs`).

## Out of scope

- No StatefulSet / DaemonSet variants.
- No Istio VirtualService / Gateway.
- No service mesh.
- No multi-environment overlays.
- No multi-container patterns / sidecars beyond the smoke test.
- No umbrella chart.

## Tech stack

- **Helm:** v3.13+
- **Kubernetes:** v1.28+ (CI uses `kind` with v1.28 node image)
- **CI:** GitHub Actions
- **Linting:** `helm lint`, `kubeconform`, `kube-linter`
- **Smoke test:** `kind` + `helm install` + `kubectl wait` + curl
- **Docs:** `helm-docs`

## File tree

```
helm-chart-template/
  README.md
  PLAN.md
  LICENSE
  .gitignore                      ← *.tgz, /bin
  Chart.yaml
  values.yaml
  values.schema.json
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
  ci/
    minimal-values.yaml
    full-values.yaml
    externalsecrets-values.yaml   ← lint-only
  tests/
    test-connection.yaml
  .github/workflows/
    lint.yml
    smoke.yml
  README.md.gotmpl                ← helm-docs template
  docs/screenshots/ci-passing.png
```

## Step-by-step build

### 1. Bootstrap

`helm create http-echo`, then strip boilerplate. Set `Chart.yaml` to v0.1.0.

### 2. `values.yaml` (top-level keys)

```yaml
image: { repository: hashicorp/http-echo, tag: "0.2.3", pullPolicy: IfNotPresent }
replicaCount: 2
service:    { type: ClusterIP, port: 80, targetPort: 5678 }
ingress:    { enabled: false, className: nginx, annotations: {}, hosts: [...], tls: [] }
resources:  { requests: { cpu: 50m, memory: 64Mi }, limits: { cpu: 200m, memory: 128Mi } }
autoscaling:        { enabled: false, minReplicas: 2, maxReplicas: 10, targetCPUUtilizationPercentage: 70, targetMemoryUtilizationPercentage: 80 }
podDisruptionBudget:{ enabled: true, maxUnavailable: 1 }
serviceAccount:     { create: true, annotations: {}, name: "" }
configMap:          { data: {} }
secret:             { data: {} }
externalSecrets:    { enabled: false, secretStoreRef: { kind: SecretStore, name: example-store }, remoteRefs: [] }
networkPolicy:      { enabled: false }
probes:             { liveness: {...}, readiness: {...} }
securityContext:    { runAsNonRoot: true, runAsUser: 65532, readOnlyRootFilesystem: true, allowPrivilegeEscalation: false, capabilities: { drop: ["ALL"] } }
podAnnotations: {}
nodeSelector: {}
tolerations: []
affinity: {}
args: ["-text=hello from helm-chart-template"]
```

### 3. Templates (key behaviour)

- `deployment.yaml`: probes wired from `.Values.probes`, securityContext at pod + container, image pull secrets, env from CM + Secret via `envFrom`, restart annotations on configmap checksum.
- `externalsecret.yaml`: `apiVersion: external-secrets.io/v1beta1`, only rendered when enabled, iterates `remoteRefs`.
- `networkpolicy.yaml`: default deny ingress + egress, allow same-namespace on configured port.
- `NOTES.txt`: helpful `kubectl port-forward` instructions.

### 4. `values.schema.json`

Hand-write or generate via `helm schema-gen`. Required fields flagged. Schema enables `helm install --validate` (implicit in 3.13+).

### 5. helm-docs

`README.md.gotmpl` with `{{ template "chart.valuesTable" . }}`. Run `helm-docs` locally before commits.

### 6. Helm test hook (`tests/test-connection.yaml`)

Pod with `"helm.sh/hook": test` running `wget -O- http://service`, exits 0 on 200.

### 7. CI workflows

**`lint.yml`:** `helm lint .` against `values.yaml` + each `ci/*-values.yaml`; `helm template .` piped into `kubeconform -strict`; `kube-linter lint .`.

**`smoke.yml`:** kind v1.28; `helm install http-echo . --wait --timeout 90s`; `kubectl wait --for=condition=available deploy/http-echo --timeout=60s`; `kubectl run curl ... -- curl -sS http://http-echo` and assert response; `helm test http-echo`.

### 8. README

1. Title — *helm-chart-template — Production-shaped Helm chart starter*
2. Demo — `docs/screenshots/ci-passing.png`
3. What it shows
4. Skills demonstrated — Helm, Kubernetes, GitOps, Production manifests, GitHub Actions, Chart Testing, kubeconform, kube-linter
5. Quick start: `helm install demo . --set ingress.enabled=true --set ingress.hosts[0].host=demo.local && helm test demo`
6. Values — auto-generated table
7. ExternalSecrets section — flip `externalSecrets.enabled=true` snippet
8. License — MIT

### 9. Polish + flip public

Topics: `helm`, `helm-chart`, `kubernetes`, `gitops`, `devops`, `chart-testing`, `github-actions`. Ask user before flipping.

## Verification

- [ ] `helm lint .` passes against all `ci/*-values.yaml`
- [ ] `helm template . | kubeconform -strict` passes
- [ ] `kube-linter lint .` no errors (warnings documented)
- [ ] CI smoke test passes (kind + install + helm test)
- [ ] `values.schema.json` rejects bad input (`--set replicaCount=foo`)
- [ ] README values table up to date (`helm-docs` no diff)
- [ ] No SAP-specific values, ingress hosts, registries
- [ ] Topics + description set

## Stretch (defer)

- ApplicationSet / ArgoCD example consumer
- Renovate config
- `chart-testing` (`ct lint --all`)
- ServiceMonitor for Prometheus
