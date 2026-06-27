# Superset on MicroK8s — Configuration Reference

This directory holds the Helm values and supporting configuration to deploy Apache Superset 5.0.0 on a MicroK8s cluster running on a DigitalOcean 2 vCPU / 4 GiB VM.

---

## File structure

```
custom_config/
├── my-values.yaml                        # Primary Helm overrides (non-sensitive)
├── .sops.yaml                            # SOPS encryption rules (age)
├── INSTALL_microk8s.md                   # End-to-end install steps
├── environments/
│   └── dev/
│       ├── secrets.example.yaml          # Secrets template (copy → secrets.yaml → encrypt)
│       └── secrets.yaml                  # SOPS-encrypted secrets (NOT committed unencrypted)
└── README.md                             # This file
```

---

## Key decisions

### 1. MicroK8s instead of Minikube

The previous iteration (`my-values_droplet.yaml`) was built for Minikube. This iteration targets MicroK8s, which ships as a snap on Ubuntu and does not require a separate Docker runtime. The behavioral difference that matters most for this config is **host MySQL reachability** — see section below.

### 2. Helm chart version pinned at 0.15.4

The vendored chart in `superset-master/helm/superset/Chart.yaml` is `0.15.4`. Install and upgrade commands use `--version 0.15.4` to keep reproducible installs. The app image is pinned at `5.0.0`.

### 3. Bitnami image repository: bitnamilegacy

The Superset chart depends on Bitnami's PostgreSQL and Redis subcharts. Versioned Bitnami image tags are no longer available under `docker.io/bitnami/*` on Docker Hub — they have moved to `docker.io/bitnamilegacy/*`. The previous minikube install failed with `manifest unknown` / `ImagePullBackOff` because of this.

Both images are explicitly overridden in `my-values.yaml`:

```yaml
postgresql:
  image:
    registry: docker.io
    repository: bitnamilegacy/postgresql
    tag: "14.17.0-debian-12-r3"

redis:
  image:
    registry: docker.io
    repository: bitnamilegacy/redis
    tag: "7.0.10-debian-11-r4"
```

### 4. PostgreSQL and Redis are metadata/cache only

PostgreSQL (chart-managed) stores Superset's own metadata: dashboards, charts, users, roles, and query history. Redis (chart-managed) is the Celery broker and results cache. Neither holds your business data.

The source business data lives in the external MySQL database on the VM.

### 5. Python driver installation at bootstrap

The Superset 5.0.0 Docker image does not ship all required Python drivers in its venv. Three are installed at pod startup via `bootstrapScript`:

| Package | Purpose |
|---|---|
| `psycopg2-binary` | PostgreSQL driver for the metadata DB (SQLAlchemy URI `postgresql+psycopg2://`) |
| `mysql-connector-python` | MySQL driver for source DB connections (`mysql+mysqlconnector://`) |
| `pymysql` | Shim for Superset engine spec paths that still `import MySQLdb` |

`pymysql` is registered as `MySQLdb` via a `configOverrides` snippet.

### 6. Secrets are managed with SOPS + age + helm-secrets

Sensitive values (`SUPERSET_SECRET_KEY`, `SOURCE_MYSQL_PASSWORD`) are not stored in `my-values.yaml`. They live in `environments/dev/secrets.yaml`, which is encrypted with [SOPS](https://github.com/getsops/sops) using an [age](https://github.com/FiloSottile/age) key stored only on the VM.

The [helm-secrets](https://github.com/jkroepke/helm-secrets) plugin decrypts the file transparently at install time and passes it as a second `-f` layer.

### 7. Resource sizing: "Suggested (B)" envelope

On a 2 vCPU / 4 GiB node with PostgreSQL and Redis chart-managed, available headroom is tight. The current resource limits are:

| Component | CPU limit | Memory limit |
|---|---|---|
| superset web | 1 | 1280 Mi |
| superset worker | 500m | 768 Mi |
| init job | 1 | 1536 Mi |
| postgresql | 500m | 768 Mi |
| redis | 200m | 256 Mi |

If the init job is `OOMKilled`, raise `init.resources.limits.memory` to `2Gi` for the first run, then lower it back.

### 8. Optional components disabled

`supersetCeleryBeat`, `supersetCeleryFlower`, and `supersetWebsockets` are disabled to conserve memory on the small node.

---

## Connecting pods to the host MySQL

### Why `host.minikube.internal` no longer works

The previous config used `host.minikube.internal` as the MySQL hostname. This is a Minikube-specific DNS entry that resolves to the host machine. MicroK8s does not provide an equivalent magic hostname — pods in a MicroK8s CNI network can only reach the VM's MySQL at a routable IP address.

### What you need to research on your VM

The following values must be discovered on the actual DigitalOcean VM before the install will succeed:

| What | Where to find it | Used in |
|---|---|---|
| VM private IP (10.x.x.x) | `ip addr show` or DigitalOcean Control Panel → Networking → Private IP | `extraEnv.SOURCE_MYSQL_HOST` in `my-values.yaml` |
| MicroK8s pod CIDR | `microk8s kubectl cluster-info dump \| grep -i cidr` | ufw allow rule and MySQL GRANT statement |
| MySQL bind-address | `/etc/mysql/mysql.conf.d/mysqld.cnf` | Must be `0.0.0.0` or the private IP, not `127.0.0.1` |
| MySQL user with remote access | `SELECT user, host FROM mysql.user;` | Must have a `%` or `10.1.%` host entry |
| MySQL port | `sudo ss -tlnp \| grep mysql` (default 3306) | `extraEnv.SOURCE_MYSQL_PORT` |

### Step-by-step network path

```
[Superset pod]
     |
     | tcp to SOURCE_MYSQL_HOST:SOURCE_MYSQL_PORT
     |
[MicroK8s CNI bridge]  (pod CIDR, typically 10.1.0.0/16)
     |
     | routes to VM network interface
     |
[VM eth0 / ens3]  (VM private IP, e.g. 10.114.0.2)
     |
     | IF mysqld bind-address includes this interface AND
     | ufw allows from pod CIDR to port 3306 AND
     | MySQL user has a matching GRANT
     |
[mysqld on VM]
```

### Changes required on the VM

**a. `my-values.yaml` — set the private IP:**

```yaml
extraEnv:
  SOURCE_MYSQL_HOST: "10.114.0.2"   # replace with your VM's actual private IP; done on the prod version
  SOURCE_MYSQL_PORT: "3306"
  SOURCE_MYSQL_DATABASE: "membership_ard"
  SOURCE_MYSQL_USER: "your_mysql_user"
```

**b. MySQL bind-address (`/etc/mysql/mysql.conf.d/mysqld.cnf`):**

```ini
[mysqld]
bind-address = 0.0.0.0
```

Then: `sudo systemctl restart mysql`

**c. Firewall rule (ufw):**

```sh
sudo ufw allow from 10.1.0.0/16 to any port 3306
```

Adjust the CIDR to match your actual pod CIDR.

**d. MySQL user GRANT:**

```sql
-- Connect as root on the VM
GRANT ALL PRIVILEGES ON membership_ard.* TO 'your_user'@'10.1.%' IDENTIFIED BY 'your_password';
FLUSH PRIVILEGES;
```

### How Superset connects to MySQL (the SQLAlchemy URI)

Once the above is in place, you register the source database in the Superset UI:

1. Go to **Settings → Database Connections → + Database**
2. Select **MySQL**
3. Use the SQLAlchemy URI:

```
mysql+mysqlconnector://SOURCE_MYSQL_USER:SOURCE_MYSQL_PASSWORD@SOURCE_MYSQL_HOST:SOURCE_MYSQL_PORT/membership_ard
```

With concrete values:

```
mysql+mysqlconnector://your_user:your_password@10.114.0.2:3306/membership_ard
```

The `mysqlconnector` driver is installed by `bootstrapScript`. The `pymysql` shim in `configOverrides` ensures Superset's internal MySQL engine spec resolves `MySQLdb` correctly without requiring the `mysqlclient` C extension.

---

## Install quick reference

Full instructions in [INSTALL_microk8s.md](INSTALL_microk8s.md). Summary:

```sh
# 1. Generate age key (once)
age-keygen -o ~/.config/sops/age/key.txt
# → paste public key into .sops.yaml

# 2. Create and encrypt secrets
cp environments/dev/secrets.example.yaml environments/dev/secrets.yaml
# fill in SUPERSET_SECRET_KEY and SOURCE_MYSQL_PASSWORD
sops --encrypt --in-place environments/dev/secrets.yaml

# 3. Install helm-secrets plugin (once)
microk8s helm plugin install https://github.com/jkroepke/helm-secrets

# 4. Add Helm repo and install
microk8s helm repo add superset https://apache.github.io/superset && microk8s helm repo update
microk8s helm secrets upgrade --install superset superset/superset \
  --version 0.15.4 -n superset --create-namespace \
  -f custom_config/my-values.yaml \
  -f custom_config/environments/dev/secrets.yaml \
  --wait --timeout 15m

# 5. Access UI
microk8s kubectl port-forward -n superset svc/superset 8088:8088
# → open http://localhost:8088
```
