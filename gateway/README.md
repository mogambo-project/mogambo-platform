# Gateway API (Envoy Gateway) — frontend edge

The frontend is exposed with the **Gateway API**, implemented by **Envoy Gateway**. This replaces the
old NGINX **Ingress** (which lived in `../ingress/`, now removed). Same job — get external HTTP to the
`frontend` Service — but expressed as typed, role-separated resources instead of annotation-laden Ingress.

## The three resources (and who owns them)

| File | Kind | Role | Analogue in Ingress world |
|------|------|------|---------------------------|
| `gatewayclass.yaml` | `GatewayClass eg` | which controller (Envoy Gateway) | `IngressClass nginx` |
| `envoyproxy.yaml`   | `EnvoyProxy custom-proxy` | data-plane customisation (kind hostPort) | — (controller install flags) |
| `gateway.yaml`      | `Gateway mogambo` | the listener (port/proto/TLS) — platform role | the ingress-nginx *controller* |
| *(in the app chart)* | `HTTPRoute <app>-<ver>` | routing rules host/path→Service — app role; now rendered per version by the shared chart (`route:` in a unit's values) | the `Ingress` object |

Creating the **Gateway** makes Envoy Gateway provision a dedicated **Envoy data plane**
(a Deployment + Service named `envoy-mogambo-mogambo-*`) in the `envoy-gateway-system` namespace.

## How it's deployed (GitOps — no scripts)

Argo CD installs and manages both layers, as two Applications under [`../infra/`](../infra/):

| Application | What | Sync-wave |
|---|---|---|
| `gateway` | the Envoy Gateway **controller** (Helm chart `gateway-helm` v1.8.3, OCI) + the Gateway API CRDs | 0 |
| `gateway-resources` | the three CRs in this folder (`GatewayClass` + `EnvoyProxy` + `Gateway`) | 1 |

The wave order guarantees the controller + CRDs are up before the CRs that use them. To upgrade the
controller, bump `targetRevision` in [`../infra/gateway.yaml`](../infra/gateway.yaml)
and push — Argo CD rolls it. (The old `install.sh` has been removed.)

## Reach it — stable `http://mogambo.localtest.me:8090/`

`*.localtest.me` resolves to `127.0.0.1`, and the kind cluster maps host **`:8090` → control-plane
node `:80`** (see [../cluster/kind-config.yaml](../cluster/kind-config.yaml)). The `EnvoyProxy` in
`envoyproxy.yaml` pins the Envoy data plane onto the control-plane node and binds **`hostPort 80`**,
so that mapping reaches Envoy directly:

```
browser :8090 → node :80 (hostPort) → Envoy :10080 → HTTPRoute → frontend
```

No cloud-provider-kind, no port-forward, no ephemeral ports:

```bash
curl http://mogambo.localtest.me:8090/            # -> 200, Mogambo HTML
# browser: http://mogambo.localtest.me:8090/
```

Because the Envoy Service is `ClusterIP` (not `LoadBalancer`), a LB provider isn't needed — this is
why `envoyproxy.yaml` sets `envoyService.type: ClusterIP`.

## Verify / inspect

```bash
kubectl get gateway,httproute -n mogambo                     # Gateway PROGRAMMED=True
kubectl describe httproute frontend -n mogambo               # conditions: Accepted, ResolvedRefs
kubectl get pod -n envoy-gateway-system -o wide \
  -l gateway.envoyproxy.io/owning-gateway-name=mogambo       # should be on mogambo-control-plane
```

If `:8090` stops working: check the Envoy pod is `Running` **on the control-plane node** (only that
node has the `ingress-ready` label + the `8090→80` mapping). `PROGRAMMED=False` here would mean the
`EnvoyProxy`/GatewayClass `parametersRef` isn't resolving.

## Why Gateway API over Ingress (the point of the migration)

- **Typed fields, not annotations** — header/path matching, redirects, and **traffic weights** (canary)
  are validated spec fields, not `nginx.ingress.kubernetes.io/...` strings.
- **Role separation** — platform owns the `Gateway` (ports/TLS), app owns the `HTTPRoute`.
- **Portability** — the `Gateway` + `HTTPRoute` are implementation-agnostic; switching from Envoy
  Gateway to NGINX Gateway Fabric / Istio means changing only `GatewayClass.spec.controllerName`.
- **Future-proof** — `ingress-nginx` is retiring; Gateway API is the project's chosen successor.
