# Deploy an NVIDIA A3 GKE Cluster for ML Training

This blueprint provisions a Google Kubernetes Engine (GKE) cluster optimized for large-scale multi-node AI/ML training and HPC workloads across the entire NVIDIA A3 GPU family.

It unifies deployment for:
- **NVIDIA A3 High (`a3-highgpu-8g`)**: 8x NVIDIA H100 80GB GPUs with GPUDirect TCPX (4 dedicated gVNIC network interfaces).
- **NVIDIA A3 Mega (`a3-megagpu-8g`)**: 8x NVIDIA H100 80GB GPUs with GPUDirect TCPXO (8 dedicated network interfaces).
- **NVIDIA A3 Ultra (`a3-ultragpu-8g`)**: 8x NVIDIA H200 141GB GPUs with GPUDirect RoCE RDMA (8 dedicated RoCE network interfaces).

---

## Architecture Overview

The blueprint automatically configures the following components to enable optimal GPU performance and multi-networking:

- **GPU-Direct Networking Stack**:
  - **A3 High**: GPU-Direct TCPX stack for high-bandwidth, low-latency communication across 4 secondary VPCs.
  - **A3 Mega**: GPU-Direct TCPXO stack with hardware offload across 8 secondary VPCs.
  - **A3 Ultra**: GPU-Direct RoCE (gIB) RDMA stack across 8 dedicated RoCE subnets with DSCP/ECN QoS profiles.
- **Multi-networking**: Configures 4 or 8 dedicated secondary VPC networks for isolated GPU-to-GPU data plane communication.
- **NRI Device Injector**: Automatically injects required networking, shared memory (`/dev/shm`), and GPU configurations into your ML workload containers.
- **Workload Management**:
  - **Kueue**: Kubernetes-native queue management with multi-tenant scheduling, custom resource flavors, and Topology Aware Scheduling (TAS).
  - **JobSet**: Distributed training job orchestration and multi-pod gang-scheduling.
- **Storage Integrations**: GCS FUSE CSI driver enabled by default, with optional support for Managed Lustre / Parallelstore.

---

## Prerequisites

1. **Cluster Toolkit**:
   - Install [dependencies](https://docs.cloud.google.com/cluster-toolkit/docs/setup/install-dependencies).
   - Set up [Cluster Toolkit](https://docs.cloud.google.com/cluster-toolkit/docs/setup/configure-environment). Build the `gcluster` binary following [Install Cluster Toolkit](https://docs.cloud.google.com/cluster-toolkit/docs/setup/configure-environment#install).
2. **Quota**: Ensure you have sufficient quota for the target machine type (`a3-highgpu-8g`, `a3-megagpu-8g`, or `a3-ultragpu-8g`) in your selected GCP region and zone.
3. **IP Address**: You will need the public IP address of the workstation where you execute `gcluster` to configure the cluster's authorized networks.

---

## Configuration

Select the deployment configuration file matching your hardware:
- **A3 High**: [`a3-highgpu-deployment.yaml`](./a3-highgpu-deployment.yaml)
- **A3 Mega**: [`a3-megagpu-deployment.yaml`](./a3-megagpu-deployment.yaml)
- **A3 Ultra**: [`a3-ultragpu-deployment.yaml`](./a3-ultragpu-deployment.yaml)

Edit the deployment file with your project-specific values:

| Variable | Description |
| :--- | :--- |
| `project_id` | Your Google Cloud Project ID. |
| `deployment_name` | A unique name for this Cluster Toolkit deployment. |
| `region` / `zone` | The GCP region and zone (e.g., `us-central1` / `us-central1-c` for High, `us-east5` / `us-east5-a` for Mega, `us-central1` / `us-central1-a` for Ultra). |
| `authorized_cidr` | Your public IP address in CIDR notation (e.g., `1.2.3.4/32`). |
| `static_node_count` | Number of A3 GPU nodes to provision in the primary node pool. |
| `reservation` | (Optional) The name of a GCE reservation to consume. |
| `bucket` | Name of the GCS bucket to store Terraform state. |
| `queue_name` | (Optional) Local Kueue queue name (default: `a3-gpu`). |

### Additional Consumption Options

The Cluster Toolkit supports alternative consumption models such as Spot VMs or Dynamic Workload Scheduler (DWS) Flex-start. Refer to step 5 of [Create a cluster using Cluster Toolkit](https://docs.cloud.google.com/ai-hypercomputer/docs/create/gke-ai-hypercompute#use-cluster-toolkit) for configuration examples.

---

## Deploy the Cluster

1. Navigate to the cluster-toolkit root directory:
   ```bash
   cd ~/cluster-toolkit
   ```

2. Build the toolkit:
   ```bash
   make
   ```

3. Deploy the infrastructure:

   **For A3 High:**
   ```bash
   ./gcluster deploy examples/gke-a3/gke-a3.yaml -d examples/gke-a3/a3-highgpu-deployment.yaml
   ```

   **For A3 Mega:**
   ```bash
   ./gcluster deploy examples/gke-a3/gke-a3.yaml -d examples/gke-a3/a3-megagpu-deployment.yaml
   ```

   **For A3 Ultra:**
   ```bash
   ./gcluster deploy examples/gke-a3/gke-a3.yaml -d examples/gke-a3/a3-ultragpu-deployment.yaml
   ```

---

## Post-Deployment Verification

1. Get cluster credentials:
   ```bash
   gcloud container clusters get-credentials DEPLOYMENT_NAME --region REGION --project PROJECT_ID
   ```

2. Verify driver installers and device injectors are running on all nodes:
   ```bash
   kubectl get daemonsets -n kube-system
   kubectl get pods -n kube-system | grep device-injector
   ```

   Ensure the corresponding driver installer (`nccl-tcpx-installer`, `nccl-tcpxo-installer`, or `nccl-rdma-installer`) and `device-injector` pods report `Running`/`READY`.

---

## Verify NCCL Performance & Run Benchmarks

Refer to the following guides and official Google Cloud documentation to verify GPU and inter-node networking performance using NVIDIA `nccl-tests`:

### 1. A3 High GPU (`a3-highgpu-8g`)
- **Included Manifests**: [`nccl-config.yaml`](./nccl-config.yaml) and [`nccl-test.yaml`](./nccl-test.yaml).
- **Multi-Node Test Plan**: Refer to the [`multi-node-test-plan.md`](./multi-node-test-plan.md) walkthrough for multi-node (16+ GPUs) NCCL test execution with TCPX socket and CPU core pinning.
- **Official Guide**: [Test GPU performance with custom NCCL on A3 High and A3 Mega](https://docs.cloud.google.com/ai-hypercomputer/docs/nccl/test-gke-custom-a3-mega-high).

### 2. A3 Mega GPU (`a3-megagpu-8g`)
- **Included Manifest**: [`nccl-test-latest.yaml`](./nccl-test-latest.yaml).
- **Official Guides**:
  - [Test GPU performance with custom NCCL on A3 High and A3 Mega](https://docs.cloud.google.com/ai-hypercomputer/docs/nccl/test-gke-custom-a3-mega-high).
  - [Deploy and run NCCL test with Topology Aware Scheduling (TAS)](https://docs.cloud.google.com/ai-hypercomputer/docs/nccl/test-gke#a3-mega).

### 3. A3 Ultra GPU (`a3-ultragpu-8g`)
- **Included Manifest**: [`nccl-jobset-example.yaml`](./nccl-jobset-example.yaml) for multi-node RDMA RoCE testing via JobSet.
- **Automated Benchmarks**: See the [`system_benchmarks/`](./system_benchmarks/) directory for Ramble experiment configurations for HPL, NCCL, and NeMo.
- **Official Guides**:
  - [Deploy and run NCCL test with Topology Aware Scheduling (TAS) on A3 Ultra](https://cloud.google.com/ai-hypercomputer/docs/nccl/test-gke#a3-ultra).
  - [Create an AI-optimized GKE cluster with default configuration](https://cloud.google.com/ai-hypercomputer/docs/create/gke-ai-hypercompute#use-cluster-toolkit).

---

## Clean Up

To avoid incurring charges for the provisioned resources, destroy the deployment:

```bash
./gcluster destroy DEPLOYMENT_NAME
```

> [!NOTE]
> GCS buckets created for Terraform state backend storage are retained by design upon `./gcluster destroy` and must be deleted manually if no longer needed.

