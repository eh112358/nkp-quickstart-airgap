# Nutanix Kubernetes Platform - Quickstart Guide - AIRGAP mode

## TL;DR

Steps to install all the required CLIs (nkp, kubectl and helm) to create and manage NKP clusters.

1. Add NKP Rocky Linux image from the Nutanix Support Portal to Prism Central

1. Create a jump host with 2 vCPUs, 4 GB memory, use the Rocky image (update disk to 128 GiB), and the following Cloud-init custom script : [cloud-init](./cloud-init)

1. SSH to `nutanix@<jump host_IP>` (default password: nutanix/4u - unless you modified it in the cloud-init file)

1. Install the NKP CLI with the command: [get-nkp-cli](./get-nkp-cli)

    When prompted, you must use the download link as-is, which is available in the Nutanix portal.

1. Download the NKP airgap bundle with the command: [get-nkp-airgap-bundle](./get-nkp-airgap-bundle)

    When prompted, you must use the download link as-is, which is available in the Nutanix portal.

1. Push NKP airgap bundle to your internal registry with the command: [push-nkp-airgap-bundle](./push-nkp-airgap-bundle.sh)

    When prompted, you must use the download link as-is, which is available in the Nutanix portal.


## Table of Contents

1. [Overview](#overview)
1. [Airgap Architecture](#airgap-architecture)
1. [Glossary](#glossary)
1. [Prerequisites Checklist](#prerequisites-checklist)
1. [Deploy Linux jump host](#deploy-linux-jump-host)
1. [Install NKP CLI](#install-nkp-cli)
1. [Airgap Deployment Guide](#airgap-deployment-guide)
1. [Creating Workload Clusters](#creating-workload-clusters)
1. [Security Considerations](#security-considerations)
1. [Troubleshooting](#troubleshooting)

## Overview

The NKP CLI is a command-line interface for managing NKP-based workflows. This guide provides a quick and easy way to install the required CLIs (nkp, kubectl and helm) using the Rocky Linux image provided by Nutanix in the [Nutanix Support Portal](https://portal.nutanix.com/page/downloads?product=nkp).

## Glossary

| Term | Definition |
|------|------------|
| **Airgap** | A network security measure where a computer or network is physically isolated from unsecured networks, including the internet. Airgapped environments require all software and updates to be transferred via physical media or secure, controlled channels. |
| **NKP** | Nutanix Kubernetes Platform - Nutanix's enterprise Kubernetes distribution based on D2iQ Konvoy. |
| **Konvoy** | The Kubernetes distribution that forms the foundation of NKP, providing cluster lifecycle management. |
| **Kommander** | The multi-cluster management component of NKP that provides a unified dashboard, policy management, and catalog applications. |
| **Management Cluster** | The primary NKP cluster that hosts Kommander and manages the lifecycle of workload clusters. It serves as the control plane for your Kubernetes infrastructure. |
| **Workload Cluster** | Kubernetes clusters created and managed by the management cluster, used to run application workloads. These are separate from the management cluster to isolate workloads. |
| **Jump Host** | A bastion or gateway VM used to access and manage infrastructure in a secured or airgapped environment. In this guide, it runs the NKP CLI and orchestrates deployments. |
| **Control Plane** | The set of Kubernetes components that manage the cluster state, including the API server, scheduler, and controller manager. Control plane nodes run these components. |
| **VIP** | Virtual IP address - A floating IP address used for high availability that can move between nodes. Used for the Kubernetes API server endpoint. |
| **Prism Central** | Nutanix's centralized management interface for managing multiple Nutanix clusters. |
| **QCOW2** | QEMU Copy On Write version 2 - A disk image format used for virtual machine images. The Rocky Linux image is provided in this format. |
| **Failure Domain** | A logical grouping that defines fault isolation boundaries (e.g., different racks, availability zones). Used to spread cluster nodes for high availability. |

## Airgap Architecture

An **airgapped environment** is a network that is physically isolated from unsecured networks (like the internet) for security purposes. This is common in government, financial, healthcare, and other regulated industries.

### How Airgap Deployment Works

In an airgap deployment, you cannot pull container images directly from public registries. Instead, you must:

1. **Download** the NKP airgap bundle on a machine with internet access
2. **Transfer** the bundle to your airgapped environment (via physical media or secure transfer)
3. **Push** the container images to an internal registry within the airgapped network
4. **Deploy** NKP clusters that pull images from your internal registry

### Airgap Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INTERNET-CONNECTED ZONE                             │
│  ┌──────────────┐     ┌─────────────────────────────────────────────────┐   │
│  │   Nutanix    │────▶│  Download Machine                               │   │
│  │   Portal     │     │  - get-nkp-cli                                  │   │
│  └──────────────┘     │  - get-nkp-airgap-bundle.sh                     │   │
│                       │  - Rocky Linux QCOW2 image                      │   │
│                       └─────────────────────────────────────────────────┘   │
└───────────────────────────────────────┬─────────────────────────────────────┘
                                        │ Transfer via
                                        │ USB/DVD/Secure Copy
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AIRGAPPED ZONE                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Jump Host (Rocky Linux VM)                                         │    │
│  │  - push-nkp-airgap-bundle.sh ──▶ Push images to internal registry   │    │
│  │  - nkp-create-image.sh ──▶ Create Rocky Linux VM template           │    │
│  │  - nkp-create-mgmt-cluster.sh ──▶ Deploy management cluster         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                       │                                                     │
│                       ▼                                                     │
│  ┌──────────────────────────┐     ┌──────────────────────────────────┐     │
│  │  Internal Registry       │     │  NKP Management Cluster          │     │
│  │  (Harbor, Docker, etc.)  │◀───▶│  - Control Plane Nodes           │     │
│  │  - Konvoy images         │     │  - Worker Nodes                  │     │
│  │  - Kommander images      │     │  - Kommander Dashboard           │     │
│  │  - Catalog apps          │     └──────────────────────────────────┘     │
│  └──────────────────────────┘                    │                          │
│                                                  ▼                          │
│                                    ┌──────────────────────────────────┐     │
│                                    │  Workload Clusters (Optional)    │     │
│                                    │  - Application workloads         │     │
│                                    └──────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Description |
|-----------|-------------|
| **Jump Host** | Rocky Linux VM used to run NKP CLI commands and orchestrate deployment |
| **Internal Registry** | Container registry (Harbor, Docker Registry, etc.) that stores NKP images |
| **Management Cluster** | The primary NKP cluster that manages workload clusters and runs Kommander |
| **Workload Cluster** | Additional clusters created from the management cluster for running applications |

## Prerequisites Checklist

### Nutanix Infrastructure

- Nutanix AOS 6.5 or later
- Prism Central 2022.6 or later
- Sufficient cluster resources:
  - Management cluster: minimum 3 control plane nodes (4 vCPU, 16 GB RAM each) + 4 worker nodes (8 vCPU, 32 GB RAM each)
  - Additional resources for workload clusters as needed

### For Initial Setup (Internet-Connected Machine)

- Internet connectivity to download from Nutanix portal
- Approximately 50 GB disk space for the airgap bundle
- Add NKP Rocky Linux to Prism Central. **DO NOT CHANGE** the auto-populated image name

    <details>
    <summary>click to view example</summary>
    <IMG src="./images/add_nkp_rocky_os_image.png" atl="Add NKP Rocky OS image" />
    </details>

### For Airgap Environment

- Internal container registry (Harbor, Docker Registry, Nexus, etc.)
  - Must support Docker V2 API
  - TLS certificate (self-signed or CA-signed)
  - Sufficient storage for NKP images (~30 GB)
  - **Note:** A local registry can be deployed directly on the jump host using the [prep-rocky-local-repo.sh](./offline-jumphost-test/prep-rocky-local-repo.sh) script. See the [offline-jumphost-test](./offline-jumphost-test/) directory for details.
- Jump host VM:
  - 2 vCPUs, 4 GB RAM, 128 GB disk
  - Network access to Prism Central and internal registry
  - Docker installed (provided by cloud-init)

### Network Requirements

- Static IP address for the control plane VIP
- One or more IP addresses for the NKP dashboard and load balancing service
- All IP addresses must be in the same subnet as cluster VMs
- DNS resolution for Prism Central endpoint
- Network connectivity between:
  - Jump host ↔ Prism Central
  - Jump host ↔ Internal registry
  - Cluster nodes ↔ Internal registry
  - Cluster nodes ↔ Prism Central

## Deploy Linux jump host

1. Connect to Prism Central

1. Create a virtual machine

    - Name: nkp-jump host
    - vCPUs: 2
    - Memory: 4
    - Disk: Clone from Image (select the Rocky Linux you previously uploaded)
    - Disk Capacity: 128 (default is 20)
    - Guest Customization: Cloud-init (Linux)
    - Custom Script: [cloud-init](./cloud-init)

1. Power on the virtual machine

## Install NKP CLI

1. Connect to your jump host using SSH (default password: nutanix/4u)

    ```shell
    ssh nutanix@<jump host_IP>
    ```

1. Install the NKP CLI with the command: [get-nkp-cli](./get-nkp-cli)

    ```shell
    curl -sL https://raw.githubusercontent.com/nutanixdev/nkp-quickstart/main/get-nkp-cli | bash
    ```

    When prompted, you must use the download link as-is, which is available in the Nutanix portal.

## Airgap Deployment Guide

This section provides detailed step-by-step instructions for deploying NKP in an airgapped environment.

### Step 1: Download the Airgap Bundle (Internet-Connected Machine)

Run the bundle download script:

```shell
./get-nkp-airgap-bundle.sh
```

**What this does**:
- Prompts for the Nutanix portal download link
- Downloads the airgap bundle (~50 GB)
- Extracts to `./nkp-{version}/` directory
- Creates `bundle-path` file storing the bundle location

**Expected output**:
```
Enter the download link: [paste link from portal]
Downloading...
Extracting...
Bundle extracted to: /home/nutanix/nkp-2.x.x
```

**Verification**:
```shell
cat bundle-path
ls -la $(cat bundle-path)
```

### Step 2: Transfer to Airgapped Environment

Transfer these items to your airgapped jump host:
- The entire `nkp-{version}/` directory
- The `bundle-path` file
- The repository scripts

**Transfer methods**:
- USB drive
- Secure file transfer
- DVD/Blu-ray media

### Step 3: Configure Environment Variables

Edit the `nkp-env` file with your environment-specific values:

```shell
vi nkp-env
```

#### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `CLUSTER_NAME` | Name for your NKP cluster | `nkp-airgap-prod` |
| `NUTANIX_USER` | Prism Central username | `admin` |
| `NUTANIX_PASSWORD` | Prism Central password | `'YourPassword'` |
| `NUTANIX_ENDPOINT` | Prism Central IP/FQDN | `10.0.0.100` |
| `NUTANIX_PORT` | Prism Central port | `9440` |
| `CONTROL_PLANE_ENDPOINT_IP` | Kubernetes API VIP | `10.0.0.150` |
| `LB_IP_RANGE` | Load balancer IP pool | `10.0.0.151-10.0.0.160` |
| `NUTANIX_PRISM_ELEMENT_CLUSTER_NAME` | PE cluster name | `cluster-01` |
| `NUTANIX_SUBNET_NAME` | Network subnet | `vm-network` |
| `NUTANIX_STORAGE_CONTAINER_NAME` | Storage container | `default-container` |
| `CONTROL_PLANE_REPLICAS` | Control plane nodes | `3` |
| `WORKER_NODES_REPLICAS` | Worker nodes | `4` |

#### Registry Variables (for push-nkp-airgap-bundle.sh)

| Variable | Description | Example |
|----------|-------------|---------|
| `AIRGAP_REGISTRY_MIRROR_URL` | Internal registry URL | `registry.local/nkp` |
| `AIRGAP_REGISTRY_MIRROR_USERNAME` | Registry username | `admin` |
| `AIRGAP_REGISTRY_MIRROR_PASSWORD` | Registry password | `'RegistryPass'` |

#### Registry Variables (for cluster deployment)

| Variable | Description | Example |
|----------|-------------|---------|
| `REGISTRY_MIRROR_URL` | Registry URL for cluster | `registry.local/nkp` |
| `REGISTRY_MIRROR_USERNAME` | Registry username | `admin` |
| `REGISTRY_MIRROR_PASSWORD` | Registry password | `'RegistryPass'` |

### Step 4: Push Images to Internal Registry

Ensure your registry variables are configured in `nkp-env`, then run:

```shell
source nkp-env
./push-nkp-airgap-bundle.sh
```

**What this does**:
1. Reads bundle path from `bundle-path` file
2. Extracts registry CA certificate
3. Pushes Konvoy images to registry
4. Pushes Kommander images to registry
5. Pushes catalog applications (if present)
6. Loads bootstrap images into local Docker

**Expected output**:
```
Reading bundle path...
Extracting registry CA certificate...
Pushing konvoy-image-bundle...
Pushing kommander-image-bundle...
Loading bootstrap images...
Done.
```

**Verification**:
```shell
# Check images in registry (example for Harbor)
curl -k -u "$AIRGAP_REGISTRY_MIRROR_USERNAME:$AIRGAP_REGISTRY_MIRROR_PASSWORD" \
  https://$AIRGAP_REGISTRY_MIRROR_URL/v2/_catalog

# Check local Docker images
docker images | grep -E "(konvoy|kommander|nkp)"
```

### Step 5: Create Rocky Linux VM Image

Run the image creation script:

```shell
./nkp-create-image.sh
```

**What this does**:
1. Presents menu of available OS versions
2. Creates VM image in Prism Central
3. Uploads QCOW2 image for cluster node deployment

**Interactive prompt**:
```
Available OS versions:
1) rocky-9.4
2) rocky-9.5
Select OS version [1-2]: 2
```

**Expected output**:
```
Creating image: nkp-rocky-9.5-release-1.30.5-20241125163629.qcow2
Uploading to Prism Central...
Image created successfully.
```

**IMPORTANT**: Copy the image name from the output. You will need it for the next step.

**Verification**:
- Log into Prism Central
- Navigate to Images
- Confirm the new NKP Rocky image appears

### Step 6: Update nkp-env with Image Name

Edit `nkp-env` and set the image name from Step 5:

```shell
vi nkp-env
```

Update this line:
```bash
NUTANIX_MACHINE_TEMPLATE_IMAGE_NAME=nkp-rocky-9.5-release-1.30.5-20241125163629.qcow2
```

### Step 7: Deploy Management Cluster

Start a tmux session (recommended for long-running operations):

```shell
tmux new -s nkp-deploy
```

Run the cluster creation script:

```shell
source nkp-env
./nkp-create-mgmt-cluster.sh
```

**What this does**:
1. Validates configuration
2. Creates bootstrap cluster (local Kind cluster)
3. Deploys control plane nodes on Nutanix
4. Deploys worker nodes on Nutanix
5. Installs Kommander (multi-cluster management)
6. Pivots to self-managed cluster

**Duration**: 30-60 minutes depending on infrastructure

**Monitoring progress**:
```shell
# In another terminal, watch cluster resources
export KUBECONFIG=$(pwd)/$CLUSTER_NAME.conf
kubectl get nodes -w
kubectl get pods -A -w
```

**Expected completion output**:
```
Cluster created successfully.
Kubeconfig written to: ./nkp-airgap-prod.conf
```

### Step 8: Access the Cluster

Set your kubeconfig:

```shell
export KUBECONFIG=$(pwd)/$CLUSTER_NAME.conf
```

Verify cluster access:

```shell
kubectl get nodes
kubectl get pods -A
```

Access Kommander dashboard:

```shell
# Get the dashboard URL
kubectl -n kommander get svc kommander-traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Get admin credentials
kubectl -n kommander get secret dkp-credentials -o jsonpath='{.data.username}' | base64 -d
kubectl -n kommander get secret dkp-credentials -o jsonpath='{.data.password}' | base64 -d
```

### Post-Deployment Checklist

- [ ] All nodes show `Ready` status
- [ ] All system pods are `Running`
- [ ] Kommander dashboard is accessible
- [ ] Can create test namespace and deployment
- [ ] Storage class is available and functional

## Creating Workload Clusters

After your management cluster is running, you can create workload clusters to run your applications. Workload clusters are managed by the management cluster and pull images from your internal registry.

### Using the Workload Cluster Scripts

The [workload-clusters](./workload-clusters/) directory contains scripts for creating and managing workload clusters:

1. **Configure the environment**: Edit [cluster-env](./workload-clusters/cluster-env) with your workload cluster settings:
   - Cluster name
   - Control plane and worker node counts
   - Resource sizing
   - Network configuration

2. **Generate the cluster manifest**: Run [nkp-create-workload-cluster.sh](./workload-clusters/nkp-create-workload-cluster.sh) to generate a YAML manifest for your workload cluster.

3. **Deploy the cluster**: Apply the generated manifest to your management cluster:
   ```shell
   kubectl apply -f <cluster-name>.yaml --server-side=true
   ```

4. **Retrieve kubeconfig**: Use [get-and-merge-kubeconfig.sh](./workload-clusters/get-and-merge-kubeconfig.sh) to retrieve the workload cluster's kubeconfig and merge it with your existing kubeconfig file.

### Management vs Workload Clusters

| Aspect | Management Cluster | Workload Cluster |
|--------|-------------------|------------------|
| **Purpose** | Runs Kommander, manages other clusters | Runs application workloads |
| **Created by** | NKP CLI on jump host | Management cluster via kubectl |
| **Hosts Kommander** | Yes | No |
| **Typical count** | 1 per environment | Multiple, as needed |

## Security Considerations

### Default Credentials

The cloud-init script creates a default user with password `nutanix/4u`. **Change this password immediately** after first login:

```shell
passwd
```

For production environments, consider:
- Using SSH key authentication instead of passwords
- Disabling password authentication in `/etc/ssh/sshd_config`
- Configuring a unique password in the cloud-init file before deployment

### Registry Credentials

When pushing images to your internal registry, credentials are passed via environment variables:
- `AIRGAP_REGISTRY_MIRROR_URL`
- `AIRGAP_REGISTRY_MIRROR_USERNAME`
- `AIRGAP_REGISTRY_MIRROR_PASSWORD`

Protect these credentials by:
- Not storing them in shell history (prefix commands with a space)
- Using a secrets manager when possible
- Rotating credentials regularly

### TLS Certificates

The internal registry should use TLS. The `push-nkp-airgap-bundle.sh` script retrieves the registry's CA certificate automatically. Ensure your registry certificate is properly configured and trusted by all cluster nodes.

## Troubleshooting

### Common Issues

#### Bundle download fails or is corrupted
- **Symptom**: `get-nkp-airgap-bundle.sh` fails or extracted files are incomplete
- **Solution**: Verify the download link from Nutanix portal is complete and hasn't expired. Re-download if necessary. Ensure sufficient disk space (50+ GB).

#### Cannot connect to internal registry
- **Symptom**: `push-nkp-airgap-bundle.sh` fails with connection errors
- **Solution**:
  - Verify network connectivity: `curl -v https://<registry-url>/v2/`
  - Check registry credentials are correct
  - Ensure the registry's TLS certificate is valid

#### Cluster creation hangs or fails
- **Symptom**: `nkp-create-mgmt-cluster.sh` doesn't complete
- **Solution**:
  - Check Prism Central connectivity from jump host
  - Verify all IP addresses in `nkp-env` are correct and available
  - Ensure sufficient resources on Nutanix cluster
  - Review bootstrap cluster logs: `docker logs -f <bootstrap-container>`

#### Images not found during cluster creation
- **Symptom**: Pods fail to start with `ImagePullBackOff`
- **Solution**:
  - Verify all images were pushed to registry: check registry UI or API
  - Confirm `REGISTRY_MIRROR_URL` in `nkp-env` matches your registry
  - Check cluster nodes can reach the registry

#### SSH connection drops during installation
- **Symptom**: Long-running commands interrupted
- **Solution**: Use `tmux` or `screen` to persist sessions:
  ```shell
  tmux new -s nkp
  # Run your commands, then detach with Ctrl+B, D
  # Reattach with: tmux attach -t nkp
  ```

### Validation Checklist

Before creating a cluster, verify:

- [ ] Jump host can reach Prism Central: `curl -k https://<prism-central>:9440`
- [ ] Jump host can reach internal registry: `curl -v https://<registry>/v2/`
- [ ] Docker is running: `docker info`
- [ ] NKP CLI is installed: `nkp version`
- [ ] Bundle path file exists: `cat bundle-path`
- [ ] Environment variables are set: `source nkp-env && echo $CLUSTER_NAME`

### Getting Help

- Review logs in `/tmp/` for script output
- Check Docker container logs for bootstrap issues
- Use `kubectl get events` for cluster-level issues
- For cluster status: `./tools/cluster-list.sh`

## Support and Disclaimer

These code samples are intended as standalone examples. Please be aware that all public code samples provided by Nutanix are unofficial in nature, are provided as examples only, are unsupported, and will need to be heavily scrutinized and potentially modified before they can be used in a production environment. All such code samples are provided on an as-is basis, and Nutanix expressly disclaims all warranties, express or implied. All code samples are © Nutanix, Inc., and are provided as-is under the MIT license (<https://opensource.org/licenses/MIT>).
