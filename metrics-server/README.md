# metrics-server

Serves pod/node CPU + memory through the `metrics.k8s.io` API — the data source for **HPA**
autoscaling (and `kubectl top`). The shared app chart renders HPA objects, so this must be running.

Managed by Argo CD via [`app/chart.yaml`](./app/chart.yaml): the `metrics-server` Helm chart into
`kube-system`, with `--kubelet-insecure-tls` set in values (kind kubelets use self-signed certs — so
cert verification is skipped). Bump `targetRevision` in `app/chart.yaml` + push to upgrade.
