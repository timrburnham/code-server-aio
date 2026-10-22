# Kubernetes deployment

This folder is the Kubernetes equivalent of the repository's Docker Compose
files. It was translated for a local, single-node cluster (`k8s-dev`, arm64)
without Compose, without any files under `/opt`, and informs:

## Compose -> Kubernetes mapping

| Compose | Kubernetes here |
| --- | --- |
| `compose.yaml` service `code-server` (privileged, init, healthcheck, named volumes) | `code-server.yaml`: Deployment `code-server` + Services |
| `compose.yaml` service `cloudflared` | `cloudflared.yaml.disabled` (kept out of `kubectl apply -f k8s/` until enabled) |
| Named volume `code-server-home` | PVC code-server-home -> PV hostPath `/var/lib/code-server-aio/home` on node `k8s-dev` |
| Named volume `docker-data` | PVC code-server-docker-data -> PV hostPath `/var/lib/code-server-aio/docker-data` |
| `.env` variables | Secret `code-server-config`, created from `config.env` (from `config.env.example`) |
| `expose: 8080` | Service `code-server` on port 8080 — **same DNS name/port as Compose**, so a remotely-managed Cloudflare tunnel configured with origin `http://code-server:8080` keeps working without dashboard changes when cloudflared moves into the cluster |
| `restart: unless-stopped` | Deployment (restartPolicy Always); `strategy: Recreate` keeps the single privileged instance |
| `stop_grace_period: 150s` | `terminationGracePeriodSeconds: 150` |
| `healthcheck` (30s/15s, start_period 12m, retries 3) | exec probes running the adaptive script in `probe-configmap.yaml` |

Backup overlays are intentionally not converted yet (see bottom).

## Health check behavior

`k8s/probe-configmap.yaml` mirrors `build/health/container-healthcheck.py`
(supervisor services RUNNING, nested Docker up, code-server `/healthz`) with one
adaptation: until `CF_ACCESS_TEAM_DOMAIN` + `CF_ACCESS_AUD` contain real values,
the HAProxy response may be any HTTP status (i.e., the 503 setup page counts),
instead of strictly requiring the 403 Access-deny response. This keeps an
unconfigured deployment Ready so you can reach it locally; with placeholder
values from `config.env.example` it also stays Ready. Once real Access settings
are present, the check enforces the 403 exactly like the Compose version.

When your Access configuration is live you can additionally tighten this in
`code-server.yaml` (otherwise optional) — see Health check behavior above.

## Quick start (no tunnel)

The stack deploys and runs fully, but traffic to the access proxy is blocked
with its 503 page until Cloudflare Access is configured — identical to Compose
without values. Local browser access without the tunnel:
NodePort `30080` (http://<node>:30080). On the k8s-dev node 192.168.64.3:

```sh
kubectl apply -f k8s/
kubectl -n code-server get pods -w          # code-server becomes Ready (first boot installs Homebrew, watch logs)
kubectl -n code-server logs deploy/code-server -f
open http://192.168.64.3:30080              # 503 "not configured / deny" page proves the proxy path
```

Without Access configured, reach the editor itself via port-forward (works with
the 127.0.0.1-bind from supervisord.conf):

```sh
kubectl -n code-server port-forward deploy/code-server 8081:8081
# open http://127.0.0.1:8081  — direct code-server UI, bypasses Access (local only)
```

## Enable the Cloudflare tunnel and Access protection

1. Configure the remotely managed tunnel in the Cloudflare dashboard and keep
   its origin as `http://code-server:8080` (this Service exists with the same
   name and port, inside namespace code-server).
2. Fill the values:

   ```sh
   cp k8s/config.env.example k8s/config.env
   $EDITOR k8s/config.env    # tunnel token, team domain, AUD
   kubectl -n code-server create secret generic code-server-config \
     --from-env-file=k8s/config.env --dry-run=client -o yaml | kubectl apply -f -
   kubectl apply -f k8s/cloudflared.yaml.disabled
   kubectl -n code-server rollout status deploy/cloudflared
   ```

3. Optional but recommended once Access validation is live: restart
   `deploy/code-server` so probes re-run with the strict/403 expectation, and
   remove the NodePort service if you no longer want host access:
   `kubectl -n code-server delete svc code-server-nodeport`

## Operations (composer-equivalents)

| Compose command | Kubernetes |
| --- | --- |
| `docker compose logs -f` | `kubectl -n code-server logs -f deploy/code-server` |
| `docker compose exec --user cloud-user code-server bash` | `kubectl -n code-server exec deploy/code-server -- sudo -H -u cloud-user -i` (`kubectl exec` has no `--user` for container users) |
| `docker compose exec code-server supervisorctl status` | `kubectl -n code-server exec deploy/code-server -- supervisorctl status` |
| `docker compose down` (keeps volumes) | `kubectl -n code-server delete deployment/code-server` (PVCs/PVs persist) |
| `docker compose down -v` (deletes volumes) | Delete the namespace **and the PVs**, then wipe `sudo rm -rf /var/lib/code-server-aio` on the node (node is a VM/lima-like host) |
| `docker compose up -d --pull always` after changing `CODE_SERVER_IMAGE` | Update the `image:` tag in `code-server.yaml`, then `kubectl -n code-server rollout restart deploy/code-server` |

Image updates: bump the tag (e.g. `4.141.0` -> next release) in `code-server.yaml`
and `rollout restart`. `imagePullPolicy: IfNotPresent` mirrors pulling a pinned
tag; use the newest released tag from GHCR deliberately.

## Storage notes

This cluster has no StorageClass, so the two PVs are statically provisioned
hostPath directories on node `k8s-dev` (`/var/lib/code-server-aio/...`), bound
to PVCs by `volumeName`. `persistentVolumeReclaimPolicy: Retain` + per-namespace
deletion means deleting the namespace does **not** touch the data, equivalent to
Compose retaining volumes on `down`. To move nodes or wipe data, see the
"down -v" row above.

## Backups (not yet converted)

`compose.backup.yaml` / `compose.backup-local.yaml` rely on a host Docker socket
and label-based container stopping, which does not exist in a Kubernetes node
(containerd, no Docker daemon). A faithful conversion is a CronJob that mounts
the `code-server-home` volume read-only, scale-suspends `deploy/code-server`
around the archive (reproducing the stop-while-backup behavior), and uploads to
S3/R2 or writes a hostPath folder. Implement when needed; skip while unset.
