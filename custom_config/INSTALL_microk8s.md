# Superset on MicroK8s — Install Guide

Target environment: DigitalOcean VM, 2 vCPU / 4 GiB, MicroK8s v1.35.0, helm v4.1.4 (snap).

---

## Prerequisites on the VM

```sh
# Confirm MicroK8s is running
microk8s status --wait-ready

# Enable required addons (if not already); deprecated, must now use hostpath-storage - apparently not suitable for production environments
microk8s enable dns storage

# Confirm helm
microk8s helm version

# Install the helm-secrets plugin (once)
microk8s helm plugin install https://github.com/jkroepke/helm-secrets

# Install SOPS (once)
# Download the binary from https://github.com/getsops/sops/releases
# or:
sudo apt-get install -y sops   # if available in your apt sources
# or via direct download:
curl -LO https://github.com/getsops/sops/releases/latest/download/sops-v3.9.4.linux.amd64
install -m 755 sops-v3.9.4.linux.amd64 /usr/local/bin/sops

# Install age (once)
sudo apt-get install -y age
# or via Go:
# go install filippo.io/age/cmd/...@latest
```

---

## 1. Generate an age key pair (once per VM)

```sh
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/key.txt
# Output: Public key: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Copy the public key line (starts with `age1...`) and paste it into
`custom_config/.sops.yaml`, replacing the `age1CHANGEME_...` placeholder.

The `.sops.yaml` rule targets `environments/.*/secrets\.yaml$` only, so the
`secrets.example.yaml` template is never accidentally encrypted. The entire
`secrets.yaml` is encrypted (no `encrypted_regex` partial selection) — SOPS
wraps every value into an `ENC[AES256_GCM,...]` block.

```sh
export SOPS_AGE_KEY_FILE=~/.config/sops/age/key.txt
# Add to ~/.bashrc or ~/.zshrc so it persists across sessions.
echo 'export SOPS_AGE_KEY_FILE=~/.config/sops/age/key.txt' >> ~/.bashrc
```

---

## 2. Prepare the secrets file

```sh
cd /path/to/droplet_dir/custom_config

cp environments/dev/secrets.example.yaml environments/dev/secrets.yaml
```

Edit `environments/dev/secrets.yaml` — fill in real values:

```yaml
extraSecretEnv:
  SUPERSET_SECRET_KEY: "paste_output_of_openssl_rand_base64_42"
  SOURCE_MYSQL_USER: "your_mysql_username"
  SOURCE_MYSQL_PASSWORD: "your_mysql_password"
```

`SOURCE_MYSQL_USER` is now a secret (K8s Secret, not ConfigMap) so it is never
visible in plaintext inside the cluster. `SOURCE_MYSQL_HOST`, `SOURCE_MYSQL_PORT`,
and `SOURCE_MYSQL_DATABASE` remain non-sensitive in `my-values.yaml`.

Generate a strong secret key:

```sh
openssl rand -base64 42
```

Encrypt the file in place:

```sh
# --in-place means overwrite the mentioned file
sops --encrypt --in-place environments/dev/secrets.yaml
```

The file now contains `ENC[AES256_GCM,...]` blocks and is safe to store.
`helm-secrets` decrypts it automatically at install time.

---

## 3. Prepare MySQL on the VM

MicroK8s pods run in an isolated CNI network (`10.1.0.0/16` by default).
`127.0.0.1` inside a pod resolves to the pod itself — not the VM.
MySQL must listen on an IP the pods can reach.

### a. Set bind-address

Edit `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```ini
bind-address = 0.0.0.0
# or bind to the VM's private IP only:
# bind-address = 10.114.0.2
```

Restart MySQL:

```sh
sudo systemctl restart mysql
```

### b. Allow the pod CIDR through the firewall

```sh
# Allow MicroK8s pod CIDR (default 10.1.0.0/16) to reach MySQL port 3306
sudo ufw allow from 10.1.0.0/16 to any port 3306
```

Confirm the pod CIDR for your install:

```sh
microk8s kubectl get node -o wide
# or
microk8s kubectl cluster-info dump | grep -i cidr
```

### c. Find the VM's private IP

```sh
ip addr show | grep 'inet ' | grep -v 127
# DigitalOcean droplets: private IP is typically 10.x.x.x
# private IP: 10.116.0.2
# public IPv4: 67.207.80.236
```

Update `my-values.yaml` — replace `CHANGEME_VM_PRIVATE_IP` with this IP:

```yaml
extraEnv:
  SOURCE_MYSQL_HOST: "10.114.0.2"   # your actual VM private IP
```

### d. Verify MySQL allows the connection user from pod IPs

```sql
-- On the VM:
mysql -u root -p
SELECT user, host FROM mysql.user WHERE user = 'your_superset_user';
-- If the host column is '127.0.0.1' or 'localhost', grant access from the pod CIDR:
GRANT ALL PRIVILEGES ON membership_ard.* TO 'lcalleja2'@'10.1.%' IDENTIFIED BY 'password';
GRANT SELECT ON membership_ard.* TO 'superset'@'localhost' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```

---

## 4. Add the Superset Helm repo

```sh
microk8s helm repo add superset https://apache.github.io/superset
microk8s helm repo update
```

---

## 5. Create the namespace

```sh
microk8s kubectl create namespace superset --dry-run=client -o yaml | \
  microk8s kubectl apply -f -
```

---

## 6. Install / upgrade

Run from the project root (parent of `custom_config/`):

```sh
microk8s helm secrets upgrade --install superset superset/superset \
  --version 0.17.2 \
  -n superset \
  --create-namespace \
  -f custom_config/my-values.yaml \
  -f custom_config/environments/dev/secrets.yaml \
  --wait --timeout 15m 
```

`helm-secrets` will decrypt `secrets.yaml` to a temp file, merge it as a
second `-f` layer on top of `my-values.yaml`, and remove the temp file after.

### Alternative: install from local vendored chart

> ⚠️ **Version mismatch:** the vendored chart in `superset-master/helm/superset`
> is `0.15.4` (app `5.0.0`). These values target chart `0.17.2` (app `6.1.0`, the
> latest published in the Helm repo — `0.19.0` is not in the repo index) and rely on
> 0.16.0+ behavior (no `initImage`; init containers run on the main image and wait via
> a bash `/dev/tcp` loop). To install from a local chart at `0.17.2`, first re-vendor
> it: `helm pull superset/superset --version 0.17.2 --untar`.
> Otherwise prefer the remote-repo install above.

If the Helm repo is unavailable or you want to use the vendored chart:

```sh
# Build subchart dependencies once
microk8s helm dependency build /path/to/superset-master/helm/superset

microk8s helm secrets upgrade --install superset \
  /path/to/superset-master/helm/superset \
  -n superset \
  --create-namespace \
  -f custom_config/my-values.yaml \
  -f custom_config/environments/dev/secrets.yaml \
  --wait --timeout 15m
```

### Non-snap helm / kubectl (plain binaries)

Replace `microk8s helm` with `helm` and `microk8s kubectl` with `kubectl`
after running `microk8s kubectl config view --raw > ~/.kube/config` to
export the cluster credentials.

---

## 7. Expose the UI

`my-values.yaml` configures `service.type: NodePort` with port `30088`.
Caddy on the VM terminates HTTPS on 443 and reverse-proxies to `127.0.0.1:30088`.
Port `30088` is intentionally **not** opened in ufw or the DigitalOcean Cloud Firewall.

### a. DNS prerequisite (one-time)

On your DNS provider, create an A record before configuring Caddy:

| Type | Name | Value |
|------|------|-------|
| A | `superset` (or the subdomain you choose) | droplet public IPv4 |

Verify propagation from your Mac:

```sh
dig +short superset.yourdomain.com
dig +short dash.greenehillfood.coop
# must return the droplet public IP before proceeding
```

### b. Helm upgrade (apply NodePort service change)

From the project root on the droplet (parent of `custom_config/`):

```sh
git pull origin master   # pull the updated my-values.yaml

microk8s helm secrets upgrade --install superset superset/superset \
  --version 0.17.2 \
  -n superset \
  -f custom_config/my-values.yaml \
  -f custom_config/environments/dev/secrets.yaml \
  --wait --timeout 15m
```

Confirm the NodePort is active:

```sh
microk8s kubectl get svc -n superset superset
# Expected: TYPE=NodePort, PORT(S)=8088:30088/TCP

curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:30088/health
# Expected: 200
```

### c. Install and configure Caddy

```sh
sudo apt update
sudo apt install -y caddy

# Back up default Caddyfile
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak.$(date +%Y%m%d)
```

Edit `/etc/caddy/Caddyfile` (replace hostname with your real domain):

```
superset.yourdomain.com {
    reverse_proxy 127.0.0.1:30088 {
        header_up Host {host}
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
        transport http {
            read_timeout 300s
            write_timeout 300s
        }
    }
}
```

Validate and reload:

```sh
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl enable caddy
sudo systemctl reload caddy
sudo systemctl status caddy
```

Caddy obtains and renews Let's Encrypt TLS certificates automatically.
Port 80 must be reachable from the internet for the ACME HTTP-01 challenge.

### d. Firewall (ufw + DigitalOcean Cloud Firewall)

**On the droplet (ufw):**

```sh
sudo ufw allow OpenSSH          # keep SSH open; restrict to admin IP if possible
sudo ufw allow 80/tcp           # required for Let's Encrypt HTTP-01 renewal
sudo ufw allow 443/tcp          # HTTPS public access

# Do NOT run: sudo ufw allow 30088/tcp  — Caddy reaches it over loopback only
# Do NOT run: sudo ufw deny 30088/tcp   — breaks Caddy → NodePort on loopback

sudo ufw enable    # if not already enabled
sudo ufw status numbered
```

**DigitalOcean Cloud Firewall** (Control Panel → Networking → Firewalls):

| Inbound | Ports | Sources |
|---------|-------|---------|
| SSH | 22 | Your admin IP(s) only |
| HTTP | 80 | 0.0.0.0/0, ::/0 |
| HTTPS | 443 | 0.0.0.0/0, ::/0 (or restrict to 4 user IPs) |
| — | 30088 | Do not add |

Optional: restrict HTTPS to known user IPs for additional defense. Let's Encrypt renewal
requires port 80 to remain open to `0.0.0.0/0`.

### e. Validate end-to-end

```sh
# From the droplet — NodePort reachable locally
curl -I http://127.0.0.1:30088/health

# From the droplet — HTTPS via Caddy
curl -sS -o /dev/null -w "%{http_code}\n" https://superset.yourdomain.com/health

# From your Mac — 30088 must NOT be reachable publicly (should fail/timeout)
curl -m 5 -I http://DROPLET_PUBLIC_IP:30088/health
```

From your Mac browser: `https://superset.yourdomain.com/login/`

Default bootstrap credentials (chart default — **change immediately after first login**): **admin / admin**

### f. Break-glass: SSH tunnel (no public exposure required)

The SSH tunnel remains available as an admin fallback at any time:

```sh
ssh -L 8088:127.0.0.1:8088 USER@DROPLET_PUBLIC_IP \
  'microk8s kubectl port-forward -n superset svc/superset 8088:8088'
# then open http://localhost:8088
```

### g. Rollback to ClusterIP

If you need to revert public access:

```sh
# In my-values.yaml, change service back to:
#   type: ClusterIP
#   port: 8088
# (remove the nodePort block)

microk8s helm secrets upgrade --install superset superset/superset \
  --version 0.17.2 -n superset \
  -f custom_config/my-values.yaml \
  -f custom_config/environments/dev/secrets.yaml \
  --wait --timeout 15m

sudo systemctl stop caddy
```

---

## 8. Validation

```sh
# Pod status
microk8s kubectl get pods -n superset

# Recent events (useful when pods are pending/crashing)
microk8s kubectl get events -n superset --sort-by='.lastTimestamp'

# Logs for a specific pod
microk8s kubectl logs -n superset -l app=superset --tail=50

# Resource usage (requires metrics-server addon)
microk8s enable metrics-server
microk8s kubectl top pod -n superset
```

Common failure modes:
- `ImagePullBackOff` on postgres/redis → confirm `bitnamilegacy/*` repository in `my-values.yaml`.
- `OOMKilled` on init job → raise `init.resources.limits.memory` to 2Gi temporarily.
- Init containers hang on the `/dev/tcp` wait loop → Postgres/Redis (or, for the source, MySQL) is not reachable from the pod; recheck bind-address, ufw, and `SOURCE_MYSQL_HOST`. (Chart 0.16.0+ no longer uses the `dockerize` binary; these values wait with a bash `/dev/tcp` loop on the main image.)

---

## 9. Uninstall / cleanup

```sh
microk8s helm uninstall superset -n superset
microk8s kubectl delete namespace superset
```

PVCs are NOT deleted automatically. Remove them manually if you want a clean slate:

```sh
microk8s kubectl delete pvc -n superset --all
```
