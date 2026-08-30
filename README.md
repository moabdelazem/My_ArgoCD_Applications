# My ArgoCD Application To Deploy On Any Cluster

## Backstage

Two Argo CD Applications, ordered by sync wave:

| Wave | App | What it is |
|------|-----|------------|
| `-1` | `local-path` | default StorageClass (already present) |
| `0`  | `backstage-db` | plain Postgres StatefulSet + secrets, from `manifests/backstage/` |
| `1`  | `backstage` | Backstage Helm chart (`backstage` namespace) |

Ingress is k3s' built-in **Traefik** (`ingressClassName: traefik`) on ports
80/443 — the chart's Service stays `ClusterIP`.

Postgres is a single-replica `StatefulSet` running the official
`postgres:16-alpine` image with a 2Gi `local-path` PVC — no operator, no CRDs.
The chart's bundled Postgres subchart stays off (`postgresql.enabled: false`);
Backstage reaches the database at `backstage-postgres.backstage.svc.cluster.local:5432`
and reads its credentials from the `backstage-postgres` secret through
`extraEnvVars`.

The catalog is seeded from `manifests/backstage/catalog-entities.yaml`, mounted
into the pod at `/app/catalog` — no egress to github.com at startup. Guest
sign-in maps to `user:default/guest` from that same file.

### Before syncing

1. **Postgres password** — replace `REPLACE_ME_...` in
   `manifests/backstage/postgres.yaml`. The Backstage signing key in
   `backend-secret.yaml` is already generated. Both sit in git in plain text;
   for real clusters use Sealed Secrets / External Secrets, keeping the same
   names and keys.
2. Nothing — the URL is `http://backstage.localhost`, which browsers resolve to
   `127.0.0.1` with no setup. If the cluster runs on another machine, add
   `<node-ip> backstage.localhost` to `/etc/hosts` on the machine you browse
   from. To use a different hostname, change `ingress.host` **and** the three
   `appConfig` URLs (`app.baseUrl`, `backend.baseUrl`, `backend.cors.origin`)
   together — they must match or the frontend fails CORS. For TLS, see the
   commented block next to `ingress.tls`.

Sign-in is the `guest` provider and the image is the community demo image
(`ghcr.io/backstage/backstage:latest`) — swap both for your own build and a real
auth provider before exposing this anywhere.

Changing the Postgres password after the first sync will not take effect: the
image only reads `POSTGRES_PASSWORD` when it initialises an empty data
directory. Delete the PVC (`data-backstage-postgres-0`) to start over.
