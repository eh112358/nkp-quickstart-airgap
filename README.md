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
1. [(Optional) Create NKP Cluster on Nutanix](#optional-create-nkp-cluster-on-nutanix)
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

## (Optional) Create NKP cluster on Nutanix

1. Before you start, ensure you meet the prerequisites:

    - Static IP address for the control plane VIP
    - One or more IP addresses for the NKP dashboard and load-balancing service

    Note: The IP addresses must be in the same subnet as the virtual machines.

1. Choose one of the following two installation methods:

    - **Prompt-based installation**. Use this method when the Internet connection for the NKP cluster isn’t shared with more users.
    - **CLI installation**. Use this method when the Internet connection for the NKP cluster is shared between many users.

### Prompt-based installation

This installation method gives less control on the cluster configuration. For example, the NKP cluster will be created with three control plane nodes and four worker nodes.

We recommend starting a tmux session in case your ssh connection is at risk of disconnection (like laptop going into sleep mode) as the process can take some time based on several paramters (like download speed).

```shell
nkp create cluster nutanix
```

### CLI installation

This installation method lets you fully customize your cluster configuration. The following commands create a cluster with one control plane node and three worker nodes.

1. Before running the following command in your jump host VM, update the values with your environment: [nkp-env](./nkp-env)

1. The next command will start the installation process of an NKP management cluster: [nkp-create-cluster](./nkp-create-mgmt-cluster.sh)

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
