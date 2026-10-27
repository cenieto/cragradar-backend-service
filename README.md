# cragradar-backend-service

Kubernetes manifests that deploy the CragRadar backend and the shared platform services. The same
manifests deploy to the local cluster (k3d on WSL) and to the cloud cluster (k3s on an Oracle Cloud
Always Free ARM VM). Only the overlays differ.

## Planned layout

```
platform/                 shared services, installed once per cluster
  postgres/               CloudNativePG cluster with a PostGIS image
  redis/                  Valkey/Redis (job queue, authorization cache)
  keycloak/               Keycloak + realm export (clients, roles)
base/                     backend workloads
  api/                    Deployment + Service
  tiles/                  Deployment + Service
  worker/                 Deployment
  tile-cache/             nginx: auth_request -> api /authz, proxy_cache of tiles
  storage/                PVCs: originals and pyramids, nginx cache
  ingress.yaml
overlays/
  local/                  k3d: image tags, small resources, *.localhost hosts, no TLS
  cloud/                  OCI k3s: domain, TLS (cert-manager), cache size, backups
secrets/                  SOPS-encrypted secrets (*.enc.yaml)
```

Deploy with `kubectl apply -k overlays/<target>` (Kustomize, built into kubectl; no Helm).

## Environment differences (only in overlays)

| Setting | local | cloud |
| --- | --- | --- |
| Cluster | k3d (k3s in Docker) | k3s on one OCI A1 VM (arm64) |
| Images | imported into k3d | `ghcr.io/...`, multi-arch (amd64 + arm64) |
| Hostnames | `*.localhost` | real domain |
| TLS | none | cert-manager + Let's Encrypt |
| Storage | k3s local-path | k3s local-path on the block volume (S3 later) |
| nginx cache `max_size` | a few GB | sized to the free disk |
