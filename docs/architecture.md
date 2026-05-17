# DPU/DPF Architecture in OpenShift

This document provides a comprehensive architectural overview of Data Processing Units (DPUs) and the DataPath Framework (DPF) within OpenShift Container Platform environments.

## Table of Contents

1. [DPU Architecture Fundamentals](#dpu-architecture-fundamentals)
2. [DPU Purpose and Value Proposition](#dpu-purpose-and-value-proposition)
3. [OpenShift Integration Fundamentals](#openshift-integration-fundamentals)
4. [DataPath Framework (DPF) Architecture](#datapath-framework-dpf-architecture)
5. [Network Data Path Architecture](#network-data-path-architecture)

---

## DPU Architecture Fundamentals

### What is a DPU?

A **Data Processing Unit (DPU)** is a specialized hardware accelerator that offloads infrastructure tasks from the main CPU. Modern DPUs, such as the NVIDIA BlueField-3, are essentially a "system-on-a-chip" that combines:

```
┌─────────────────────────────────────────────────────┐
│             NVIDIA BlueField-3 DPU                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────┐  ┌─────────────────────────────┐ │
│  │   ARM CPU    │  │  Hardware Accelerators       │ │
│  │   Cores      │  │  ┌────────────────────────┐  │ │
│  │  (16 cores)  │  │  │ - Packet Processing    │  │ │
│  │              │  │  │ - Crypto Engine        │  │ │
│  └──────────────┘  │  │ - Compression          │  │ │
│                    │  │ - RegEx Matching       │  │ │
│  ┌──────────────┐  │  │ - Storage Offload      │  │ │
│  │    Memory    │  │  └────────────────────────┘  │ │
│  │  (16-32 GB)  │  └─────────────────────────────┘ │
│  └──────────────┘                                  │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │      Network Interfaces                      │  │
│  │  - 200Gb/s or 400Gb/s Ethernet               │  │
│  │  - PCIe Gen5 connection to host              │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### Key Components

1. **Multi-core ARM CPU**: Runs a full Linux operating system (typically Rocky Linux or RHEL)
2. **Hardware Accelerators**: Fixed-function engines for networking, security, and storage tasks
3. **High-speed Network Interfaces**: 100Gb/s to 400Gb/s Ethernet ports
4. **PCIe Interface**: Connects to the host server's PCIe bus
5. **Local Memory**: 16-32 GB DDR4/DDR5 for DPU operations

### Physical Deployment

In a typical server deployment:

```
┌───────────────────────────────────────────────────────┐
│                   Physical Server                     │
├───────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────────────────────────────────────┐     │
│  │         Host x86 CPU and Memory             │     │
│  │    (OpenShift Worker Node OS and Pods)      │     │
│  └─────────────────┬───────────────────────────┘     │
│                    │ PCIe Gen5                        │
│                    │                                  │
│  ┌─────────────────▼───────────────────────────┐     │
│  │         NVIDIA BlueField-3 DPU              │     │
│  │  ┌──────────────────────────────────────┐   │     │
│  │  │ DPU OS (Rocky Linux/RHEL)            │   │     │
│  │  │ - OVN-Kubernetes components          │   │     │
│  │  │ - SR-IOV VF management               │   │     │
│  │  │ - DPF agents                         │   │     │
│  │  └──────────────────────────────────────┘   │     │
│  └─────────────────┬───────────────────────────┘     │
│                    │                                  │
│                    │ 200Gb/s Ethernet                 │
│                    ▼                                  │
│          [Physical Network Switch]                    │
└───────────────────────────────────────────────────────┘
```

---

## DPU Purpose and Value Proposition

### Primary Objectives

DPUs address three critical challenges in modern cloud infrastructure:

#### 1. **CPU Offload and Performance Isolation**

**Problem**: Networking, storage, and security tasks consume 20-30% of CPU cores in virtualized environments.

**DPU Solution**:
- Network packet processing moved to DPU hardware accelerators
- Host CPU freed for application workloads
- No "noisy neighbor" impact from infrastructure tasks

**Example**: In OpenShift, OVN-Kubernetes overlay networking typically consumes 1-2 CPU cores per node. With DPU offload, this processing happens on the DPU's ARM cores and hardware accelerators.

#### 2. **Security and Isolation**

**Problem**: Tenant workloads share the same network stack as infrastructure services, creating security boundaries at the software level.

**DPU Solution**:
- Physical separation: tenant data plane runs on host, control plane runs on DPU
- Zero-trust architecture: DPU validates all traffic before it reaches the host
- Attack surface reduction: host OS cannot directly access network interfaces

**Security Model**:
```
┌────────────────────────────────────────────────────┐
│  Host (Tenant Workload Domain)                     │
│  - VMs and containers run here                     │
│  - No direct access to physical network            │
│  - Cannot bypass network policies                  │
└──────────────────┬─────────────────────────────────┘
                   │ VirtIO / SR-IOV VF
                   │ (controlled by DPU)
┌──────────────────▼─────────────────────────────────┐
│  DPU (Infrastructure Control Plane)                │
│  - Enforces network policies                       │
│  - Controls QoS, metering, and security            │
│  - Cannot be tampered with by tenant               │
└────────────────────────────────────────────────────┘
```

#### 3. **Network Performance and Scalability**

**DPU Benefits**:
- **Line-rate processing**: 200-400 Gb/s throughput with zero packet loss
- **Consistent latency**: Hardware acceleration provides deterministic performance
- **Connection scale**: Millions of flows tracked in hardware
- **OVS offload**: Open vSwitch tables implemented in hardware

---

## OpenShift Integration Fundamentals

### OpenShift Networking Recap

Before understanding DPU integration, recall OpenShift's networking model:

```
┌──────────────────────────────────────────────────────┐
│         OpenShift Cluster Network Architecture       │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Pod Network (Overlay)                               │
│  └─ OVN-Kubernetes manages pod-to-pod networking    │
│     - 10.128.0.0/14 default pod CIDR                 │
│     - Distributed virtual router per node            │
│     - Network policies enforced in OVN               │
│                                                      │
│  Service Network                                     │
│  └─ Kubernetes Services (ClusterIP, NodePort, LB)   │
│     - 172.30.0.0/16 default service CIDR             │
│                                                      │
│  External Network                                    │
│  └─ Physical network connectivity                   │
│     - Node network interfaces                        │
│     - Ingress/egress routing                         │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### How DPUs Fit into OpenShift

DPUs augment OpenShift's architecture by **moving the network control plane to dedicated hardware**:

#### Traditional OpenShift Worker Node
```
┌─────────────────────────────────────────┐
│       Worker Node (x86 Host)            │
├─────────────────────────────────────────┤
│  ┌────────────────────────────────┐     │
│  │  Application Pods              │     │
│  │  (User Workloads)              │     │
│  └────────────────────────────────┘     │
│                                         │
│  ┌────────────────────────────────┐     │
│  │  OVN-K Components (DaemonSets) │     │
│  │  - ovnkube-node                │     │
│  │  - ovs-node                    │     │
│  │  - OVS kernel module           │     │
│  └────────────────────────────────┘     │
│                                         │
│         ▼ Host CPU Cycles ▼             │
│  [20-30% CPU for networking overhead]   │
└─────────────────────────────────────────┘
```

#### DPU-Accelerated OpenShift Worker Node
```
┌──────────────────────────────────────────────────────┐
│         Host (x86 Worker Node)                       │
├──────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────┐  │
│  │      Application Pods (User Workloads)         │  │
│  │      - 100% CPU available for apps             │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Minimal OVN-K Components                      │  │
│  │  - SR-IOV device plugin                        │  │
│  │  - VF passthrough to pods                      │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│                    ▲ PCIe ▼                          │
└────────────────────┼────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│         DPU (BlueField-3 ARM SoC)                   │
├─────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────┐ │
│  │  OVN-Kubernetes Control Plane                  │ │
│  │  - ovnkube-controller                          │ │
│  │  - OVN Northbound/Southbound DBs              │ │
│  │  - ovn-controller (programs hardware)          │ │
│  └────────────────────────────────────────────────┘ │
│                                                     │
│  ┌────────────────────────────────────────────────┐ │
│  │  OVS with Hardware Offload                     │ │
│  │  - Flow tables in ASIC                         │ │
│  │  - Line-rate packet processing                 │ │
│  └────────────────────────────────────────────────┘ │
│                                                     │
│              ▼ Physical Network ▼                   │
└─────────────────────────────────────────────────────┘
```

### Key Integration Points

1. **Node Roles**: OpenShift cluster has both standard workers and DPU-enabled workers
2. **Network Offload**: OVN-K networking offloaded to DPU for tagged workloads
3. **SR-IOV VFs**: DPU exposes Virtual Functions (VFs) to host for direct pod connectivity
4. **Dual OS Model**: DPU runs its own Linux OS, managed separately from host
5. **Lifecycle Management**: DPF operators manage DPU provisioning and updates

---

## DataPath Framework (DPF) Architecture

### What is DPF?

**DataPath Framework (DPF)** is NVIDIA's Kubernetes-native platform for managing DPU-accelerated infrastructure in OpenShift. It provides:

- **Automated DPU provisioning**: OS installation, configuration, and updates
- **Network offload orchestration**: OVN-Kubernetes acceleration
- **SR-IOV lifecycle management**: VF creation, assignment, and monitoring
- **Observability**: Monitoring and logging for DPU components

### DPF Component Architecture

```
┌───────────────────────────────────────────────────────────────┐
│              OpenShift Cluster (Control Plane)                │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  DPF Operator (Namespace: nvidia-dpu-platform)       │    │
│  │  ┌────────────────────────────────────────────────┐  │    │
│  │  │  - DPU Provisioning Controller                 │  │    │
│  │  │  - Network Configuration Controller            │  │    │
│  │  │  - SR-IOV Network Operator integration         │  │    │
│  │  │  - OVN-K offload manager                       │  │    │
│  │  └────────────────────────────────────────────────┘  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  DPU Custom Resources                                │    │
│  │  - DPUCluster: Defines DPU cluster configuration    │    │
│  │  - DPUFlavor: DPU hardware profiles                 │    │
│  │  - DPUService: Services running on DPUs             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
└───────────────────────────────────────────────────────────────┘
         │
         │ Manages
         ▼
┌───────────────────────────────────────────────────────────────┐
│                   Worker Nodes with DPUs                      │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Host Side:                                                   │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  - DPF Device Plugin                                 │    │
│  │  - SR-IOV CNI                                        │    │
│  │  - Application Pods with VF interfaces              │    │
│  └──────────────────────────────────────────────────────┘    │
│                    ▲ PCIe ▼                                   │
│  DPU Side:                                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  - DPF Agent (reports status to operator)           │    │
│  │  - OVN-Kubernetes components                         │    │
│  │  - SR-IOV configuration daemon                       │    │
│  │  - Monitoring exporters                              │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### DPF Workflow: VM Deployment with DPU Acceleration

When deploying a VM with DPU-accelerated networking:

```
1. User creates VM with UDN network annotation
   ↓
2. OpenShift scheduler places VM on DPU-enabled worker
   ↓
3. DPF Device Plugin allocates SR-IOV VF from DPU
   ↓
4. kubevirt creates VM with VF passthrough
   ↓
5. OVN-K controller (on DPU) programs network policies
   ↓
6. Hardware accelerators enforce policies at line rate
   ↓
7. VM traffic flows through DPU hardware offload
```

### DPU Provisioning Lifecycle

DPF manages the complete DPU lifecycle:

```
┌─────────────────────────────────────────────────────────┐
│  Phase 1: Discovery                                     │
│  - DPF detects BlueField cards via PCIe                 │
│  - Reads DPU inventory (model, firmware version)        │
└────────────────────┬────────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Phase 2: Provisioning                                  │
│  - Flashes DPU with Rocky Linux/RHEL image              │
│  - Installs OVN-K components                            │
│  - Configures SR-IOV VFs                                │
└────────────────────┬────────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Phase 3: Configuration                                 │
│  - Applies network policies                             │
│  - Sets up OVN SB/NB databases                          │
│  - Enables hardware offload                             │
└────────────────────┬────────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Phase 4: Runtime Management                            │
│  - Monitors DPU health                                  │
│  - Handles updates and patches                          │
│  - Reports status to OpenShift                          │
└─────────────────────────────────────────────────────────┘
```

---

## Network Data Path Architecture

### Packet Flow: Traditional vs. DPU-Accelerated

#### Traditional OpenShift Pod-to-Pod Communication

```
Pod A (Worker 1)
  │
  ▼
OVS kernel module (software)
  │
  ▼ (encapsulate in Geneve tunnel)
Physical NIC
  │
  ▼
Network Switch
  │
  ▼
Physical NIC (Worker 2)
  │
  ▼
OVS kernel module (software)
  │
  ▼ (decapsulate Geneve)
Pod B (Worker 2)

Total CPU cost: ~4-6 cores across both workers
Latency: ~50-100µs software processing overhead
```

#### DPU-Accelerated Pod-to-Pod Communication

```
Pod A (Worker 1 - with DPU)
  │
  ▼ SR-IOV VF (PCIe passthrough)
DPU Hardware Pipeline
  │
  ├─ OVS flow lookup (in ASIC, <1µs)
  ├─ Geneve encapsulation (hardware)
  ├─ ACL enforcement (hardware)
  └─ TX to physical port
  │
  ▼
Network Switch
  │
  ▼
DPU Hardware Pipeline (Worker 2)
  │
  ├─ Geneve decapsulation (hardware)
  ├─ OVS flow lookup (in ASIC)
  └─ Forward to SR-IOV VF
  │
  ▼ SR-IOV VF (PCIe passthrough)
Pod B (Worker 2 - with DPU)

Total CPU cost: ~0 host cores (all on DPU ARM cores)
Latency: ~10-20µs (mostly PCIe and network wire time)
Throughput: Line rate (200Gb/s sustained)
```

### OVN-Kubernetes Offload Architecture

DPF integrates with OVN-K by offloading the data plane:

```
┌────────────────────────────────────────────────────────┐
│         OVN-Kubernetes Control Plane (on DPU)          │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ovn-northbound DB                                     │
│  ├─ Logical switches (pod networks)                   │
│  ├─ Logical routers (inter-subnet routing)            │
│  └─ ACLs (NetworkPolicy enforcement)                  │
│                                                        │
│              ▼ Translated by ovn-northd ▼             │
│                                                        │
│  ovn-southbound DB                                     │
│  ├─ Logical flows (match-action rules)                │
│  └─ Port bindings (pod → VF mappings)                 │
│                                                        │
│              ▼ Programmed by ovn-controller ▼         │
│                                                        │
│  Open vSwitch (running on DPU)                         │
│  ├─ Flow tables (installed in kernel)                 │
│  └─ tc-offload enabled                                │
│                                                        │
│              ▼ Offloaded to hardware ▼                │
│                                                        │
│  BlueField-3 eSwitch and Packet Processing Engine      │
│  ├─ Hardware flow cache (millions of flows)           │
│  ├─ Connection tracking (stateful firewall)           │
│  ├─ Geneve encap/decap (tunnel overhead removed)      │
│  └─ ACL enforcement (zero CPU cost)                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### SR-IOV Virtual Function Model

DPUs expose network interfaces to VMs/pods via SR-IOV:

```
┌────────────────────────────────────────────────────────┐
│              DPU Physical Function (PF)                │
│              - Represents the physical port            │
└───────┬────────────────────────────────────────────────┘
        │
        ├─ Creates Virtual Functions (VFs)
        │
        ├──┬─────────────────────────────────────────────┐
        │  │  VF 0: Assigned to Pod A on Host            │
        │  │  - MAC: aa:bb:cc:00:00:01                   │
        │  │  - VLAN: 100                                │
        │  └─────────────────────────────────────────────┘
        │
        ├──┬─────────────────────────────────────────────┐
        │  │  VF 1: Assigned to Pod B on Host            │
        │  │  - MAC: aa:bb:cc:00:00:02                   │
        │  │  - VLAN: 100                                │
        │  └─────────────────────────────────────────────┘
        │
        └──┬─────────────────────────────────────────────┐
           │  VF N: Available for assignment             │
           │  - Resource pool managed by SR-IOV operator │
           └─────────────────────────────────────────────┘

Each VF:
- Appears as a dedicated NIC to the VM/pod
- PCIe passthrough for near-native performance
- Isolated from other VFs (hardware-enforced)
- Managed by DPU (host cannot tamper with config)
```

---

## Summary: Why DPU + OpenShift?

| Aspect | Traditional OpenShift | DPU-Accelerated OpenShift |
|--------|----------------------|---------------------------|
| **CPU Overhead** | 20-30% for networking | <1% (offloaded to DPU) |
| **Network Throughput** | Software-limited (~50Gb/s) | Line-rate (200-400Gb/s) |
| **Latency** | 50-100µs software overhead | 10-20µs (mostly hardware) |
| **Security Posture** | Software isolation only | Physical isolation (DPU) |
| **Connection Scale** | Limited by CPU/memory | Millions of flows in HW |
| **Tenant Isolation** | cgroups + namespaces | Physical separation (DPU controls network) |

### When to Use DPUs in OpenShift

**Ideal Use Cases**:
- **High-throughput workloads**: Network-intensive applications (NFV, 5G, video processing)
- **Multi-tenant environments**: Strong isolation requirements (telco, financial services)
- **Security-critical deployments**: Zero-trust architectures with hardware-enforced policies
- **Edge computing**: Resource-constrained nodes where CPU efficiency matters

**Not Ideal For**:
- **Small clusters**: Overhead of managing DPU infrastructure
- **Compute-bound workloads**: Where networking is not the bottleneck
- **Development/test environments**: Unless specifically testing DPU features

---

## Next Steps

- [Basic Installation Guide](./basic-instalation.md) - Manual CNV/UDN setup
- [OpenShift DPF Automation](./openshift-dpf-readme.md) - Automated DPF deployment
- **[Coming Soon]** DPU Troubleshooting and Debugging
- **[Coming Soon]** Performance Tuning and Benchmarking
