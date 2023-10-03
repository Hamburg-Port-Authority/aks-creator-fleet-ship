# AKS Creator

This tool image allows you to create and manage the configuration files of a AKS Kubernetes repository.

## Getting started

In order to be able to use this image, it must be started in the desired AKS Cluster repository using Docker or Podman.
Furthermore, the repository directory must be mounted in the container.
To avoid permission errors, it is recommended to do this with the permissions of the executing user.

### Igor

We will use the `igor`.
The tool opens a shell in your favorite docker container mounting your current workspace into the container.

The following steps are executed:

- Start the image
- Mount necessary directories
- Set permissions
- Clean up

To install `igor` just download the `igor.sh` and store it in your `$PATH` like this:

    sudo curl https://raw.githubusercontent.com/la-cc/igor/main/igor.sh -o /usr/local/bin/igor
    sudo chmod +x /usr/local/bin/igor

#### Configure igor

Running `igor` without configuration will launch a busybox image. In order to use the tool with the AKS Creator image,
a configuration file is required.

> **_NOTE:_** You can get the recent tag from [la-cc/aks-creator-core](https://github.com/la-cc/aks-creator-fleet-ship/tags)

Open the file `.igor.sh` in your preferred editor and use the following settings to configure `igor`:

    # select docker image
    IGOR_DOCKER_IMAGE=ghcr.io/la-cc/aks-creator-fleet-ship:0.0.X
    IGOR_DOCKER_COMMAND=                                  # run this command inside the docker container
    IGOR_DOCKER_PULL=0                                    # force pulling the image before starting the container (0/1)
    IGOR_DOCKER_RM=1                                      # remove container on exit (0/1)
    IGOR_DOCKER_TTY=1                                     # open an interactive tty (0/1)
    IGOR_DOCKER_USER=$(id -u)                             # run commands inside the container with this user
    IGOR_DOCKER_GROUP=$(id -g)                            # run commands inside the container with this group
    IGOR_DOCKER_ARGS=''                                   # default arguments to docker run
    IGOR_PORTS=''                                         # space separated list of ports to expose
    IGOR_MOUNT_PASSWD=1                                   # mount /etc/passwd inside the container (0/1)
    IGOR_MOUNT_GROUP=1                                    # mount /etc/group inside the container (0/1)
    IGOR_MOUNTS_RO="${HOME}/.azure"                       # space separated list of volumes to mount read only
    IGOR_MOUNTS_RW=''                                     # space separated list of volumes to mount read write
    IGOR_WORKDIR=${PWD}                                   # use this workdir inside the container
    IGOR_WORKDIR_MODE=rw                                  # mount the workdir with this mode (ro/rw)
    IGOR_ENV=''                                           # space separated list of environment variables set inside the container

### Workflow

The following workflow is recommended as part of a aks cluster creation.

| No.                      | Step                                                                          | required | Tool                           |
| ------------------------ | ----------------------------------------------------------------------------- | -------- | ------------------------------ |
| [0](#ConfigAzureBackend) | Allow you configure the azure backend to save the remote state for terraform. | no       | `config-azure-backend`         |
| [1](#InitConfig)         | Initialize empty configuration file.                                          | yes      | `config-init`                  |
| [2](#FillMissing)        | Fill missing fields in configuration file.                                    | yes      | -                              |
| [3](#TemplateConfig)     | Template the whole aks platform folder structure.                             | yes      | `config-template`              |
| [4](#Terraform)          | Create the AKS Cluster with needed context.                                   | yes      | `terrform` (version >= 1.3.0") |

### <a id="ConfigAzureBackend"></a>0. Create a Azure Backend for Terraform State (Optional) `config.yaml`

**Requirements**:

- Azure Active Directory Access
- Azure Subscription Access

The easiest way is to fill the file `.backend.env` with the necessary values.

Then execute the script (from inside the aks-creator-core container):

    config-azure-backend

You can also start the script with interactive mode:

    config-azure-backend -i or config-azure-backend --interactive

### <a id="InitConfig"></a>1. Initialize empty configuration file

**Requirements**: none

Create an empty `config.yaml`.
This already contains the necessary structure and placeholders for the values, as required in the following step.
To do so simply execute the script (from inside the aks-creator-core container):

    config-init

You will get a `config.yaml` that can be filled by you.

### <a id="FillMissing"></a>2. Fill missing fields in `config.yaml`

**Requirements**:

- `config.yaml` of [1. 1. Get configuration file config.yaml](#GetConfig)

You can get more information from [00. Configuration Options for `config.yaml`](#ConfigOptions)

### <a id="TemplateConfig"></a>3. Template the AKS folder structure from `config.yaml`

**Requirements**:

- `config.yaml` of [1. 1. Get configuration file config.yaml](#GetConfig)

To do so simply execute the script (from inside the aks-creator-core container):

    config-template

### <a id="Terraform"></a>4. Terraform Apply

**Requirements**:

- Azure Active Directory Access
- Azure Subscription Access

### 4.1 Terraform Apply + Azure Backend

If go through the step [0. Create a Azure Backend for Terraform State (Optional)](#ConfigAzureBackend) then you need to execute the following commands (from inside the aks-creator-core container or local terraform binary):

    terraform init
    terraform plan -var-file=terraform.tfvars

If the plan is fine for you, then apply it with:

    terraform apply -var-file=terraform.tfvars -auto-approve

### 4.2 Terraform Apply + Local Backend

If you don't create azure backend then execute the following commands (from inside the aks-creator-core container or local terraform binary):

    terraform init
    terraform plan -var-file=terraform.tfvars

If the plan is fine for you, then apply it with:

    terraform apply -var-file=terraform.tfvars -auto-approve

## <a id="ConfigOptions"></a>00. Configuration Options for `config.yaml`

The following examples show the possible configuration of the templating. The used module itself can be further adjusted or overwritten.

### Configuration:

```
---
# Azure Devops Pipeline related data
azure_devops_pipeline:
  library_group: <tkc-fleet-config>

azure_tags:
  maintainer: <"Platform Team">
  owner: <"Platform Team">

# Azure Kubernetes Cluster related data
clusters:
  - name: <vengeance>
    stage: <development>
    kubernetes_version: <1.25.11>
    orchestrator_version: <1.25.11>
    enable_auto_scaling: <true>
    node_pool_count: <2>
    node_pool_min_count: <2>
    node_pool_max_count: <5>
    admin_list: ["8a7...."]
    authorized_ip_ranges: [
        <"1.3.5.7/32", #bastion_vm>
      ]
    grafana_aad_app:
      - name: <"grafana_aks_development">
        display_name: <"Grafana Vengeance Developmenr">
        app_owners:
          <- "101e7...">
        logout_url: <"https://google.de">
        redirect_uris:
          - <"https://grafana.vengeance-dev.hpa-cloud.com/login/azuread">
        roles:
          - name: <"admin_it">
            id: <"1dc...">
            object_id: "8a7042f2-9566-4adf-b9cd-272f39837378" #[PORTDESK]_IT43_ADMIN
    # Azure AD Group related data
    azuread_group:
      name: <"Vengeance_Development_Developer">
      owners: <["platform.engineerl@exmaple.onmicrosoft.com"]>
      members: <["platform.engineerl@exmaple.onmicrosoft.com"]>
    azure_vm:
      jumphost: <true>
    acr:
      name: <"acrvengeancedev">
    # Azure AD User related data
    azuread_user:
      name: <"svc_portdesk-vengeance-dev_devops@example.onmicrosoft.com">
      display_name: <"SVC PortDesk vengeance Development (DevOps)">
      mail_nickname: <"svc_portdesk-vengeance-dev_devops">
    # Azure Key Vault related data
    azure_key_vault:
      git_repo_url: <git@ssh.dev.azure.com:v3/YOUR_ORGA/vengeance/application>
      service_principal_name: <"devops-terraform-cicd">
      svc_user_pw_name: <"svc-vengeance-user-pw">
      name: <"kv-vengeance-dev-713">
      admin_object_ids:
        ID: <"8a70....">
        name: <"IT_ADM">
    # Azure Backend for Terraform related data
    azure_backend:
      resource_group_name: <rg-vengeance-tf-backend>
      storage_account_name: <savengeancetfbackend>
      stage: development
    # Azure Public DNS related data
    azure_public_dns:
      azure_cloud_zone: <vengeance-dev.example.com>
```
