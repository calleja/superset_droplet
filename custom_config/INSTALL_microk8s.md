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
  --version 0.15.4 \
  -n superset \
  --create-namespace \
  -f custom_config/my-values.yaml \
  -f custom_config/environments/dev/secrets.yaml \
  --wait --timeout 15m 
```

`helm-secrets` will decrypt `secrets.yaml` to a temp file, merge it as a
second `-f` layer on top of `my-values.yaml`, and remove the temp file after.

### Alternative: install from local vendored chart

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

```sh
microk8s kubectl port-forward -n superset svc/superset 8088:8088
```

Open http://localhost:8088 in your browser.
Default bootstrap credentials (chart default, change after first login): **admin / admin**.

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
- Init job hangs on dockerize wait → MySQL is not reachable from the pod; recheck bind-address, ufw, and `SOURCE_MYSQL_HOST`.

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
