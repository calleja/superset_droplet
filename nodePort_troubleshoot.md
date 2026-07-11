# NodePort Exposure Troubleshooting — DigitalOcean Droplet

Context document for continued diagnosis of the Apache Superset deployment on a DigitalOcean droplet. Scope is the **NodePort + nginx HTTPS** deployment only
(`custom_config/my-values.yaml`). The ClusterIP / port-forward variant
(`my-values.port-forward.yaml`) is intentionally **out of scope** here.

---

## 1. Target architecture

```
Browser (HTTPS :443)
  → nginx (droplet host, systemd)
  → 127.0.0.1:30088 (Kubernetes NodePort, loopback only)
  → Superset pod :8088
```

Key facts:
- Superset runs in **MicroK8s** (snap) in namespace `superset`.
- Service is `type: NodePort`, `port: 8088`, `nodePort.http: 30088`
  (`custom_config/my-values.yaml` service block).
- **nginx** (not Caddy) terminates TLS on 443. Caddy was abandoned because nginx already owns ports 80/443 on this droplet and cannot be stopped.
- nginx site: `/etc/nginx/sites-available/dash.greenehillfood.coop.conf`
  proxies to `http://127.0.0.1:30088`.
- Public domain: `dash.greenehillfood.coop`.
- Droplet IPs: public `67.207.80.236`, private `10.116.0.2`.

Relevant Superset config (`my-values.yaml` → `configOverrides.proxy_fix`):
```
ENABLE_PROXY_FIX = True
PREFERRED_URL_SCHEME = "https"
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
```

---

## 2. Issue A — Port 30088 reachable from the public internet

### Symptom
From the Mac (off-droplet), NodePort answered publicly:
```
curl -m 5 -I http://67.207.80.236:30088/health
HTTP/1.1 200 OK
Server: gunicorn
```
This is a security problem: the UI should only be reachable via nginx on 443.

### Root cause
Kubernetes NodePort binds to `0.0.0.0:30088` by default, and MicroK8s/kube-proxy programs iptables in a way that **bypasses ufw**. So even though no ufw rule allowed 30088, it was still publicly reachable.

### Ruled out
- ufw rules were reviewed (`sudo ufw status numbered`): **no rule allows 30088**.
  Deleting ufw rules does NOT fix this exposure. Do **not** rely on ufw here.
- Do NOT run `sudo ufw allow 30088` (opens it) or `sudo ufw deny 30088`
  (can break nginx → NodePort on loopback).

### Intended fix (bind NodePort to loopback only)
Restrict kube-proxy so NodePort listens on `127.0.0.1` only; nginx still reaches it, the public IP cannot:
```sh
grep nodeport-addresses /var/snap/microk8s/current/args/kube-proxy || \
  echo '--nodeport-addresses=127.0.0.1/32' | sudo tee -a /var/snap/microk8s/current/args/kube-proxy
sudo microk8s stop && sudo microk8s start
```

### Verification
```sh
sudo ss -tlnp | grep 30088
# WANT: 127.0.0.1:30088   (NOT 0.0.0.0:30088)

curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:30088/health   # on droplet → 200
curl -m 5 -I http://67.207.80.236:30088/health                            # from Mac → timeout/refused
```
Also confirm the DigitalOcean Cloud Firewall inbound rules are only 22/80/443
(no 30088, no NodePort range 30000–32767).

---

## 3. Issue B — `sudo ss -tlnp | grep 30088` returns nothing

### Symptom
```
sudo ss -tlnp | grep 30088        # empty
curl http://127.0.0.1:30088/health
curl: (7) Failed to connect to 127.0.0.1 port 30088 after 0 ms: Connection refused
```

### Interpretation
`Connection refused` (not timeout) = **nothing is listening** on 30088. For the
NodePort deployment this is a fault state. Likely causes:
1. MicroK8s stopped (`sudo microk8s start` needed).
2. Superset pods not Running.
3. The cluster was last deployed with the ClusterIP values instead of
   `my-values.yaml` (out of scope, but a known foot-gun — must redeploy NodePort).

### Diagnosis
```sh
sudo microk8s status
microk8s kubectl get svc  -n superset superset   # WANT TYPE=NodePort, PORT(S)=8088:30088/TCP
microk8s kubectl get pods -n superset            # WANT all Running
```

### Fix
```sh
sudo microk8s start
microk8s status --wait-ready

microk8s helm secrets upgrade --install superset superset/superset \
  --version 0.17.2 -n superset \
  -f custom_config/my-values.yaml \
  -f custom_config/environments/dev/secrets.yaml \
  --wait --timeout 15m
```
Then re-apply the loopback kube-proxy binding from Issue A and re-verify.

---

## 4. Issue C — HTTPS `/health` returns 301 (RESOLVED 2026-07-11)

### Symptom
```
curl -sS -o /dev/null -w "%{http_code}\n" https://dash.greenehillfood.coop/health
301
```
Expected 200.

### Interpretation
301 is **not** a TLS/nginx failure — TLS terminated and nginx routed the request; something issued a redirect. This is the current focus of diagnosis.

### Root cause (confirmed)
The nginx site `dash.greenehillfood.coop.conf` was mangled. The server block
that Certbot turned into the `listen 443 ssl` (HTTPS) block still had the
HTTP-redirect body `location / { return 301 https://$host$request_uri; }`, so
every HTTPS request was redirected to the identical HTTPS URL → infinite 301
loop (`Location: https://dash.greenehillfood.coop/health`, same URL). The real
`proxy_pass http://127.0.0.1:30088` server block was entirely commented out, so
nginx never proxied to Superset. Backend was healthy the whole time
(`curl http://127.0.0.1:30088/health` → 200), matching the decision-table row
"backend 200 / nginx 301 loop → fix nginx :443 server block".

### Fix applied
Rewrote the file into two clean blocks: port 80 = ACME challenge + HTTP→HTTPS
redirect; port 443 = TLS termination + `proxy_pass` to the loopback NodePort
with forwarded headers (see §5). Backup saved to
`~/dash.greenehillfood.coop.conf.bak.<timestamp>`. Applied via
`sudo tee` + `sudo nginx -t` + `sudo systemctl reload nginx`.

### Verification (passing)
```
https://dash.greenehillfood.coop/health   → 200
https://dash.greenehillfood.coop/login/   → 200
https://dash.greenehillfood.coop/         → 302 → /superset/welcome/ (relative, no scheme downgrade)
http://dash.greenehillfood.coop/health    → 301 → https, follows to 200
```

### Most likely causes (in order)
1. **Trailing-slash redirect** — `/health` → `/health/` (often benign).
2. **nginx server-block/vhost issue** — request handled by wrong/default block, or an HTTP→HTTPS `return 301` rule being hit unexpectedly.
3. **Superset proxy redirect** — `proxy_fix` / `X-Forwarded-Proto` mismatch causing scheme/host redirects.
4. **Redirect to `/login/`** — a protected-route redirect (less common for `/health`).

### Next diagnostic commands (not yet run / inconclusive)
```sh
# Where does it redirect, and who issued it?
curl -sS -I https://dash.greenehillfood.coop/health          # inspect Location: and Server:
curl -sS -I -L https://dash.greenehillfood.coop/health       # follow chain; want final 200
curl -v https://dash.greenehillfood.coop/health 2>&1 | grep -iE '< HTTP|location:|server:'

# Trailing slash test
curl -sS -o /dev/null -w "%{http_code}\n" https://dash.greenehillfood.coop/health/

# Split nginx vs Superset: hit NodePort directly on droplet
curl -sS -I http://127.0.0.1:30088/health
```

### Decision table
| Backend `127.0.0.1:30088` | Via nginx | Likely cause |
|---------------------------|-----------|--------------|
| 200 | 301 | nginx redirect (trailing slash / HTTP→HTTPS / wrong vhost) |
| 301 | 301 | Superset-side redirect (proxy headers / URL scheme) |
| refused | 301 or 502 | NodePort down — resolve Issue B first |

| `Location:` value | Action |
|-------------------|--------|
| `.../health/` | Accept, or use `/health/` in checks |
| same HTTPS URL (loop) | Fix nginx `:443` server block / SSL vhost for this domain |
| `/login/` | Check Superset config + nginx `proxy_pass` |
| `http://...` (downgrade) | Fix `X-Forwarded-Proto $scheme` and `PREFERRED_URL_SCHEME` |

### "Good enough" acceptance check
```sh
curl -sS -o /dev/null -w "%{http_code}\n" -L https://dash.greenehillfood.coop/health   # 200 after -L
# then browser: https://dash.greenehillfood.coop/login/
```

---

## 5. nginx reference (expected proxy config)

`/etc/nginx/sites-available/dash.greenehillfood.coop.conf` should include a
loopback proxy to the NodePort with forwarded headers:
```nginx
location / {
    proxy_pass http://127.0.0.1:30088;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout  300s;
    proxy_send_timeout  300s;
}
```
Validate/reload after edits:
```sh
sudo nginx -t
sudo systemctl reload nginx
```
Note: test the MAIN config with `sudo nginx -t` (not `nginx -t -c <site-fragment>`;
a site fragment fed as the main config errors with
`"server" directive is not allowed here`). TLS certs come from certbot
(`sudo certbot --nginx -d dash.greenehillfood.coop`); the cert files must exist or
`nginx -t` fails on the `ssl_certificate` lines.

---

## 6. Firewall summary (NodePort mode)

- **ufw** (droplet): allow 22, 80, 443, and 3306 from pod CIDR `10.1.0.0/16`.
  Do NOT add any 30088 rule.
- **DO Cloud Firewall**: inbound 22 (admin IPs), 80, 443 only. No 30088, no
  NodePort range.
- Public 30088 exposure is closed via kube-proxy `--nodeport-addresses=127.0.0.1/32`
  (Issue A), not via ufw.

---

## 7. Current status / next action

- Issue C (301 loop on `/health`): **RESOLVED 2026-07-11** — nginx site rewritten so
  the `:443` block proxies to `127.0.0.1:30088` (was stuck in a self-redirect loop
  with the proxy block commented out). HTTPS `/health` and `/login/` now return 200.
- Issue B (nothing listening on 30088): **not present** — backend healthy
  (`curl http://127.0.0.1:30088/health` → 200). Note: `ss -tlnp | grep 30088` shows
  nothing because MicroK8s kube-proxy runs in **iptables mode** (NodePort handled by
  NAT rules, no listening socket). "Connection refused" (not empty `ss`) is the real
  fault signal for Issue B; empty `ss` alone is expected here.
- Issue A (public 30088 exposure): **still open / unverified**. Could not confirm the
  kube-proxy `--nodeport-addresses=127.0.0.1/32` binding (needs
  `sudo` edit of `/var/snap/microk8s/current/args/kube-proxy` + `microk8s stop/start`,
  which requires config-change approval). From the droplet, verify whether
  `http://67.207.80.236:30088/health` is reachable off-host and close it via the
  kube-proxy arg (not ufw) if so.
