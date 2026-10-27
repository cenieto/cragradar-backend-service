# cragradar-backend-service

Kubernetes manifests that deploy the CragRadar backend and the shared platform services. The same
manifests deploy to the local cluster (k3d on WSL) and to the cloud cluster (k3s on an Oracle Cloud
Always Free ARM VM). Only the overlays differ.

## Layout

```
platform/
  operators/              CloudNativePG operator (cluster-wide, installed once, server-side)
  postgres/               CNPG Cluster with PostGIS: databases `cragradar` and `keycloak`
  redis/                  Valkey: job queue, authorization cache
  keycloak/               Keycloak under /auth + realm `cragradar` (roles, clients)
base/
  api/  tiles/            backend Deployments + Services
  tile-cache/             nginx: auth_request -> api /api/authz/tiles, proxy_cache of tiles
  storage/                PVC image-data (originals and pyramids)
  ingress.yaml            /api -> api, /tiles -> tile-cache
overlays/
  local/                  k3d: PUBLIC_URL, cache size, development secrets and users
  cloud/                  (to do) domain, TLS, SOPS secrets, backups
```

Install once per cluster, then deploy:

```
kubectl apply --server-side -k platform/operators
kubectl apply -k overlays/<target>
```

Locally, `scripts/cluster.sh create` and `tilt up` in the umbrella repository do both.

Each overlay provides the ConfigMap `cragradar-config` (`PUBLIC_URL`, `TILE_CACHE_MAX_SIZE`) and
the Secrets `keycloak-admin`, `keycloak-db` and `keycloak-backend-client`.

## Environment differences (only in overlays)

| Setting | local | cloud |
| --- | --- | --- |
| Cluster | k3d (k3s in Docker) | k3s on one OCI A1 VM (arm64) |
| Images | imported into k3d | `ghcr.io/...`, multi-arch (amd64 + arm64) |
| Hostnames | `*.localhost` | real domain |
| TLS | none | cert-manager + Let's Encrypt |
| Storage | k3s local-path | k3s local-path on the block volume (S3 later) |
| nginx cache `max_size` | a few GB | sized to the free disk |
