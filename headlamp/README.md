# Headlamp — local web console

A free, open-source (CNCF) web UI to browse/edit/manage the cluster — the lightweight stand-in for a
cloud console. Served through the shared Envoy Gateway at **http://headlamp.localtest.me:8090/**.

## How it's deployed (GitOps — no scripts)

Argo CD manages Headlamp as two Applications under [`app/`](./app/):

| Application | What | Sync-wave |
|---|---|---|
| `headlamp` | the Helm chart (`headlamp` 0.44.0), namespace `headlamp` | 0 |
| `headlamp-resources` | a long-lived login token + HTTPRoute (this folder) | 1 |

The chart already ships a `headlamp` ServiceAccount bound to **cluster-admin** (its own
`headlamp-admin` ClusterRoleBinding), so we don't re-declare any RBAC — only a stable token Secret
for that SA (see [`admin-token.yaml`](./admin-token.yaml)). Bump the chart in
[`app/chart.yaml`](./app/chart.yaml) and push to upgrade. (The old `install.sh` has been removed.)

Prerequisite: the shared `mogambo` Gateway allow-lists this namespace to attach routes — its
`allowedRoutes.namespaces` `Selector` lists `mogambo` + `headlamp`
(see [../gateway/gateway.yaml](../gateway/gateway.yaml)). To expose another UI, add its
namespace to that list.

## Log in

Headlamp asks for a bearer token and then acts in the API **as that token** (so RBAC = what you can
edit). Get the stable cluster-admin token:

```bash
kubectl -n headlamp get secret headlamp-admin-token -o jsonpath='{.data.token}' | base64 -d; echo
```

Open **http://headlamp.localtest.me:8090/**, choose the cluster, paste the token.

## Notes

- The token is **cluster-admin** — fine for a local single-user cluster reachable only on localhost;
  don't expose the Gateway externally with this in place.
- Footprint is small (~150 MB), unlike Rancher — chosen deliberately for this memory-limited kind/WSL cluster.
- Same edit/logs/exec capabilities as `k9s`, but in a browser.
