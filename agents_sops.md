## Persona
You have knowledge on SOPS, age and how to integrate and install helm charts that decrypt age files managed by sops using helm secrets plugin.

## Objective
Determine a satisfactory security scheme that encrypts/decrypts database credentials in the helm yaml files that will properly compile when the helm chart is installed (with command `helm secrets upgrade`). Additionally, determine where the 
Review **the proposal** section below.
Why can't I just encrypt the values directly in my-values.yaml using SOPS? Why does it have to be in another file? If I can overwrite my-values.yaml in-place with SOPS, then the values are not in plain text.

## Deliverable
  - an ordered project plan: file creation, SOPS operations, helm operations
  - a list of commands to run on the server: SOPS commands, shell commands, etc.
  - a .sops.yaml file with appropriate values to manage encryption
  - a microk8s helm install command that overlays all necessary my-values and secrets along with the superset chart


## Project knowledge
/custom_config
  /environments/dev/secrets.example.yaml: contains
  /.sops.yaml: contains best estimate on a final version
  /my-values.yaml: contains last best config for the helm chart

## Proposal
Encrypt the following values from the following files:
  - SOURCE_MYSQL_USER: my-values.yaml
  - SOURCE_MYSQL_PASSWORD: my-values.yaml
  - SUPERSET_SECRET_KEY: my-values.yaml

Keep encryption to the /environments/dev/secrets.example.yaml file or encrypt my-values.yaml directly.