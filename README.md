# Nextcloud Deployment on Kubernetes with Traefik — Project Report

## Project Goal

Deploy the following architecture on a single-node Kubernetes cluster based on K3s:

```text
                         Internet
                             │
                             │ HTTP / HTTPS
                             ▼
                    ┌──────────────────┐
                    │     Traefik      │
                    │ Ingress Controller│
                    └─────────┬────────┘
                              │
                  ┌───────────┴───────────┐
                  │                       │
                  ▼                       ▼
      Traefik Dashboard              Nextcloud
          api@internal                    │
                                           ▼
                                      PostgreSQL
```

## Technical Specs

| Item | Value |
|---|---|
| Nextcloud domain | `cloud.<your-domain>.tld` |
| Traefik Dashboard domain | `traefik.<your-domain>.tld` |
| Server IP | `<SERVER_IP>` |
| Kubernetes | K3s v1.36.3+k3s1 |
| OS | Ubuntu 24.04.4 LTS |
| Container Runtime | containerd |
| Traefik | v3.7.12 |
| StorageClass | local-path (WaitForFirstConsumer) |

---

## Final Architecture

```text
                                Internet
                                    │
                     ┌──────────────▼──────────────┐
                     │        <SERVER_IP>           │
                     └──────────────┬──────────────┘
                                    │
                         HTTP 80 / HTTPS 443
                                    ▼
                         ┌──────────────────┐
                         │     Traefik      │
                         │ Ingress Controller│
                         │  SSL Termination  │
                         └─────────┬────────┘
                                   │
              ┌────────────────────┴────────────────────┐
              ▼                                         ▼
 traefik.<your-domain>.tld                cloud.<your-domain>.tld
   Path: /dashboard/                        Nextcloud IngressRoute
              │                                        │
              ▼                                        ▼
        api@internal                         Nextcloud Service
              │                                        │
              ▼                                        ▼
      Traefik Dashboard                         Nextcloud Pod
                                                        │
                                                        │ PostgreSQL
                                                        ▼
                                                 PostgreSQL Service
                                                        │
                                                        ▼
                                                 PostgreSQL Pod
```

---

## Issues Encountered and Solutions

### 1. Node stuck in `NotReady`

At the start, the node status was:

```text
NAME     STATUS     ROLES
ubuntu   NotReady   control-plane
```

By checking `kubectl describe node` and confirming Memory/Disk/PID Pressure were healthy, the node returned to `Ready`.

---

### 2. Default K3s Traefik was not functional

- No Traefik pod existed in `kube-system`.
- The Service existed, but its `Endpoints` were empty (`<none>`).
- Checking `helmcharts` showed the HelmChart and manifest existed, but no actual Deployment had been created.

**Takeaway:** A Service existing does not mean the service is healthy — Pods and Endpoints must also be checked.

**Fix:** Install a standalone Traefik via Helm instead of relying on the half-active default K3s Traefik.

---

### 3. Helm error on reinstall

```text
cannot reuse a name that is still in use
```

**Cause:** A release named `traefik` already existed.

**Fix:** Use `helm upgrade` instead of `helm install` for subsequent installs.

---

### 4. Gateway API deprecation warning

After installing Traefik, a warning appeared that Gateway API CRDs would no longer ship with the chart in future versions.

**Fix:** Installed the Gateway API directly from the official release (v1.5.1).

---

### 5. PVC stuck in `Pending`

The PVCs for PostgreSQL and Nextcloud remained in `Pending` state after creation.

**Cause:** The StorageClass uses `WaitForFirstConsumer` mode, meaning the PV isn't provisioned until a consuming Pod is scheduled.

**Takeaway:** `Pending` PVCs in this mode are expected behavior, not necessarily an error.

---

### 6. Internal connectivity testing (DNS and network)

Using a temporary BusyBox pod, internal service connectivity was verified:

- `nslookup postgres` → `postgres.nextcloud.svc.cluster.local`
- `nc -zv postgres 5432` → `open`
- `nslookup nextcloud` → `nextcloud.nextcloud.svc.cluster.local`
- `wget -qO- http://nextcloud` → successfully returned Nextcloud's HTML page

These tests confirmed the CoreDNS → Service → Pod path was healthy.

---

### 7. Routing test via IngressRoute

A test `IngressRoute` was defined for `nextcloud.local`, and a `curl` request (with a manual `Host` header) returned HTTP `200`, confirming the Traefik → IngressRoute → Service → Pod path worked correctly.

---

### 8. `405` error when testing the Dashboard

```bash
curl -I -H "Host: dashboard.localhost" http://<SERVER_IP>
```

**Cause:** The `HEAD` request (triggered by the `-I` flag) wasn't supported by the Dashboard.

**Fix:** Switched to a `GET` request, which correctly returned a `302 Redirect`.

---

### 9. Expired certificate

An SSL check revealed the previous certificate had expired (`notAfter=Aug 25 2026`). A new **wildcard** certificate (`*.<your-domain>.tld`) from Let's Encrypt was obtained and put in place.

---

### 10. Wrong private key file path

Wrong path:

```text
/data/cert/private.key
```

Correct path:

```text
/data/cert/privkey.pem
```

The issue was found by listing the directory contents (`ls -lah`) and corrected.

---

### 11. Certificate files on disk don't auto-update the Kubernetes Secret

**Key point:** Replacing the certificate file on the server (`/data/cert/`) alone is not enough — Traefik reads the certificate from a **Kubernetes Secret**, not directly from the server's filesystem. After every certificate renewal, the corresponding Secret must be re-applied with `kubectl create secret tls ... --dry-run=client -o yaml | kubectl apply -f -`.

---

### 12. `404 page not found` error

After configuring SSL, a 404 error occurred, requiring a check across the full request chain:

```text
DNS → Traefik Service → EntryPoint → IngressRoute → Service → Pod
```

The issue was resolved by verifying the `web` (HTTP) and `websecure` (HTTPS) entry points and confirming ports 80/443 were correctly mapped.

---

### 13. Wrong path for accessing the Dashboard

The root path (`/`) is not a valid route for the Dashboard. The correct access URL is:

```text
https://traefik.<your-domain>.tld/dashboard/
```

---

### 14. Nextcloud incorrectly redirecting to HTTP

Since the connection between Traefik and Nextcloud inside the cluster was plain HTTP, Nextcloud assumed the entire connection was HTTP and redirected users to an `http://` URL.

**Fix:** Made Nextcloud reverse-proxy aware:

```bash
php occ config:system:set overwriteprotocol --value=https
php occ config:system:set overwritehost --value=cloud.<your-domain>.tld
php occ config:system:set overwrite.cli.url --value=https://cloud.<your-domain>.tld
```

---

## Key Lessons Learned

1. **A Service existing doesn't mean the app is healthy** — always check `pods`, `svc`, and `endpoints` together.
2. **A `Pending` PVC isn't always an error** — it's expected with a `WaitForFirstConsumer` StorageClass.
3. **Kubernetes internal DNS** follows the `service.namespace.svc.cluster.local` pattern.
4. **Traefik is more than a simple reverse proxy** — it handles routing, TLS termination, service discovery, CRD integration, and the dashboard.
5. **The certificate on disk and the certificate inside Kubernetes are different things** — Traefik reads certs from a Secret, not from the server's filesystem.
6. **An app behind a reverse proxy must be aware of the proxy** — settings like `overwriteprotocol` and `trusted_proxies` exist for exactly this reason.

---

## Current Project Status

| Component | Status |
|---|---|
| Kubernetes Node | ✅ Ready |
| K3s | ✅ Running |
| CoreDNS | ✅ |
| StorageClass | ✅ local-path |
| PostgreSQL PVC | ✅ Bound |
| PostgreSQL Pod | ✅ Running |
| PostgreSQL Service | ✅ |
| Internal DNS (PostgreSQL) | ✅ |
| Nextcloud PVC | ✅ Bound |
| Nextcloud Pod | ✅ Running |
| Nextcloud Service | ✅ |
| Internal DNS (Nextcloud) | ✅ |
| Traefik | ✅ Running |
| Traefik LoadBalancer | ✅ |
| Traefik CRDs | ✅ |
| Nextcloud IngressRoute | ✅ |
| Internal Traefik routing | ✅ |
| Domain DNS | ✅ |
| TLS Secret | ✅ |
| Wildcard Certificate | ✅ |
| Traefik SSL | ✅ |
| Traefik Dashboard | ✅ |
| Traefik Dashboard over HTTPS | ✅ |
| Nextcloud over HTTPS | ✅ |
| HTTPS Redirect Awareness | ✅ |
| trusted_proxies | ⏳ Next step |

---

## Roadmap (Next Steps)

1. **Complete `trusted_proxies` setup** — define the Pod network (`10.42.0.0/24`) or Traefik's exact address as a trusted proxy for Nextcloud.
2. **HTTP → HTTPS redirect** — automatically redirect all HTTP requests to HTTPS at the Traefik level.
3. **Security headers** — add headers such as `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`, and `Referrer-Policy` via Traefik Middleware.
4. **Dashboard authentication** — add BasicAuth or ForwardAuth to prevent public access to the Dashboard.
5. **Health checks** — configure health checks for the services.
6. **Resource limits** — set `requests`/`limits` for CPU and memory on Nextcloud, PostgreSQL, and Traefik.
7. **PostgreSQL backups** — implement scheduled backups using a Kubernetes CronJob.
8. **Nextcloud data backups** — separately back up the Nextcloud data PVC and the PostgreSQL database.
9. **Monitoring** — add monitoring for service and cluster resource health.
