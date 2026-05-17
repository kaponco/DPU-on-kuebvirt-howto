# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This repository serves as a knowledge base and documentation collection for **DPU (Data Processing Unit) management** and **DPF (DataPath Framework)** in OpenShift environments. The focus is on collecting useful information, procedures, and artifacts related to:

- DPU hardware acceleration and management
- DataPath Framework (DPF) integration with OpenShift
- CNV (Container-native Virtualization) on DPU-enabled clusters
- OVN-Kubernetes User Defined Networks (UDN) with DPU acceleration
- SR-IOV configuration for hardware-accelerated networking

This is a **documentation-first repository** where materials will be added incrementally.

## Repository Structure

```
dpu-misc/
├── docs/                      # Documentation and procedures
│   ├── architecture.md        # DPU/DPF architecture deep-dive
│   ├── basic-instalation.md   # Manual CNV cluster setup with OVN-K UDN
│   ├── openshift-dpf-readme.md # Automated DPF deployment framework
│   └── gemini-instructions.md # SNO + DPU worker integration guide
├── custom-resources/          # Kubernetes YAML manifests
│   ├── user-defined-network.yaml     # UDN configuration
│   ├── virtual-machine.yaml          # Standard VM with UDN
│   └── virtual-machine-metal.yaml    # Metal worker VM with nodeSelector
├── tasks-lists/               # Operational checklists
│   ├── rosa-cli-setup.md             # ROSA CLI installation guide
│   ├── rosa-metal-worker-tasks.md    # ROSA worker provisioning tasks
│   └── vm-tasks.md                   # VM operation checklists
└── CLAUDE.md                  # This file
```

## Key Concepts

**DPU (Data Processing Unit)**: Hardware accelerator that offloads networking, storage, and security tasks from the CPU. Typically NVIDIA BlueField-3 cards with ARM processors, dedicated memory, and hardware accelerators for line-rate packet processing.

**DPF (DataPath Framework)**: NVIDIA's Kubernetes-native platform for managing DPU-accelerated infrastructure in OpenShift. Provides automated DPU provisioning, network offload orchestration, and SR-IOV lifecycle management.

**UDN (User Defined Networks)**: OVN-Kubernetes feature that allows VMs and pods to use custom network configurations beyond the cluster's default network.

**SR-IOV (Single Root I/O Virtualization)**: Technology that allows a PCIe device (like a DPU) to appear as multiple separate physical devices (Virtual Functions), enabling direct hardware passthrough to VMs with near-native performance.

**CNV (Container-native Virtualization)**: OpenShift Virtualization operator (KubeVirt) that enables running VMs alongside containers in OpenShift.

**SNO (Single Node OpenShift)**: Minimal OpenShift deployment on a single server, commonly used for edge computing or development environments.

**ROSA (Red Hat OpenShift Service on AWS)**: Managed OpenShift service running on AWS infrastructure.

**BFB (BlueField Bootstream)**: Operating system image file (.bfb) that provisions the DPU's ARM processor with Rocky Linux/RHEL and required software components.

## Current Content

### Documentation (`docs/`)

#### Architecture Guide (`architecture.md`)
Comprehensive architectural overview of DPU/DPF integration with OpenShift:
- DPU hardware fundamentals (NVIDIA BlueField-3 architecture)
- Network data path architecture (traditional vs. DPU-accelerated)
- OVN-Kubernetes offload mechanisms
- SR-IOV Virtual Function model
- Security and performance benefits

#### Basic Installation Guide (`basic-instalation.md`)
Manual setup workflow for CNV clusters with DPU acceleration:
- **Phase 1**: CNV Operator installation and HyperConverged CR deployment
- **Phase 2**: Standard OVN-K UDN setup and VM deployment
- **Phase 3**: DPF-enabled cluster with DPU workers and accelerated networking

#### OpenShift DPF Automation (`openshift-dpf-readme.md`)
Documentation for the automated DPF deployment framework ([rh-ecosystem-edge/openshift-dpf](https://github.com/rh-ecosystem-edge/openshift-dpf)):
- One-command deployment (`make all`)
- Environment configuration and credential setup
- Worker node provisioning with Bare Metal Operator
- Supported versions: OpenShift 4.20, DPF v25.7+

#### SNO + DPU Worker Integration (`gemini-instructions.md`)
Step-by-step guide for adding DPU workers to Single Node OpenShift:
- OpenShift Virtualization operator installation
- DPF operator and infrastructure prerequisites
- BlueField-3 provisioning via BFB image
- SR-IOV configuration for KubeVirt VMs
- VM deployment targeting DPU worker nodes

### Custom Resources (`custom-resources/`)

Example Kubernetes manifests for deploying VMs with various networking configurations:
- `user-defined-network.yaml` - OVN-K UDN configuration
- `virtual-machine.yaml` - Standard VM with UDN networking
- `virtual-machine-metal.yaml` - VM with nodeSelector for metal worker nodes

### Task Lists (`tasks-lists/`)

Operational checklists for common procedures:
- `rosa-cli-setup.md` - Complete ROSA CLI installation and authentication workflow
- `rosa-metal-worker-tasks.md` - Tasks for provisioning ROSA worker nodes
- `vm-tasks.md` - VM operation and management checklists

## Working with This Repository

### Applying Custom Resources
To deploy VMs or networks using the provided YAML manifests:
```bash
oc apply -f custom-resources/user-defined-network.yaml
oc apply -f custom-resources/virtual-machine.yaml
```

### Using Task Lists
Task lists in `tasks-lists/` are Markdown checklists for operational procedures. Use them as step-by-step guides for setup and configuration tasks.

### Documentation Standards

When adding new documentation to this repository:

- **Documentation**: Place in `docs/` directory with descriptive filenames
- **YAML manifests**: Place in `custom-resources/` with resource type in filename
- **Checklists**: Place in `tasks-lists/` for operational procedures
- Include step-by-step procedures with validation criteria
- Document both standard and DPU-accelerated configurations where applicable
- Reference official Red Hat/OpenShift documentation for prerequisites

## Future Topics

Potential additions to this knowledge base:
- DPF troubleshooting and debugging procedures
- Performance benchmarking results (with/without DPU acceleration)
- Network topology diagrams for production deployments
- Migration guides (standard → DPU-accelerated clusters)
