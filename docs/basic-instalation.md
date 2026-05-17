# CNV Cluster Setup and OVN-Kubernetes UDN Configuration

This document outlines the procedure for setting up an OpenShift Container Platform cluster with the Container-native Virtualization (CNV) operator, followed by the configuration of User Defined Networks (UDN) for Virtual Machines (VMs) in both standard and Data Processing Unit (DPU) environments.

## 1. Cluster Initialization and CNV Operator Installation

The first phase involves preparing the OpenShift cluster to support virtualization. Refer to the official Red Hat documentation for [Creating a Virtual Machine](https://docs.openshift.com/container-platform/latest/virt/virtual_machines/creating_vms_rh/virt-creating-vms.html) for detailed prerequisites.

### Steps

1. **Install the OpenShift Virtualization Operator** from the OperatorHub
2. **Create the HyperConverged Custom Resource (CR)** to deploy the virtualization components
3. **Verify that the virt-handler pods are running** on all eligible worker nodes

```bash
# Verify virt-handler pods
oc get pods -n openshift-cnv | grep virt-handler
```

## 2. Standard OVN-Kubernetes UDN Configuration

Before moving to hardware-accelerated networking, a standard OVN-K UDN must be established.

### Implementation Steps

| Step | Action | Description |
|------|--------|-------------|
| 1 | Define UDN | Create a `UserDefinedNetwork` CR specifying the OVN-Kubernetes configuration |
| 2 | Deploy VM | Instantiate a Virtual Machine that references the UDN in its interface settings |
| 3 | Validation | Confirm network connectivity within the VM and verify the OVN pipeline |

### Example UserDefinedNetwork CR

```yaml
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: my-udn
  namespace: default
spec:
  topology: Layer2
  layer2:
    subnets:
      - "10.100.0.0/24"
```

## 3. DPF and DPU-Accelerated UDN Configuration

Once the standard configuration is validated, the task transitions to clusters running the Data Path Framework (DPF) with DPU worker nodes.

### DPU Implementation Steps

#### Environment Preparation
- Ensure the cluster is running with DPF enabled
- Verify DPU workers are joined to the cluster

```bash
# List DPU worker nodes
oc get nodes -l node-role.kubernetes.io/dpu-worker
```

#### SR-IOV Integration
While a specific SR-IOV pool will be provided later, the initial setup will use a standard UDN configuration on the DPU.

#### VM Deployment on DPU

1. **Schedule the VM** specifically on the DPU-equipped worker node
2. **Configure the VM** to use the UDN network
3. **(Future Task)** Transition the network interface to utilize the SR-IOV pool for hardware acceleration

### Node Affinity Example

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/dpu-worker
            operator: Exists
```

## 4. Success Criteria

- ✅ CNV Operator is successfully deployed and healthy
- ✅ VM on standard OVN-K UDN can communicate across the network
- ✅ VM on DPU worker node is successfully instantiated and utilizes the defined UDN

## Next Steps

- Configure SR-IOV network device pool
- Transition VM interfaces to hardware-accelerated SR-IOV
- Benchmark performance comparison between standard and DPU-accelerated networking