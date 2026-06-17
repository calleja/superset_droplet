## Persona
- You understand architecture specs, kubernetes minikube, kubernetes microk8s, helm charts and help business analysts decide between different choices when configuring an Apache Superset ecosystem and configuration.
- Your output: a my-values.yaml file for a superset helm chart that is flexible and compatible with the ecosystem context of this project folder

## Deliverables
- Deliverable is custom_config/my-values.yaml that, with the host-MySQL reachable, makes helm upgrade --install produce a Superset UI on localhost:8088 after kubectl port-forward.
- Provide the appropriate, upgrade command compatible with the final output, ex:
  - `helm chart install/upgrade command: helm install superset apache/superset -n superset — create-namespace -f values.yaml`
  - `helm upgrade --install --values my-values.yaml superset superset/superset`
  
## Project knowledge
- **Tech Stack:** 
  - digitalOcean virtual machine of size: 2 vCPU / 4 GiB VM
  - mysql 8.0.45-0ubuntu0.22.04.1: installed on the VM (not in a pod or container) 
  - Apache Superset 5.0.0
  - helm v4.1.4
  - MicroK8s v1.35.0 revision 8612
- **File Structure:**
  - `my-values_droplet.yaml` - previous iteration of my-values_droplet.yaml
  
## REFACTOR CURRENT YAML TO WORK WITH MICROK8S 
This is the second iteration of this project. A helm chart `my-values.yaml` file was created previously. This project is an attempt to iterate on it. The previous project was configured for Kubernetes minikube; this iteration will integrate with kubernetes microk8s. The original `my-values_droplet.yaml` is a good foundation but must be refactored. 

## DEPENDENCIES AND BITNAMI IMAGES
Superset helm chart depends on postgresql and redis. The previous project tagged particular versions of these images. The proper repo and images are cited on the helm chart download `helm/superset/values.yaml`.

**Background info:** The initial install failed because the chart tried to pull PostgreSQL and Redis images from `docker.io/bitnami/*` using tags that no longer resolved on Docker Hub.

The failing images were:

```text
docker.io/bitnami/postgresql:14.17.0-debian-12-r3
docker.io/bitnami/redis:7.0.10-debian-11-r4
```

Kubernetes reported `manifest unknown` and `ImagePullBackOff`. The chart version and dependency tags were not the core problem. The problem was the repository path. Versioned Bitnami tags are currently available under `bitnamilegacy/*`.

As of the composing of this file, superset relies on postgresql version 16.7.27 and redis version 17.9.4. It is ideal to set up the kubernetes yaml manifests with these versions.


## CHOOSING WHERE TO STORE ENVIRONMENTAL VARIABLES
The most sensitive environmental variables are those of the source database (mysql cited in the "tech stack" section). Provide instructions on how to store a secrets file on the VM, encrypt the keys, decrypt it with SOPS (as necessary) and apply it to the superset cluster or namespace.

On previous version of my-values_droplet.yaml, SECRETS were cited in **configOverrides:**
```
secret: |
    SECRET_KEY = os.environ["SUPERSET_SECRET_KEY"]
```

The virtual machine has SOPS encryption abilities, as well as a helm integration with SOPS (command `helm plugin install https://github.com/jkroepke/helm-secrets`); helm-secrets plugin.

Age keys

Config file to instruct encryption

Folder/file location in project directory <- notice it's placed in `environments/`
my-chart/
├── Chart.yaml
├── values.yaml           # Non-sensitive defaults
├── secrets.yaml          # Encrypted sensitive values
└── environments/
    ├── dev/

## CONNECTING PODS TO MYSQL DB IN VM
**Source MySQL:** The local value `host.minikube.internal` is Minikube-specific. In cloud, replace it with a private DNS name, service endpoint, VPN-reachable hostname, or managed database endpoint.

MicroK8s & server reachability: Host MySQL reachability — the main real migration task. MicroK8s pods use a CNI network; 127.0.0.1 inside a pod is not the VM’s MySQL. Same problem INSTALL_droplet already calls out for any non-Minikube cluster.
MySQL bind address — ensure MySQL listens on an interface pods can hit (0.0.0.0 or the node IP), not only 127.0.0.1, if you use the LAN IP approach.

The `psycopg2-binary` package is required because the Superset 5.0.0 image does not bundle a PostgreSQL Python driver in its venv, but the app uses `postgresql+psycopg2://` by default for the metadata DB SQLAlchemy URI. Made decision here to include the driver in the bootstrapScript.

**MySQL charts and `MySQLdb`:** Superset’s MySQL engine spec still does `import MySQLdb` in some paths (e.g. chart queries). The image does not ship `mysqlclient`. Install `pymysql` in bootstrap and register it in `configOverrides` so `MySQLdb` resolves:

```yaml
configOverrides:
  pymysql_as_mysqldb: |
    try:
      import pymysql
      pymysql.install_as_MySQLdb()
    except Exception:
      pass
```

`uv pip` will be used to install the python mysql connector


  
## EXPOSING SUPERSET UI INTERNALLY & EXTERNALLY
The helm chart will publish appropriate services to expose the Superset UI internally within my microk8s cluster. To access it externally you will have to either:
1. Configure the Service as a LoadBalancer or NodePort
2. Run kubectl port-forward superset-xxxx-yyyy :8088 to directly tunnel one pod's port into your localhost
port forwarding:
kubectl port-forward service/superset 8088:8088 - namespace superset

