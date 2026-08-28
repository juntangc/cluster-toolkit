# NVIDIA A3 GPU GKE Cluster Blueprint

This directory contains the unified **Google Cloud Cluster Toolkit** blueprint for deploying GKE clusters optimized for large-scale multi-node AI/ML training and HPC workloads on NVIDIA A3 GPU architectures.

It consolidates support for:
- **NVIDIA A3 High (`a3-highgpu-8g`)**: 8x NVIDIA H100 80GB GPUs with GPUDirect TCPX (4 gVNIC network interfaces).
- **NVIDIA A3 Mega (`a3-megagpu-8g`)**: 8x NVIDIA H100 Mega 80GB GPUs with GPUDirect TCPXO (8 network interfaces).
- **NVIDIA A3 Ultra (`a3-ultragpu-8g`)**: 8x NVIDIA H200 141GB GPUs with GPUDirect RoCE (8 RDMA network interfaces).

---

## Architecture Overview

The blueprint provisions:
1. **Primary VPC & Subnets**: Regional VPC with secondary IP ranges for Kubernetes Pods (`/14`) and Services (`/20`).
2. **GPU Data Plane (Multi-VPC)**: 4 to 8 dedicated VPC networks for high-throughput inter-node GPU communications.
3. **GKE Cluster & Node Pool**: A GKE cluster with Workload Identity, GCS Fuse CSI driver, and a dedicated 8-GPU node pool.
4. **Workload Management**:
   - **Kueue (`0.17.1`)**: Multi-tenant workload scheduling, Topology-aware placement, and ResourceFlavors.
   - **JobSet**: Distributed training job orchestration.
5. **Drivers & Accelerator Plugins**:
   - `nri-device-injector`: NRI plugin DaemonSet for multi-NIC device injection.
   - Hardware-specific NCCL plugin DaemonSet (TCPX, TCPXO, or gIB RDMA).

---

## Quick Start

### 1. Choose Your Hardware Flavor

Select the appropriate deployment configuration file:
- `a3-highgpu-deployment.yaml`: For A3 High (H100 + TCPX)
- `a3-megagpu-deployment.yaml`: For A3 Mega (H100 + TCPXO)
- `a3-ultragpu-deployment.yaml`: For A3 Ultra (H200 + RoCE)

### 2. Configure Deployment Variables

Edit the selected deployment file to set your GCP project details:

```yaml
vars:
  project_id: <YOUR-GCP-PROJECT-ID>
  deployment_name: my-a3-cluster
  region: us-central1
  zone: us-central1-c
  authorized_cidr: <YOUR-IP-ADDRESS>/32
  static_node_count: 2
  reservation: <OPTIONAL-RESERVATION-NAME>
```

### 3. Deploy the Cluster

Use `gcluster` to create and deploy the infrastructure:

```bash
# Create the Terraform deployment files
./gcluster create examples/gke-a3/gke-a3.yaml -d examples/gke-a3/a3-highgpu-deployment.yaml

# Deploy resources to Google Cloud
./gcluster deploy gke-a3-cluster
```

### 4. Connect and Run Sample Benchmarks

Get GKE credentials:

```bash
gcloud container clusters get-credentials my-a3-cluster --region us-central1 --project <YOUR-GCP-PROJECT-ID>
```

Run the benchmark corresponding to your deployed hardware flavor:

- **For A3 High (TCPX)**:
  ```bash
  kubectl apply -f examples/gke-a3/nccl-test.yaml
  ```
- **For A3 Mega (TCPXO)**:
  ```bash
  kubectl apply -f examples/gke-a3/nccl-test-latest.yaml
  ```
- **For A3 Ultra (RoCE / JobSet)**:
  ```bash
  kubectl apply -f examples/gke-a3/nccl-jobset-example.yaml
  ```
  *(Or see `system_benchmarks/` for automated Ramble benchmarks for HPL, NCCL, and NeMo).*

---

## Teardown

To cleanly destroy all provisioned infrastructure:

```bash
./gcluster destroy gke-a3-cluster
```
