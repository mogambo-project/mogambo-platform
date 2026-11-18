# VPA — Vertical Pod Autoscaler (Fairwinds)

The VPA **controller** + its CRDs. The shared app chart renders VPA objects for the DB StatefulSets
(right-sizing recommendations), so this controller must be installed for them to work.

Managed by Argo CD via [`app/chart.yaml`](./app/chart.yaml): the `fairwinds-stable/vpa` chart
(v4.11.0) into the `vpa` namespace. Bump `targetRevision` + push to upgrade.
