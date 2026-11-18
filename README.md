# mogambo-platform

The **platform / infra** repo for mogambo — the shared cluster tools (ingress gateway, metrics,
logs & traces, the cluster console, autoscaling controllers, network policy), owned separately from
the application-delivery config in [`mogambo-kubernetes`](https://github.com/mogambo-project/mogambo-kubernetes).

## Why this repo exists

mogambo's GitOps config is deliberately split in two, by **ownership and lifecycle**:

- **App delivery** — *what apps run and which versions* — lives in `mogambo-kubernetes`. It changes
  constantly (every deploy) and is the app developers' concern.
- **The platform** — *the shared tools every app relies on* — lives **here**. It changes rarely,
  carries cluster-wide blast radius, and is the platform/ops concern.

Keeping them apart buys:

- **Ownership / RBAC** — different reviewers and access per repo (this repo can require ops sign-off).
- **Blast radius & cadence** — a risky Prometheus upgrade never shares a PR or git history with a routine app deploy.
- **A clean app repo** — app teams don't see (or accidentally touch) cluster infra; they just ship to their own repo.
- **Reuse** — this repo can stand up the same tool stack on another cluster as-is.

This is the real-world **platform-team model**. Argo CD watches both repos and stitches them together
at runtime — the separation is organizational, not a runtime boundary.

> The one platform piece that is **not** here is the CNI (**Calico**). It's a day-0 prerequisite —
> nodes are `NotReady` until it runs, so it must exist *before* Argo CD — so it stays in
> `mogambo-kubernetes` alongside the cluster bring-up.

## Who uses it, and how

| Who | What they do |
|---|---|
| **Platform / ops** | Own this repo. Add or upgrade a tool by editing its `<tool>/` folder (bump an upstream chart version, tweak values) and pushing. Never `kubectl apply` by hand. |
| **Argo CD** | The only thing that *reads* this repo. The `mogambo-infra` app-of-apps discovers `<tool>/app/*.yaml`, creates one Application per tool, and reconciles them onto the cluster. |
| **App developers** | Do **not** touch this repo — they work in `mogambo-kubernetes` (`values-*/`). They *consume* the platform (scrape into Prometheus, route through the gateway, emit traces) without owning it. |

It is consumed as **plain Git** — Argo CD tracks the `master` branch. There is nothing to publish and
no version tag to reference: push to `master` = deploy. (The upstream chart **versions** are pinned
inside each tool's `app/*.yaml` — those are what you bump to upgrade a tool.)

## How it connects

Argo CD is installed and bootstrapped from `mogambo-kubernetes` (the `bootstrap/` umbrella chart).
That bootstrap plants an app-of-apps called **`mogambo-infra`** which points *here*:

```
mogambo-kubernetes  ──bootstrap──▶  Argo CD
        │                              │
        │  (app CD: values-*/)         ├─ mogambo-root  → app units + ArgoCD core
        │                              └─ mogambo-infra → THIS repo: <tool>/app/*.yaml
```

Nothing here is applied by hand — edit a file, push, Argo CD reconciles.

## Layout — one self-contained folder per tool

Each tool is a single folder: its **config** sits at the top, and an **`app/`** subfolder holds the
Argo CD `Application`(s) that install it (one object per file). The config and the thing that manages
it live together, and the `app/` convention makes discovery unambiguous (`*/app/*.yaml`).

```
<tool>/
  app/
    chart.yaml        # Application → the upstream Helm chart (+ values via $values)
    resources.yaml    # Application → the tool's own CRs/manifests in this folder
  <config files>      # values.yaml, routes, CRDs, dashboards, …
  README.md
```

## The tools

### `gateway/` — the edge (Gateway API via Envoy Gateway)
The cluster's HTTP entry point. The **controller** (`gateway-helm`) provisions an Envoy data plane;
the **CRs** (`GatewayClass`, `EnvoyProxy`, `Gateway`) define the listener and pin it to the kind node
so a browser on `:8090` reaches it. Every app's `HTTPRoute` (rendered by the shared app chart) attaches
to this single shared Gateway. → `app/controller.yaml` (chart, wave 0) + `app/resources.yaml` (CRs, wave 1).

### `monitoring/` — metrics (Prometheus + Grafana)
`kube-prometheus-stack`: Prometheus scrapes and stores metrics, Grafana visualizes them, plus the
prometheus-operator, node-exporter and kube-state-metrics — trimmed for a memory-limited cluster. Also
ships the Grafana route, alert rules, an Envoy `PodMonitor`, and dashboard ConfigMaps.
→ `app/chart.yaml` (chart + `values.yaml` via `$values`) + `app/resources.yaml` (route/alerts/podmonitor/dashboards).

### `observability/` — logs & traces (Loki · Alloy · Tempo · OTel Operator)
The other two observability pillars, surfacing in the *same* Grafana. **Loki** stores logs; **Alloy**
tails every pod's logs into Loki; **Tempo** stores traces; the **OTel Operator** injects language
auto-instrumentation agents via a pod annotation (no app code changes). Plus the `Instrumentation` CR
and the Grafana datasources. → `app/{loki,tempo,alloy,otel-operator}.yaml` (four charts, wave 0) +
`app/resources.yaml` (namespace + Instrumentation + datasources, wave 1).

### `headlamp/` — the cluster console
A lightweight web UI to browse / edit / exec / tail the cluster — the local stand-in for a cloud
console. Exposed through the shared gateway; logs in with a stable cluster-admin token.
→ `app/chart.yaml` (chart) + `app/resources.yaml` (admin RBAC + HTTPRoute).

### `network-policies/` — the cluster firewall
Zero-trust `NetworkPolicy` objects (a default-deny plus explicit per-tier allows), enforced by Calico.
Plain manifests — no chart (rendering these from a chart would be over-engineering). Keeping them in
Git means a loosened or deleted policy is auto-restored (security drift protection). → `app/app.yaml`.

### `metrics-server/` — the HPA data source
Serves pod/node CPU + memory through the `metrics.k8s.io` API. The app chart renders **HPA** objects,
which can't scale without this (and it powers `kubectl top`). Runs with `--kubelet-insecure-tls`
because kind's kubelets use self-signed certs. → `app/chart.yaml`.

### `vpa/` — right-sizing controller
The Fairwinds **VerticalPodAutoscaler** controller + its CRDs. The app chart renders **VPA** objects
for the DB `StatefulSet`s; they need this controller installed to produce right-sizing
recommendations. → `app/chart.yaml`.

## Onboarding a tool

1. Add a folder `<tool>/` with its config.
2. Add `<tool>/app/chart.yaml` (the Helm chart Application) and, if it has extra manifests,
   `<tool>/app/resources.yaml` (pointing at `<tool>/`, excluding `app/`).
3. Push. `mogambo-infra` picks it up automatically — no edit anywhere else.

Sync-wave the chart (`"0"`) ahead of its resources (`"1"`) so CRDs land before the CRs that use them.

## CI

[`Jenkinsfile`](./Jenkinsfile) validates every manifest with **kubeconform** on each push (core
Kubernetes kinds strictly; CRDs without public schemas are skipped, and Helm values files + raw
dashboard JSON are ignored) — so a malformed manifest is caught before Argo CD tries to apply it.
