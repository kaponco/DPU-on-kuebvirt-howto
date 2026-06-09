## **DPF Installation On OpenShift User Manual** 

## **(Technical Preview)**

## 

Revision: 1.0  
Date: 05.01.2025

## Table of Contents:

**References**

**Chapter 1: Architecture Overview**

* 1.1 The Topology  
* 1.2 Example Lab Topology  
* 1.3 Architectural Benefits  
  * 1.3.1 Operational Simplicity and Integrated Management  
  * Accelerated Infrastructure  
  *  Kubernetes Native Orchestration  
* 1.4 Component Placement  
  * Management Components  
* 1.5 Deployment Flow Overview

**Chapter 2: Prerequisites**

* 2.1 Hardware Requirements  
  * 2.1.1 Workstation  
  * 2.1.2 Control Plane Nodes  
  * 2.1.3 Worker Nodes  
  * 2.1.4 NVIDIA BlueField-3 DPUs  
* 2.2 Network Infrastructure  
* 2.3 Software Requirements  
* 2.4 DNS, Access, Storage, and Security  
* 2.5 Additional Guidance

**Chapter 3: Management Cluster Setup**

* 3.1 Overview  
* 3.2 Installation Method  
  * 3.2.1 Cluster Installation  
  * 3.2.2 Cluster State Verification

**Chapter 4: Environment Setup**

* 4.1 Worker Node Configuration  
  * 4.1.1 Worker MachineConfig Resource  
  * 4.1.2 MachineConfig Decoded Scripts  
* 4.2 Cluster Configuration  
  * 4.2.1 Custom Cluster Feature \- MutatingAdmissionPolicy  
  * Control Plane Node Patching  
  * Create DPF Namespace  
* 4.3 Operators Installation  
  * 4.3.1 Cert Manager Operator  
  * 4.3.2 MetalLB Operator  
  * 4.3.3 GitOps Operator  
  * 4.3.4 NVIDIA Maintenance Operator  
* 4.4 Operator Configuration  
  * 4.4.1 Configure NFD Operator  
  * 4.4.2 Configure MetalLB Operator  
  * 4.4.3 Create GitOps Operator  
  * 4.4.4 Configure Cluster Network Operator  
* 4.5 Custom SR-IOV Device Plugin Installation

**Chapter 5: Hosted Cluster for DPU Control-Plane**

* 5.1 Define Environment Variables  
* 5.2 Create a DPU Hosted Cluster  
* 5.3 Hosted Cluster creation verification  
* 5.4 Extract the Hosted Cluster kubeconfig  
* 5.5 Adding DPU IPAM Controller Manager to the Hosted Cluster namespace

**Chapter 6: DPF Operator Installation and Configuration**

* 6.1 Environment Variables  
* 6.2 Create DPU Image Storage Objects  
* 6.3 Install DPF Operator  
* 6.4 Generate BF.cfg ConfigMap  
* 6.5 Create DPF Resources  
  * 6.5.1 DPFOperatorConfig  
  * 6.5.2 DPUCluster  
  * 6.5.3 DPUFlavor  
  * 6.5.4 DPUDeployment  
  * 6.5.5 BFB  
* 6.6 Create DPU Services Resources  
  * 6.6.1 HBN DPU Service  
  * 6.6.2 OVN-Kubernetes DPU Service  
  * 6.6.3 OVN-Kubernetes DPUServiceCredentialRequest  
  * 6.6.4 DPUServiceInterface Physical  
  * 6.6.5 DPUServiceInterface OVN-Kubernetes  
  * 6.6.6 DPUServiceNADs  
  * 6.6.7 DPUServiceIPAM  
* 6.7 Verify DPU Services Resources

**Chapter 7: OVN-Kubernetes CNI Adjustments**

* 7.1 Enable OVN-Kubernetes Resource Injector  
* 7.2 Enable OVN-Kubernetes DPU-Host Mode

**Chapter 8: Adding a Worker Node**

* 8.1 Prerequisites   
* 8.2 Add Worker Nodes  
  * 8.3.1 Option 1: Adding Worker Nodes Using Assisted Installer  
  * 8.3.2 Option 2: Adding Worker Nodes Using Baremetal Operator  
* 8.3 Approve Worker Nodes certificate requests  
* 8.4 Verify DPU Provisioning is started automatically  
* 8.5 Authorization Configuration for the Hosted DPU Cluster Components  
* 8.6 Approve DPU Nodes Certificate Requests to Complete Provisioning  
* 8.7 Verify Cluster Readiness

**Chapter 9: Traffic Validation Test**

* 9.1 Create Traffic Test Pods/Services  
* 9.2 Run Traffic Validation Tests


**Chapter 10: Basic Troubleshooting**

* 10.1 DPU Provisioning  
* 10.2 DPU Object State  
* 10.3 Management cluster nodes state

**Chapter 11: Appendix**

* 11.1 Unsupported OVN Kubernetes Features

## 

## References

* [OCP Installation With Assisted Installer](https://docs.redhat.com/en/documentation/assisted_installer_for_openshift_container_platform/2025/html/installing_openshift_container_platform_with_the_assisted_installer/index)  
* [DOCA Platform Framework (DPF) GitHub](https://github.com/NVIDIA/doca-platform/tree/public-release-v25.7)  
* [DOCA Platform Framework (DPF) Documentation v.25.7.1](https://docs.nvidia.com/networking/display/dpf2571)  
* [RDG for DPF Host Trusted with OVN-Kubernetes and HBN Services](https://docs.nvidia.com/networking/display/public/sol/rdg-for-dpf-host-trusted-with-ovn-kubernetes-and-hbn-services)  
* [OpenShift DPF Automation](https://github.com/rh-ecosystem-edge/openshift-dpf/tree/main) 

##  

## **Chapter 1:** Architecture Overview

This section outlines the architecture and benefits of deploying NVIDIA BlueField-3 Data Processing Units (DPUs) managed by the DOCA Platform Framework (DPF) within a Red Hat OpenShift environment. This deployment model fundamentally shifts data center architecture by decoupling infrastructure services from tenant workloads.

The NVIDIA DOCA Platform Framework (DPF) in Host Trusted mode revolutionizes infrastructure efficiency by transforming BlueField DPUs into integrated host accelerators, a capability that reaches its full potential when paired with Red Hat OpenShift. This powerful combination enables a "single pane of glass" where administrators can orchestrate both workloads and DPU-accelerated infrastructure using standard OpenShift APIs and Custom Resource Definitions (CRDs). 

By seamlessly offloading critical OpenShift networking functions—such as OVN-Kubernetes \- to the DPU, the architecture frees up substantial host CPU resources for tenant applications while delivering wire-speed performance. Furthermore, DPF streamlines DevOps workflows through automated lifecycle management, allowing infrastructure admins to provision, configure, and update fleets of DPUs directly from the OpenShift, ensuring a cloud-native, consistent operational model without the complexity of out-of-band management.

### 

#### 1.1 The Topology

The architecture consists of two distinct OpenShift clusters operating on the same physical server hardware:

1. **OpenShift Management Cluster:**  
   * **Location:** Runs on the server's main x86 CPU cores.  
   * **Role:** Hosts the actual business logic, such as AI workloads or enterprise applications.  
   * **Components:** Red Hat OpenShift worker nodes containing Application Pods.  
   * **Networking:** The Application Pods on the worker node rely on the DPU accelerated network; the host OS sees accelerated interfaces (e.g., SR-IOV VFs or SFs) but does not manage the network control plane.  
   * **Role:** It serves as the primary administrative interface and the "Host Cluster" that provisions and manages both the fleet of DPUs and the users workloads. In a Host Trusted deployment, the worker nodes in this cluster are the physical servers that house the BlueField DPUs. It is responsible for the lifecycle management of the DPU hardware and the orchestration system running on the DPUs.  
   * **Deployment:**  
     1. User Workloads: The actual AI or Telco applications running on the x86 host processors reside here  
     2. DPF Operator: The core operator that installs and configures the DPF system.  
     3. Control Plane Components: It hosts the DPU Cluster Control Plane (typically using Kamaji to run the control plane as pods) which manages the separate cluster running on the DPUs,.  
     4. Controllers: Various controllers run here, including the Provisioning Controllers (BFB, DPUSet, DPU, DPUCluster), DPUService Controllers (to manage apps on the DPU), and DPUServiceChain Controllers.  
   * **Usage:**  
     1. **Run users workloads**      
     2. **Single Pane of Glass:** Users interact with this cluster to define the desired state of the infrastructure using Kubernetes Custom Resource Definitions (CRDs) like `DPUSet`, `DPUDeployment`, and `DPUService`,.  
     3.     **Provisioning:** It drives the discovery of DPUs, flashes the BlueField Bootstream (BFB) images, and configures the host-to-DPU networking,.  
     4.     **Orchestration:** It coordinates the deployment of services and network flows down to the DPU Cluster  
2. **OpenShift DPU Hosted Cluster:**  
   * **Role:** The OpenShift DPU Hosted Cluster serves as a dedicated, secondary Kubernetes control plane specifically for managing the fleet of NVIDIA BlueField DPUs. In this architecture, the DPUs themselves function as the worker nodes of this hosted cluster, distinct from the bare-metal hosts they are physically attached to. This separates the management domain of the infrastructure acceleration hardware from the tenant workloads running on the main host servers.  
   * **Deployment:** This cluster hosts the infrastructure software and DOCA services required to operate the DPU. Key components deployed on it include:  
     1. **Networking Stack:** OVN-Kubernetes (running in DPU mode to offload flows).   
     2.  **DOCA Services:** Specialized applications such as Host-Based Networking (HBN) for BGP routing, DOCA Telemetry Service (DTS) for monitoring, and Firefly for time synchronization.  
     3. **System Components:** NVIDIA IPAM, Multus, and SR-IOV Device Plugins to manage the DPU's hardware resources.  
   * **Usage:** It is used to orchestrate and lifecycle-manage the software running on the DPU independently of the host. Its primary functions are to **offload** critical networking (like OVS processing) from the host CPU to the DPU, **isolate** infrastructure services for better security, and perform **Service Function Chaining** to route traffic through specific network functions (e.g., firewalls or encryption) directly on the hardware

#### 1.2 Example Lab Topology

The following diagram illustrates the physical connectivity for a reference lab environment. It serves as a baseline example to demonstrate the core components and their interactions.![][image1]

#### 1.3 Architectural Benefits

##### 1.3.1 Operational Simplicity and Integrated Management

**Host-Based Management:** In a Host Trusted deployment, the DPU is managed from the host. This model allows cloud operators to manage BlueField-bound services directly from their standard OpenShift control plane, providing a "single pane of glass" view for both OpenShift tenant nodes and DPUs.

**Streamlined Deployment:** DPF in this mode facilitates the automated discovery and provisioning of DPUs. The DPF Operator detects worker nodes, creates DPU objects, and deploys the DOCA Management Service (DMS) to flash the BlueField Boot (BFB) firmware and configure networking between the host and DPU.

**Reduced Complexity:** This mode eliminates the complexity of low-level device configuration by utilizing standard Kubernetes APIs and workflows for the DPU lifecycle.

##### 1.3.2 Accelerated Infrastructure

**Resource Offloading:** In Host Trusted mode, the architecture allows for the offloading of infrastructure services—such as networking, storage, and security—from the host CPU to the DPU. This frees up host CPU resources for applications and boosts infrastructure performance. The upcoming release is focused on networking services.

**Hardware Acceleration:** The framework enables the creation of hardware-accelerated, software-defined solutions that manage data center traffic and storage through dedicated ports on the BlueField DPU.

##### 1.3.3 Kubernetes Native Orchestration

**Seamless Integration:** DPF extends Kubernetes control plane functionality to the DPUs, enabling administrators to deploy and orchestrate NVIDIA DOCA services and third-party applications directly on the BlueField DPU using familiar Kubernetes constructs.

**Automated Lifecycle Management:** The architecture supports automated rolling updates, scaling, and rollbacks for services, allowing administrators to manage changes efficiently without disrupting ongoing operations.

It is important to note that in Host Trusted mode, the host is part of the trusted domain.

#### 1.4 Component Placement

The various software components and operators are strategically placed across the two clusters to maintain this separation.

##### 1.4.1 Management Components

The following operators and services run within the management cluster:

* **NVIDIA DPF Operator:** The core operator that manages DPU services and configurations within the hosted cluster, including DPU provisioning, networking acceleration, and DOCA service orchestration  
* **MultiCluster Engine (MCE) & HyperShift:** Provide the control plane and management framework for the DPU-hosted cluster.  
* **Node Feature Discovery (NFD) Operator:** Discovers and labels hardware features on the nodes, including the presence of DPUs.  
* **MetalLB Operator:** Provides load-balancing services for the management cluster.  
* **GitOps Operator:** Facilitates ArgoCD based deployment of applications and configurations.  
* **Cert-Manager Operator:** Automates the management, issuance, and renewal of TLS certificates within the cluster.  
* **NVIDIA Maintenance Operator:** Assists in performing maintenance tasks and gracefully draining DPU worker nodes.  
* **LVM Storage Operator:** Provides persistent ReadWriteMany (RWX) storage required for various components, such as the etcd database of the hosted cluster.  
* **SR-IOV Device Plugin (Custom):** A standalone SR-IOV device plugin is used instead of the full SR-IOV Network Operator. Features an enhanced device discovery mechanism that ensures DPU PF/VF devices are fully initialized before registering resources with Kubernetes.  
* **Baremetal Operator:** Provisions and adds worker nodes with DPUs to the management cluster 

#### 1.5 Deployment Flow Overview

The end-to-end deployment process follows these high-level steps:

1. **Management Cluster Setup:** A standard OCP cluster is installed and configured on the x86 servers (control-plane nodes only).  
2. **Management Cluster Configuration:** Nodes and cluster level configuration, followed by operators installation and configuration on the management cluster.  
3. **Hosted Cluster Creation:** A hosted DPU cluster is created explicitly using HyperShift Operator. A kubeconfig secret for the hosted cluster is created to be used by DPF using the DPUCluster resource.  
4. **DPF Installation:**  DPF Operator, controllers and related components are deployed on the management cluster.  
5. **Worker Nodes Scale-Out and DPU Provisioning:** The moment worker nodes with DPUs are added to the cluster, the DPF Operator flashes the DPUs with a Red Hat Enterprise Linux CoreOS (RHCOS) image and configures them to join the hosted cluster as worker nodes.  
6. **Service Deployment:** Once the DPU-hosted cluster is operational, data plane DPU services and chains are automatically deployed by DPF.  
7. **Workload Deployment:** Test pods utilizing the DPU services and chains are created and can communicate over DPU-accelerated network.

## **Chapter 2:** Prerequisites

This chapter outlines the prerequisites for deploying the DPF Operator. Fulfilling these hardware, software, and environmental requirements is essential for setting up the architecture described in Chapter 1\.

#### 2.1 Hardware Requirements

##### 2.1.1 Workstation

A workstation with the following command-line interface (CLI) tools installed:

* OpenShift CLI (oc) \- Download from the [OpenShift mirror](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/)  
* Helm CLI (helm) \- See the [Helm installation guide](https://helm.sh/docs/intro/install/%20)  
* HyperShift CLI 

```shell
# Instructions to install HyperShift CLI
git clone https://github.com/openshift/hypershift.git
cd hypershift
make build
sudo install -m 0755 bin/hypershift /usr/local/bin/hypershift
# Verify the installation 
hypershift --help
```

* A modern internet browser for accessing the OpenShift web console.

##### 2.1.2 Control Plane Nodes (3 Nodes)

These nodes form the control plane of the management cluster.

* **Form Factor:** Can be virtual machines or physical servers.  
* **Memory:** 60GB Memory  
* **CPU:** 16 vCPUs (Intel/AMD x86\_64)  
* **Storage:**   
  * Disk of 120 GB NVMe/SSD storage  
  * An additional disk of 80GB   
    * Needed for OpenShift LVM Storage, which is configured in a later step.  
* **Networking:** 1x 1GbE network interface.  
* **Note:** DPUs must not be installed

##### 2.1.3 Worker Nodes (2 Nodes)

The physical x86 servers which host the NVIDIA BlueField-3 DPUs and act as worker nodes for the management cluster. For the purpose of this document we used two physical servers.

* **Memory:** 256GB RAM   
* **CPU:** 16 cores (Intel/AMD x86\_64)  
* **Storage:**   
  * A minimum of 500 GB NVMe/SSD storage for the base operating system.  
* **DPU Slot:** Each server can have multiple DPUs but only one  NVIDIA BlueField-3 DPU can be provisioned.  
* **BIOS Settings:**  
  * SR-IOV must be enabled.  
  * In-Band Manageability Interface must be enabled.  
* **Note on Network Configuration:** As part of the installation process, a Linux bridge named “br-dpu” will be automatically created on the worker node's physical management port via a MachineConfig Custom Resource to facilitate control-plane traffic from the DPU through the host server.

##### 2.1.4 NVIDIA BlueField-3 DPUs (One per Worker Node)

* **Model:** BlueField-3  
* **Memory:** 32GB  
  * Dual-port DPUs with 32GB requires external power connection to the x86 server  
* **Networking:** Dual 200GbE ports per DPU. Both ports must be connected to the high-speed switch for ECMP routing.  
* **Management:** The out-of-band management port is not used in this configuration.  
* **Operating System:** The DPUs will be provisioned with a BlueField Bootstream File (BFB) containing a Red Hat Enterprise Linux CoreOS (RHCOS) base image, DOCA runtime, drivers, and firmware.

#### 2.2 Network Infrastructure

* **Switches:**  
  * **Management Switch:** Provides 1GbE connectivity for the control plane and worker node management interfaces.  
  * **High-Speed Switch:** An NVIDIA SN3700 or similar switch providing 2x 200GbE connectivity per DPU.  
* **Connectivity:**  
  * All nodes have full internet access \- both from the host out-of-band and DPU high speed interfaces.  
  * The management network and the high-speed DPU (VTEP CIDR) network must be routable to each other.  
  * A Virtual IP (VIP) from the management subnet must be reserved for Hosted DPU Cluster’s Control-Plane services.  
    * The Virtual IP Address used for DPU Cluster must have a DNS A Record.  
* **MTU Configuration:**  
  * For optimized performance, all environment components—platform, nodes, interfaces, and network hardware must support jumbo frames.  
    * This configuration is set during deployment and cannot be altered later.  
  * When using VMs for the OCP cluster, set the MTU on the bridge of the hypervisor that the VMs are using.  
* **Reference:** For detailed network topology, interface names, and bridge configurations, refer to the [RDG for DPF with OVN-Kubernetes and HBN Services](https://docs.nvidia.com/networking/display/public/sol/rdg-for-dpf-host-trusted-with-ovn-kubernetes-and-hbn-services), specifically the "Solution Design" and "Node and Switch Definitions" sections.

#### 

#### 2.3 Software Requirements

* OpenShift Version: Red Hat OpenShift Container Platform 4.20.4  
* OpenShift CLI (oc) Version: 4.20   
* HyperShift Hosted OpenShift Cluster Version: 4.20.4  
* NVIDIA DPF (Doca Platform Framework) Operator Version: 25.07.1   
* RHCOS BFB Version: 4.20.4 (DOCA 3.1)

#### 2.4 DNS, Access, Storage, and Security

* **Cluster Privileges:** `cluster-admin` privileges are required for the management cluster.  
* **NFS Storage:** NFS share (or equivalent ReadWriteMany storage) with a dedicated folder and Read/Write permissions from the Management Network is required to store the downloaded BFB image.  
* **DPU UEFI Secure Boot:** Secure Boot must be disabled in the DPU UEFI.  
  * For instructions please use the official [NVIDIA guide](https://docs.nvidia.com/networking/display/bluefielddpuosv396/uefi+secure+boot#src-138805874_UEFISecureBoot-DisablingUEFISecureBoot) for disabling Secure Boot

#### 2.5 Additional Guidance

* **NVIDIA Documentation:** For foundational hardware and software details regarding the BlueField-3 DPU, it is also recommended to consult the official NVIDIA user guides.  
  * [Get Started with DPF Host Trusted](https://github.com/NVIDIA/doca-platform/blob/public-release-v25.7/docs/public/getting-started/dpf-host-trusted.md)  
  * [DPF OVN Kubernetes with Host Based Networking User Guide](https://github.com/NVIDIA/doca-platform/blob/public-release-v25.7/docs/public/user-guides/host-trusted/use-cases/hbn-ovnk/README.md)  
  * [RDG for DPF with OVN-Kubernetes and HBN Services](https://docs.nvidia.com/networking/display/public/sol/rdg-for-dpf-host-trusted-with-ovn-kubernetes-and-hbn-services)

## 

## **Chapter 3:** Management Cluster Setup

#### 3.1 Overview 

The management cluster is a standard OpenShift v4.20.4 cluster installed using Red Hat’s Assisted Installer.  
This cluster will host the DPF operators and HyperShift control plane for managing the hosted cluster on DPUs.

#### 3.2 Cluster Installation Using Assisted Installer 

##### 3.2.1 Cluster Installation

* Create a cluster (control-plane only) at [https://console.redhat.com/openshift/create](https://console.redhat.com/openshift/create)  
  * Select “Datacenter” type \-\> Assisted Installer

* Configure Jumbo MTU for each one of the control plane nodes  
  * Under “Hosts' network configuration” in Assisted Installer wizard, select “Static IP, bridges, and bonds”  
  * Set the “Static network configurations” section per node according to the following template (use the relevant MAC address and interface-name for each node):

```shell
interfaces:
  - ipv4:
      dhcp: true
      enabled: true
    mac-address: <xx:xx:xx:xx:xx:xx>
    mtu: 9000
    name: <interface-name>
    state: up
    type: ethernet
```

**Notes:**

1. Alternatively, Jumbo MTU allocation can be configured on the DHCP server allocating IPs to the control plane nodes  
2. In case VMs are used for control-plane nodes, Jumbo MTU should be set on the bridge of the hypervisor used by the VMs.  
3. Make sure the switch ports that connect the cluster’s control-plane nodes are set to handle jumbo frames.  
* Select the following operators to install with the cluster:  
  * Storage  
    * Logical Volume Manager Storage    
  * Platform Operations & Lifecycle  
    * MultiCluster engine (Used to deploy Hosted Control Plane Operator)  
  * Scheduling  
    * Node Feature Discovery  
* Add hosts to the cluster  
  * Control plane nodes only  
* After the installation is finished \-\> download the KUBECONFIG and save it as a file. i.e **mgmt-kubeconfig**

##### 3.2.2 Cluster State Verification 

* Run the following commands to verify the cluster is up:

```shell
# Set KUBECONFIG environment variable
$ export KUBECONFIG=mgmt-kubeconfig
# Verify cluster health
# All nodes should be Ready
$ oc get nodes
NAME STATUS ROLES AGE VERSION
master-0 Ready control-plane,worker 10m v1.27.8
master-1 Ready control-plane,worker 10m v1.27.8
master-2 Ready control-plane,worker 10m v1.27.8
```

## 

## **Chapter 4:** Environment Setup

#### 4.1 Worker Nodes Pre-Configuration

In this step, a MachineConfig resource for worker nodes is created. Note that at this stage of the installation, there are no worker nodes in the management cluster yet. The MachineConfig will be stored in the cluster's configuration and will be automatically applied by the Machine Config Operator (MCO) when worker nodes are added to the cluster. Once added, the MCO will apply and reboot the worker nodes to ensure all configurations take effect.

##### 4.1.1 Worker MachineConfig Resource 

The described MachineConfig resource performs several configuration tasks required by DPF:

* **Bridge Configuration**: Creates a br-dpu bridge interface that enables communication between the DPU and the hosted cluster control plane running on the management cluster. For more information refer to [DPF Operator Prerequisites](https://github.com/NVIDIA/doca-platform/blob/public-release-v25.7/docs/public/user-guides/host-trusted/prerequisites/host-network-configuration-prerequisite.md).  
  Key Features:  
* Bridge port configuration \- Enslaves worker node management interface to br-dpu bridge  
* Automatic interface detection \- No manual configuration required  
* Network readiness validation \- Waits for proper network setup  
* Idempotent operation \- Safe to run multiple times  
* Jumbo Frame MTU \- Configures 9000 MTU for optimal performance  
* **OVS Service Management**: Disables OpenShift's default OVS services on x86 worker nodes. This is required for OVN-Kubernetes DPU Host mode operation, where networking functions are offloaded to the DPU rather than running on the host CPU.  
* **IP Rules Configuration:** Sets routing rules required for pod-to-host control-plane traffic  
    
1. Create The MachineConfig Resource: 

```shell
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
 labels:
   machineconfiguration.openshift.io/role: worker
 name: worker-dpu-configuration
spec:
 config:
   ignition:
     version: 3.2.0
   storage:
     files:
       - contents:
           source: data:text/plain;charset=utf-8;base64,IyEvYmluL2Jhc2gKCkRFRkFVTFRfTVRVPTE1MDAKTk9ERVNfTVRVPSR7MTotJERFRkFVTFRfTVRVfQoKIyBXYWl0IGluZGVmaW5pdGVseSBmb3IgYW4gaW50ZXJmYWNlIHRvIGdldCBhbiBJUCBhZGRyZXNzIGFuZCBkZWZhdWx0IHJvdXRlCldBSVRfSU5URVJWQUw9NSAgIyBDaGVjayBpbnRlcnZhbCBpbiBzZWNvbmRzCkFUVEVNUFQ9MQoKZWNobyAiV2FpdGluZyBmb3IgbmV0d29yayBjb25maWd1cmF0aW9uLi4uIgp3aGlsZSB0cnVlOyBkbwogICAgIyBDaGVjayBpZiB3ZSBoYXZlIGFuIGludGVyZmFjZSB3aXRoIGEgZGVmYXVsdCByb3V0ZQogICAgREVGQVVMVF9JTlRFUkZBQ0U9JChpcCByb3V0ZSB8IGdyZXAgZGVmYXVsdCB8IGF3ayAne3ByaW50ICQ1fScgfCBoZWFkIC1uIDEpCgogICAgaWYgWyAtbiAiJERFRkFVTFRfSU5URVJGQUNFIiBdOyB0aGVuCiAgICAgICAgIyBDaGVjayBpZiB0aGF0IGludGVyZmFjZSBoYXMgYW4gSVAgYWRkcmVzcwogICAgICAgIElQX0FERFI9JChpcCBhZGRyIHNob3cgZGV2ICIkREVGQVVMVF9JTlRFUkZBQ0UiIHwgZ3JlcCAtdyAiaW5ldCIgfCBhd2sgJ3twcmludCAkMn0nKQoKICAgICAgICBpZiBbIC1uICIkSVBfQUREUiIgXTsgdGhlbgogICAgICAgICAgICBlY2hvICJOZXR3b3JrIGlzIHJlYWR5LiBJbnRlcmZhY2UgJERFRkFVTFRfSU5URVJGQUNFIGhhcyBJUCBhZGRyZXNzICRJUF9BRERSIgogICAgICAgICAgICBicmVhawogICAgICAgIGZpCiAgICBmaQoKICAgIGVjaG8gIldhaXRpbmcgZm9yIG5ldHdvcmsgY29uZmlndXJhdGlvbi4uLiAoQXR0ZW1wdCAkQVRURU1QVCkiCiAgICBzbGVlcCAkV0FJVF9JTlRFUlZBTAogICAgQVRURU1QVD0kKChBVFRFTVBUICsgMSkpCmRvbmUKCiMgQ2hlY2sgaWYgdGhlIGJyaWRnZSBhbHJlYWR5IGV4aXN0cwppZiBpcCBsaW5rIHNob3cgYnItZHB1ICY+L2Rldi9udWxsOyB0aGVuCiAgICBlY2hvICJCcmlkZ2UgYnItZHB1IGFscmVhZHkgZXhpc3RzLiBObyBuZWVkIHRvIGNvbmZpZ3VyZSBpdC4iCiAgICBleGl0IDAKZmkKCiMgRmluZCB0aGUgaW50ZXJmYWNlIHRoYXQgaGFzIHRoZSBkZWZhdWx0IHJvdXRlCkRFRkFVTFRfSU5URVJGQUNFPSQoaXAgcm91dGUgfCBncmVwIGRlZmF1bHQgfCBhd2sgJ3twcmludCAkNX0nIHwgaGVhZCAtbiAxKQoKaWYgWyAteiAiJERFRkFVTFRfSU5URVJGQUNFIiBdOyB0aGVuCiAgICBlY2hvICJFcnJvcjogQ291bGQgbm90IGZpbmQgaW50ZXJmYWNlIHdpdGggZGVmYXVsdCByb3V0ZSIgPiYyCiAgICBleGl0IDEKZmkKCmVjaG8gIkZvdW5kIGRlZmF1bHQgaW50ZXJmYWNlOiAkREVGQVVMVF9JTlRFUkZBQ0UiCgojIENyZWF0ZSBhIHRlbXBvcmFyeSBZQU1MIGZpbGUgd2l0aCB0aGUgaW50ZXJmYWNlIHN1YnN0aXR1dGVkClRNUF9GSUxFPSQobWt0ZW1wKQoKY2F0ID4gIiRUTVBfRklMRSIgPDwgRU9GCmludGVyZmFjZXM6CiAgLSBuYW1lOiBici1kcHUKICAgIHR5cGU6IGxpbnV4LWJyaWRnZQogICAgc3RhdGU6IHVwCiAgICBtdHU6ICROT0RFU19NVFUKICAgIGlwdjY6CiAgICAgIGVuYWJsZWQ6IGZhbHNlCiAgICBpcHY0OgogICAgICBlbmFibGVkOiB0cnVlCiAgICAgIGRoY3A6IHRydWUKICAgICAgYXV0by1kbnM6IHRydWUKICAgICAgYXV0by1nYXRld2F5OiB0cnVlCiAgICAgIGF1dG8tcm91dGVzOiB0cnVlCiAgICBicmlkZ2U6CiAgICAgIG9wdGlvbnM6CiAgICAgICAgc3RwOgogICAgICAgICAgZW5hYmxlZDogZmFsc2UKICAgICAgcG9ydDoKICAgICAgICAtIG5hbWU6ICRERUZBVUxUX0lOVEVSRkFDRQogIC0gbmFtZTogJERFRkFVTFRfSU5URVJGQUNFCiAgICB0eXBlOiBldGhlcm5ldAogICAgc3RhdGU6IHVwCiAgICBtdHU6ICROT0RFU19NVFUKRU9GCgplY2hvICJDcmVhdGVkIE5NU3RhdGUgY29uZmlndXJhdGlvbiB3aXRoIGludGVyZmFjZSAkREVGQVVMVF9JTlRFUkZBQ0UiCmVjaG8gIkFwcGx5aW5nIGNvbmZpZ3VyYXRpb24gdXNpbmcgbm1zdGF0ZWN0bC4uLiIKCiMgQXBwbHkgdGhlIGNvbmZpZ3VyYXRpb24Kbm1zdGF0ZWN0bCBhcHBseSAiJFRNUF9GSUxFIgpSRVNVTFQ9JD8KCiMgQ2xlYW4gdXAKcm0gIiRUTVBfRklMRSIKCiMgUmV0dXJuIHRoZSByZXN1bHQKZXhpdCAkUkVTVUxUCg==
         mode: 0755
         overwrite: true
         path: /usr/local/bin/apply-nmstate-bridge.sh
       - contents:
           source: data:text/plain;charset=utf-8;base64,IyEvYmluL2Jhc2gKCiMgU2NyaXB0IHRvIHdhaXQgZm9yIE9WTi1LIGludGVyZmFjZSAoaWRlbnRpZmllZCBieSBsaW5rLWxvY2FsKSBhbmQgY29uZmlndXJlIHJvdXRpbmcgdGFibGUgMTAwCgpDSEVDS19JTlRFUlZBTD0yICAjIENoZWNrIGV2ZXJ5IDIgc2Vjb25kcwpQUklNQVJZX0lQX0ZJTEU9Ii9ydW4vbm9kZWlwLWNvbmZpZ3VyYXRpb24vcHJpbWFyeS1pcCIKCmVjaG8gIldhaXRpbmcgZm9yIHByaW1hcnkgSVAgZmlsZSB0byBkZXRlcm1pbmUgSVAgdmVyc2lvbi4uLiIKCiMgV2FpdCBmb3IgcHJpbWFyeS1pcCBmaWxlIGFuZCBkZXRlcm1pbmUgSVAgdmVyc2lvbiBmaXJzdAp3aGlsZSBbICEgLWYgIiRQUklNQVJZX0lQX0ZJTEUiIF0gfHwgWyAhIC1zICIkUFJJTUFSWV9JUF9GSUxFIiBdOyBkbwogICAgc2xlZXAgJENIRUNLX0lOVEVSVkFMCmRvbmUKCmJyX2RwdV9pcD0kKGNhdCAiJFBSSU1BUllfSVBfRklMRSIgfCB0ciAtZCAnWzpzcGFjZTpdJykKZWNobyAiVXNpbmcgYnItZHB1IElQIGZyb20gJFBSSU1BUllfSVBfRklMRTogJGJyX2RwdV9pcCIKCiMgRGV0ZWN0IElQIHZlcnNpb24gYmFzZWQgb24gYnItZHB1IElQIGFuZCBzZXQgYWxsIHZhcmlhYmxlcyBhY2NvcmRpbmdseQppZiBbWyAiJGJyX2RwdV9pcCIgPX4gOiBdXTsgdGhlbgogICAgSVBfVkVSU0lPTj0iNiIKICAgIElQX0ZMQUc9Ii02IgogICAgUFJFRklYX0xFTj0iMTI4IgogICAgTElOS19MT0NBTF9QQVRURVJOPSJeZmU4MDoiCiAgICBlY2hvICJEZXRlY3RlZCBJUHY2IGNvbmZpZ3VyYXRpb24iCmVsc2UKICAgIElQX1ZFUlNJT049IjQiCiAgICBJUF9GTEFHPSItNCIKICAgIFBSRUZJWF9MRU49IjMyIgogICAgTElOS19MT0NBTF9QQVRURVJOPSJeMTY5Wy5dMjU0IgogICAgZWNobyAiRGV0ZWN0ZWQgSVB2NCBjb25maWd1cmF0aW9uIgpmaQoKZWNobyAiV2FpdGluZyBmb3IgT1ZOLUsgaW50ZXJmYWNlICh3aXRoIGxpbmstbG9jYWwgYWRkcmVzcykgdG8gZ2V0IGFuIElQIGFkZHJlc3MuLi4iCgp3aGlsZSB0cnVlOyBkbwogICAgIyBGaW5kIGFsbCBpbnRlcmZhY2VzIHdpdGggbGluay1sb2NhbCBhZGRyZXNzZXMgdXNpbmcgSlNPTiBvdXRwdXQKICAgIG92bmtfaWZhY2VzPSQoaXAgLWogYWRkciBzaG93IHwganEgLS1hcmcgcGF0dGVybiAiJExJTktfTE9DQUxfUEFUVEVSTiIgLXIgJy5bXSB8IHNlbGVjdCguYWRkcl9pbmZvW10/IHwgLmxvY2FsIHwgdGVzdCgkcGF0dGVybikpIHwgLmlmbmFtZScgfCBzb3J0IC11KQogICAgCiAgICBmb3Igb3Zua19pZmFjZSBpbiAkb3Zua19pZmFjZXM7IGRvCiAgICAgICAgIyBHZXQgbm9uLWxpbmstbG9jYWwgSVAgYWRkcmVzcyBmb3IgdGhlIGRldGVjdGVkIElQIHZlcnNpb24KICAgICAgICBvdm5rX2lwPSQoaXAgJElQX0ZMQUcgLWogYWRkciBzaG93ICIkb3Zua19pZmFjZSIgfCBqcSAtLWFyZyBwYXR0ZXJuICIkTElOS19MT0NBTF9QQVRURVJOIiAtciAnLltdIHwgLmFkZHJfaW5mb1tdPyB8IHNlbGVjdCgubG9jYWwgfCB0ZXN0KCRwYXR0ZXJuKSB8IG5vdCkgfCAubG9jYWwnIHwgaGVhZCAtbjEpCiAgICAgICAgCiAgICAgICAgaWYgWyAtbiAiJG92bmtfaXAiIF07IHRoZW4KICAgICAgICAgICAgZWNobyAiRm91bmQgT1ZOLUsgaW50ZXJmYWNlOiAkb3Zua19pZmFjZSB3aXRoIElQdiR7SVBfVkVSU0lPTn06ICRvdm5rX2lwIgogICAgICAgICAgICAKICAgICAgICAgICAgIyBHZXQgYnItZHB1IG5ldHdvcmsgZnJvbSByb3V0aW5nIHRhYmxlIHVzaW5nIEpTT04KICAgICAgICAgICAgYnJfZHB1X25ldHdvcms9JChpcCAkSVBfRkxBRyAtaiByb3V0ZSBzaG93IGRldiBici1kcHUgfCBqcSAtciAnLltdIHwgc2VsZWN0KC5wcm90b2NvbCA9PSAia2VybmVsIikgfCAuZHN0JyB8IGhlYWQgLW4xKQogICAgICAgICAgICAKICAgICAgICAgICAgaWYgWyAteiAiJGJyX2RwdV9uZXR3b3JrIiBdOyB0aGVuCiAgICAgICAgICAgICAgICBlY2hvICJFcnJvcjogQ291bGQgbm90IGZpbmQgYnItZHB1IG5ldHdvcmsiCiAgICAgICAgICAgICAgICBleGl0IDEKICAgICAgICAgICAgZmkKICAgICAgICAgICAgCiAgICAgICAgICAgIGVjaG8gImJyLWRwdSBuZXR3b3JrOiAkYnJfZHB1X25ldHdvcmsiCiAgICAgICAgICAgIAogICAgICAgICAgICAjIEdldCBici1kcHUgZ2F0ZXdheSBmcm9tIGRlZmF1bHQgcm91dGUKICAgICAgICAgICAgYnJfZHB1X2dhdGV3YXk9JChpcCAkSVBfRkxBRyAtaiByb3V0ZSB8IGpxIC1yICcuW10gfCBzZWxlY3QoLmRzdCA9PSAiZGVmYXVsdCIgYW5kIC5kZXYgPT0gImJyLWRwdSIpIHwgLmdhdGV3YXknIHwgaGVhZCAtbjEpCiAgICAgICAgICAgIAogICAgICAgICAgICBpZiBbIC16ICIkYnJfZHB1X2dhdGV3YXkiIF07IHRoZW4KICAgICAgICAgICAgICAgIGVjaG8gIkVycm9yOiBDb3VsZCBub3QgZmluZCBnYXRld2F5IGZvciBici1kcHUiCiAgICAgICAgICAgICAgICBleGl0IDEKICAgICAgICAgICAgZmkKICAgICAgICAgICAgCiAgICAgICAgICAgIGVjaG8gImJyLWRwdSBnYXRld2F5OiAkYnJfZHB1X2dhdGV3YXkiCiAgICAgICAgICAgIAogICAgICAgICAgICAjIEdldCBPVk4tSyBzdWJuZXQgZnJvbSBESENQIHJvdXRlCiAgICAgICAgICAgIG92bmtfc3VibmV0PSQoaXAgJElQX0ZMQUcgLWogcm91dGUgfCBqcSAtciAiLltdIHwgc2VsZWN0KC5kZXYgPT0gXCIkb3Zua19pZmFjZVwiIGFuZCAucHJvdG9jb2wgPT0gXCJkaGNwXCIgYW5kIC5kc3QgIT0gXCJkZWZhdWx0XCIpIHwgLmRzdCIgfCBoZWFkIC1uMSkKICAgICAgICAgICAgCiAgICAgICAgICAgIGlmIFsgLXogIiRvdm5rX3N1Ym5ldCIgXTsgdGhlbgogICAgICAgICAgICAgICAgZWNobyAiRXJyb3I6IENvdWxkIG5vdCBmaW5kIHN1Ym5ldCBmb3IgJG92bmtfaWZhY2UiCiAgICAgICAgICAgICAgICBleGl0IDEKICAgICAgICAgICAgZmkKICAgICAgICAgICAgCiAgICAgICAgICAgIGVjaG8gIk9WTi1LIHN1Ym5ldDogJG92bmtfc3VibmV0IgogICAgICAgICAgICAKICAgICAgICAgICAgIyBHZXQgbWV0cmljIGZyb20gYnItZHB1IChkZWZhdWx0IHRvIDQyNSBpZiBub3QgZm91bmQpCiAgICAgICAgICAgIGJyX2RwdV9tZXRyaWM9JChpcCAkSVBfRkxBRyAtaiByb3V0ZSBzaG93IGRldiBici1kcHUgfCBqcSAtciAnLltdIHwgc2VsZWN0KC5wcm90b2NvbCA9PSAia2VybmVsIikgfCAubWV0cmljIC8vIDQyNScgfCBoZWFkIC1uMSkKICAgICAgICAgICAgCiAgICAgICAgICAgIGVjaG8gIiIKICAgICAgICAgICAgZWNobyAiQ3JlYXRpbmcgcm91dGluZyBydWxlcyBpbiB0YWJsZSAxMDAuLi4iCiAgICAgICAgICAgIAogICAgICAgICAgICAjIDEuIEFkZCBwb2xpY3kgcm91dGluZyBydWxlCiAgICAgICAgICAgIGVjaG8gIkFkZGluZyBydWxlOiBmcm9tICRicl9kcHVfaXAvJFBSRUZJWF9MRU4gbG9va3VwIDEwMCIKICAgICAgICAgICAgIyBDaGVjayBpZiBydWxlIGFscmVhZHkgZXhpc3RzIHVzaW5nIEpTT04KICAgICAgICAgICAgaWYgaXAgJElQX0ZMQUcgLWogcnVsZSBsaXN0IHwganEgLWUgLS1hcmcgc3JjICIkYnJfZHB1X2lwIiAnLltdIHwgc2VsZWN0KC5zcmMgPT0gJHNyYyBhbmQgLnRhYmxlID09ICIxMDAiKScgPiAvZGV2L251bGwgMj4mMTsgdGhlbgogICAgICAgICAgICAgICAgZWNobyAiUnVsZSBhbHJlYWR5IGV4aXN0cyIKICAgICAgICAgICAgZWxzZQogICAgICAgICAgICAgICAgaXAgJElQX0ZMQUcgcnVsZSBhZGQgZnJvbSAkYnJfZHB1X2lwLyRQUkVGSVhfTEVOIGxvb2t1cCAxMDAgMj4vZGV2L251bGwgJiYgZWNobyAiUnVsZSBhZGRlZCBzdWNjZXNzZnVsbHkiIHx8IGVjaG8gIkZhaWxlZCB0byBhZGQgcnVsZSAobWF5IGFscmVhZHkgZXhpc3QpIgogICAgICAgICAgICBmaQogICAgICAgICAgICAKICAgICAgICAgICAgIyAyLiBBZGQgT1ZOLUsgcm91dGUgdmlhIGdhdGV3YXkKICAgICAgICAgICAgZWNobyAiQWRkaW5nIHJvdXRlOiAkb3Zua19zdWJuZXQgdmlhICRicl9kcHVfZ2F0ZXdheSB0YWJsZSAxMDAiCiAgICAgICAgICAgICMgQ2hlY2sgaWYgcm91dGUgYWxyZWFkeSBleGlzdHMgdXNpbmcgSlNPTgogICAgICAgICAgICBpZiBpcCAkSVBfRkxBRyAtaiByb3V0ZSBzaG93IHRhYmxlIDEwMCB8IGpxIC1lIC0tYXJnIGRzdCAiJG92bmtfc3VibmV0IiAnLltdIHwgc2VsZWN0KC5kc3QgPT0gJGRzdCknID4gL2Rldi9udWxsIDI+JjE7IHRoZW4KICAgICAgICAgICAgICAgIGVjaG8gIlJvdXRlIGFscmVhZHkgZXhpc3RzIgogICAgICAgICAgICBlbHNlCiAgICAgICAgICAgICAgICBpcCAkSVBfRkxBRyByb3V0ZSBhZGQgJG92bmtfc3VibmV0IHZpYSAkYnJfZHB1X2dhdGV3YXkgdGFibGUgMTAwIDI+L2Rldi9udWxsICYmIGVjaG8gIlJvdXRlIGFkZGVkIHN1Y2Nlc3NmdWxseSIgfHwgZWNobyAiRmFpbGVkIHRvIGFkZCByb3V0ZSAobWF5IGFscmVhZHkgZXhpc3QpIgogICAgICAgICAgICBmaQogICAgICAgICAgICAKICAgICAgICAgICAgIyAzLiBBZGQgYnItZHB1IHN1Ym5ldCByb3V0ZQogICAgICAgICAgICBlY2hvICJBZGRpbmcgcm91dGU6ICRicl9kcHVfbmV0d29yayBkZXYgYnItZHB1IHByb3RvIGtlcm5lbCBzY29wZSBsaW5rIHNyYyAkYnJfZHB1X2lwIG1ldHJpYyAkYnJfZHB1X21ldHJpYyB0YWJsZSAxMDAiCiAgICAgICAgICAgICMgQ2hlY2sgaWYgcm91dGUgYWxyZWFkeSBleGlzdHMgdXNpbmcgSlNPTgogICAgICAgICAgICBpZiBpcCAkSVBfRkxBRyAtaiByb3V0ZSBzaG93IHRhYmxlIDEwMCB8IGpxIC1lIC0tYXJnIGRzdCAiJGJyX2RwdV9uZXR3b3JrIiAnLltdIHwgc2VsZWN0KC5kc3QgPT0gJGRzdCBhbmQgLmRldiA9PSAiYnItZHB1IiknID4gL2Rldi9udWxsIDI+JjE7IHRoZW4KICAgICAgICAgICAgICAgIGVjaG8gIlJvdXRlIGFscmVhZHkgZXhpc3RzIgogICAgICAgICAgICBlbHNlCiAgICAgICAgICAgICAgICBpcCAkSVBfRkxBRyByb3V0ZSBhZGQgJGJyX2RwdV9uZXR3b3JrIGRldiBici1kcHUgcHJvdG8ga2VybmVsIHNjb3BlIGxpbmsgc3JjICRicl9kcHVfaXAgbWV0cmljICRicl9kcHVfbWV0cmljIHRhYmxlIDEwMCAyPi9kZXYvbnVsbCAmJiBlY2hvICJSb3V0ZSBhZGRlZCBzdWNjZXNzZnVsbHkiIHx8IGVjaG8gIkZhaWxlZCB0byBhZGQgcm91dGUgKG1heSBhbHJlYWR5IGV4aXN0KSIKICAgICAgICAgICAgZmkKICAgICAgICAgICAgCiAgICAgICAgICAgIGVjaG8gIiIKICAgICAgICAgICAgZWNobyAiUm91dGluZyBjb25maWd1cmF0aW9uIGNvbXBsZXRlZCEiCiAgICAgICAgICAgIGV4aXQgMAogICAgICAgIGZpCiAgICBkb25lCiAgICAKICAgIHNsZWVwICRDSEVDS19JTlRFUlZBTApkb25lCg==
         mode: 0755
         overwrite: true
         path: /usr/local/bin/configure-p0-routing.sh
       - contents:
           source: data:text/plain;charset=utf-8;base64,W2tleWZpbGVdCnVubWFuYWdlZC1kZXZpY2VzPWludGVyZmFjZS1uYW1lOm92bi1rOHMtKg==
         mode: 0644
         overwrite: true
         path: "/etc/NetworkManager/conf.d/unmanage-ovnk-interface.conf"
   systemd:
     units:
       - contents: |
           [Unit]
           Description=Apply NMState bridge configuration
           After=network.target NetworkManager.service
           Wants=NetworkManager.service
          
           [Service]
           Type=oneshot
           ExecStart=/usr/local/bin/apply-nmstate-bridge.sh 9000
           RemainAfterExit=yes
          
           [Install]
           WantedBy=multi-user.target
         enabled: true
         name: nmstate-bridge.service
       - name: ovs-configuration.service
         enabled: false
       - name: openvswitch.service
         enabled: false
         mask: true
       - name: wait-for-br-ex-up.service
         enabled: false
       - name: p0-routing.service
         enabled: true
         contents: |
           [Unit]
           Description=Configure p0 interface routing
           After=kubelet.service network-online.target
           Wants=network-online.target
          
           [Service]
           Type=oneshot
           ExecStart=/usr/local/bin/configure-p0-routing.sh
           RemainAfterExit=yes
           StandardOutput=journal
           StandardError=journal
          
           [Install]
           WantedBy=multi-user.target

```

2. Apply the configuration:

```shell
oc apply -f worker-dpu-configuration.yaml
# Expected output
machineconfig.machineconfiguration.openshift.io/worker-dpu-configuration created
```

3. Verify that the MachineConfig is applied successfully: 

```shell
oc get machineconfig worker-dpu-configuration
# Expected output 
NAME                     GENERATEDBYCONTROLLER   IGNITIONVERSION   AGE
worker-dpu-configuration                         3.2.0             2m
```

**Note:** The Machine Config Operator will automatically reboot worker nodes to apply these changes once the nodes are added to the cluster. Monitor the worker nodes during the update process to ensure successful configuration.   

##### 4.1.2 MachineConfig Decoded Scripts (For Information Only)

The MachineConfig resource applied in the previous step includes a few encoded scripts.   
Here is a decoded description of the scripts:

* A script for detecting the default network interface and creating the **br-dpu** bridge.


```shell
#!/bin/bash

NODES_MTU=9000

# Wait indefinitely for an interface to get an IP address and default route
WAIT_INTERVAL=5  # Check interval in seconds
ATTEMPT=1

echo "Waiting for network configuration..."
while true; do
    # Check if we have an interface with a default route
    DEFAULT_INTERFACE=$(ip route | grep default | awk '{print $5}' | head -n 1)

    if [ -n "$DEFAULT_INTERFACE" ]; then
        # Check if that interface has an IP address
        IP_ADDR=$(ip addr show dev "$DEFAULT_INTERFACE" | grep -w "inet" | awk '{print $2}')

        if [ -n "$IP_ADDR" ]; then
            echo "Network is ready. Interface $DEFAULT_INTERFACE has IP address $IP_ADDR"
            break
        fi
    fi

    echo "Waiting for network configuration... (Attempt $ATTEMPT)"
    sleep $WAIT_INTERVAL
    ATTEMPT=$((ATTEMPT + 1))
done

# Check if the bridge already exists
if ip link show br-dpu &>/dev/null; then
    echo "Bridge br-dpu already exists. No need to configure it."
    exit 0
fi

# Find the interface that has the default route
DEFAULT_INTERFACE=$(ip route | grep default | awk '{print $5}' | head -n 1)

if [ -z "$DEFAULT_INTERFACE" ]; then
    echo "Error: Could not find interface with default route" >&2
    exit 1
fi

echo "Found default interface: $DEFAULT_INTERFACE"

# Create a temporary YAML file with the interface substituted
TMP_FILE=$(mktemp)

cat > "$TMP_FILE" << EOF
interfaces:
  - name: br-dpu
    type: linux-bridge
    state: up
    mtu: $NODES_MTU
    ipv6:
      enabled: false
    ipv4:
      enabled: true
      dhcp: true
      auto-dns: true
      auto-gateway: true
      auto-routes: true
    bridge:
      options:
        stp:
          enabled: false
      port:
        - name: $DEFAULT_INTERFACE
  - name: $DEFAULT_INTERFACE
    type: ethernet
    state: up
    mtu: $NODES_MTU
EOF

echo "Created NMState configuration with interface $DEFAULT_INTERFACE"
echo "Applying configuration using nmstatectl..."

# Apply the configuration
nmstatectl apply "$TMP_FILE"
RESULT=$?

# Clean up
rm "$TMP_FILE"

# Return the result
exit $RESULT
```


* A script for setting ip rules required to allow control plane traffic between pods and workers over the management network:

```shell
#!/bin/bash

# Script to wait for OVN-K interface (identified by link-local) and configure routing table 100

CHECK_INTERVAL=2  # Check every 2 seconds
PRIMARY_IP_FILE="/run/nodeip-configuration/primary-ip"

echo "Waiting for primary IP file to determine IP version..."

# Wait for primary-ip file and determine IP version first
while [ ! -f "$PRIMARY_IP_FILE" ] || [ ! -s "$PRIMARY_IP_FILE" ]; do
   sleep $CHECK_INTERVAL
done

br_dpu_ip=$(cat "$PRIMARY_IP_FILE" | tr -d '[:space:]')
echo "Using br-dpu IP from $PRIMARY_IP_FILE: $br_dpu_ip"

# Detect IP version based on br-dpu IP and set all variables accordingly
if [[ "$br_dpu_ip" =~ : ]]; then
   IP_VERSION="6"
   IP_FLAG="-6"
   PREFIX_LEN="128"
   LINK_LOCAL_PATTERN="^fe80:"
   echo "Detected IPv6 configuration"
else
   IP_VERSION="4"
   IP_FLAG="-4"
   PREFIX_LEN="32"
   LINK_LOCAL_PATTERN="^169[.]254"
   echo "Detected IPv4 configuration"
fi

echo "Waiting for OVN-K interface (with link-local address) to get an IP address..."

while true; do
   # Find all interfaces with link-local addresses using JSON output
   ovnk_ifaces=$(ip -j addr show | jq --arg pattern "$LINK_LOCAL_PATTERN" -r '.[] | select(.addr_info[]? | .local | test($pattern)) | .ifname' | sort -u)
  
   for ovnk_iface in $ovnk_ifaces; do
       # Get non-link-local IP address for the detected IP version
       ovnk_ip=$(ip $IP_FLAG -j addr show "$ovnk_iface" | jq --arg pattern "$LINK_LOCAL_PATTERN" -r '.[] | .addr_info[]? | select(.local | test($pattern) | not) | .local' | head -n1)
      
       if [ -n "$ovnk_ip" ]; then
           echo "Found OVN-K interface: $ovnk_iface with IPv${IP_VERSION}: $ovnk_ip"
          
           # Get br-dpu network from routing table using JSON
           br_dpu_network=$(ip $IP_FLAG -j route show dev br-dpu | jq -r '.[] | select(.protocol == "kernel") | .dst' | head -n1)
          
           if [ -z "$br_dpu_network" ]; then
               echo "Error: Could not find br-dpu network"
               exit 1
           fi
          
           echo "br-dpu network: $br_dpu_network"
          
           # Get br-dpu gateway from default route
           br_dpu_gateway=$(ip $IP_FLAG -j route | jq -r '.[] | select(.dst == "default" and .dev == "br-dpu") | .gateway' | head -n1)
          
           if [ -z "$br_dpu_gateway" ]; then
               echo "Error: Could not find gateway for br-dpu"
               exit 1
           fi
          
           echo "br-dpu gateway: $br_dpu_gateway"
          
           # Get OVN-K subnet from DHCP route
           ovnk_subnet=$(ip $IP_FLAG -j route | jq -r ".[] | select(.dev == \"$ovnk_iface\" and .protocol == \"dhcp\" and .dst != \"default\") | .dst" | head -n1)
          
           if [ -z "$ovnk_subnet" ]; then
               echo "Error: Could not find subnet for $ovnk_iface"
               exit 1
           fi
          
           echo "OVN-K subnet: $ovnk_subnet"
          
           # Get metric from br-dpu (default to 425 if not found)
           br_dpu_metric=$(ip $IP_FLAG -j route show dev br-dpu | jq -r '.[] | select(.protocol == "kernel") | .metric // 425' | head -n1)
          
           echo ""
           echo "Creating routing rules in table 100..."
          
           # 1. Add policy routing rule
           echo "Adding rule: from $br_dpu_ip/$PREFIX_LEN lookup 100"
           # Check if rule already exists using JSON
           if ip $IP_FLAG -j rule list | jq -e --arg src "$br_dpu_ip" '.[] | select(.src == $src and .table == "100")' > /dev/null 2>&1; then
               echo "Rule already exists"
           else
               ip $IP_FLAG rule add from $br_dpu_ip/$PREFIX_LEN lookup 100 2>/dev/null && echo "Rule added successfully" || echo "Failed to add rule (may already exist)"
           fi
          
           # 2. Add OVN-K route via gateway
           echo "Adding route: $ovnk_subnet via $br_dpu_gateway table 100"
           # Check if route already exists using JSON
           if ip $IP_FLAG -j route show table 100 | jq -e --arg dst "$ovnk_subnet" '.[] | select(.dst == $dst)' > /dev/null 2>&1; then
               echo "Route already exists"
           else
               ip $IP_FLAG route add $ovnk_subnet via $br_dpu_gateway table 100 2>/dev/null && echo "Route added successfully" || echo "Failed to add route (may already exist)"
           fi
          
           # 3. Add br-dpu subnet route
           echo "Adding route: $br_dpu_network dev br-dpu proto kernel scope link src $br_dpu_ip metric $br_dpu_metric table 100"
           # Check if route already exists using JSON
           if ip $IP_FLAG -j route show table 100 | jq -e --arg dst "$br_dpu_network" '.[] | select(.dst == $dst and .dev == "br-dpu")' > /dev/null 2>&1; then
               echo "Route already exists"
           else
               ip $IP_FLAG route add $br_dpu_network dev br-dpu proto kernel scope link src $br_dpu_ip metric $br_dpu_metric table 100 2>/dev/null && echo "Route added successfully" || echo "Failed to add route (may already exist)"
           fi
          
           echo ""
           echo "Routing configuration completed!"
           exit 0
       fi
   done
  
   sleep $CHECK_INTERVAL
done
```

                                      

#### 4.2 Cluster Configuration

Two critical cluster-level modifications are necessary to enable automatic SR-IOV resource injection for DPU-accelerated networking.

##### 4.2.1 Custom Cluster Feature \- MutatingAdmissionPolicy

The MutatingAdmissionPolicy feature gate must be enabled on the cluster to support the OVN-Kubernetes resource injector. This Kubernetes admission policy automatically injects SR-IOV virtual function (VF) resource requests into pods that require OVN-Kubernetes accelerated networking on DPU-enabled worker nodes.

This configuration modifies the global cluster FeatureGate to explicitly enable the MutatingAdmissionPolicy feature.

**Warning:** Using CustomNoUpgrade prevents the cluster from being upgraded. This state is generally irreversible and intended for use cases where future updates are not required.

1. Patch the existing cluster FeatureGate to enable MutatingAdmissionPolicy:

```shell
oc patch featuregate cluster --type=merge -p '
  {
    "spec": {
      "featureSet": "CustomNoUpgrade",
      "customNoUpgrade": {
        "enabled": [
          "MutatingAdmissionPolicy"
        ]
      }
    }
  }'
```

2. Expected Output:

```shell
featuregate.config.openshift.io/cluster patched
```

   

##### 4.2.2 Control Plane Node Patching

The MutatingAdmissionPolicy injects SR-IOV resource requests into pods that use the OVN-Kubernetes network attachment. Some system pods may need to use this network attachment while running on control plane nodes.  
All control plane nodes must be patched to include resource capacity and allocatable values for the SR-IOV VF resources.  
The patch allows these pods to be scheduled on control plane nodes by satisfying the scheduler's resource requirements (actual device allocation only occurs on worker nodes with physical DPU hardware).

1. Apply the configuration:

```shell
for node in $(oc get nodes -l node-role.kubernetes.io/control-plane -o jsonpath='{.items[*].metadata.name}'); do oc patch node "$node" --subresource=status --type=json -p="[{\"op\": \"add\", \"path\": \"/status/capacity/openshift.io~1bf3-p0-vfs\", \"value\": \"10000\"},{\"op\": \"add\", \"path\": \"/status/allocatable/openshift.io~1bf3-p0-vfs\", \"value\": \"10000\"}]"; done

# Expected output 
node/master-0 patched
node/master-1 patched
node/master-2 patched
```

2. Verify the configuration was applied:

```shell
oc get nodes -l node-role.kubernetes.io/control-plane -o json | \
  jq '.items[] | {name: .metadata.name, capacity: .status.capacity."openshift.io/bf3-p0-vfs", allocatable: .status.allocatable."openshift.io/bf3-p0-vfs"}'

# Example output:
{
  "name": "master-0",
  "capacity": "10k",
  "allocatable": "10k"
}
{
  "name": "master-1",
  "capacity": "10k",
  "allocatable": "10k"
}
{
  "name": "master-2",
  "capacity": "10k",
  "allocatable": "10k"
}
```

**Note:** This patch only applies to existing control plane nodes at the time this script runs. Any new control plane nodes added to the cluster will need to be patched as well.

##### 4.2.3 Create DPF Namespace

```shell
oc create namespace dpf-operator-system

# Expected output 
namespace/dpf-operator-system created
```

#### 4.3 Operators Installation 

* Use OpenShift CLI and follow the instructions below to install operators:  
  * **Note:** Some of the operators are already installed via Assisted Installer

##### 4.3.1 Cert Manager Operator

**Installation Instructions:**

1. Create the operator resources file:

```shell
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-cert-manager-operator
  namespace: cert-manager
spec:
  targetNamespaces:
  - cert-manager
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-cert-manager-operator
  namespace: cert-manager
spec:
  channel: stable-v1
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace                                          
```

   

2. Apply the file:

```shell
oc apply -f cert-manager-operator.yaml

# Expected output 
namespace/cert-manager created
operatorgroup.operators.coreos.com/openshift-cert-manager-operator created
subscription.operators.coreos.com/openshift-cert-manager-operator created
```

3. Verify operator pods are running:

```shell
oc get pods -n cert-manager

# Example Output:
NAME                                                       READY   STATUS    RESTARTS   AGE
cert-manager-7cfb4fbb84-wj2gg                              1/1     Running   0          2d22h
cert-manager-cainjector-854f669657-zlrqd                   1/1     Running   0          2d22h
cert-manager-operator-controller-manager-cd468b77f-9k24n   1/1     Running   0          2d22h
cert-manager-webhook-68fd6d5f5c-2pcq5                      1/1     Running   0          2d22h
```

##### 4.3.2 MetalLB Operator

**Installation Instructions:**

1. Create the operator resources file:

```shell
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: metallb-operator
  namespace: openshift-operators
spec:
  channel: "stable"
  name: metallb-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
  config:
    # Tolerate the taint on the master nodes
    tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
    # Force scheduling only on nodes with the control-plane label
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: "node-role.kubernetes.io/control-plane"
              operator: "Exists"
```

   

2. Apply the file:

```shell
oc apply -f metallb-operator.yaml
# Expected output:
namespace/metallb-system created
subscription.operators.coreos.com/metallb-operator created
```

3. Verify operator pods are running:

```shell
oc get pods -n openshift-operators | grep metallb

# Example output:
metallb-operator-controller-manager-76d599495-8rcf7   1/1     Running   0          2d19h
metallb-operator-webhook-server-5bb6fcbbb8-chg8z      1/1     Running   0          2d19h
```

##### 4.3.3 GitOps Operator

**Installation Instructions:**

1. Create the operator resources file:

```shell
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-gitops-operator
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  channel: gitops-1.16
  config:
    env:
    - name: ARGOCD_CLUSTER_CONFIG_NAMESPACES
      value: "openshift-gitops,dpf-operator-system"
    - name: CONTROLLER_CLUSTER_ROLE
      value: "cluster-admin"
    - name: SERVER_CLUSTER_ROLE
      value: "cluster-admin"
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

   

2. Apply the file:

```shell
oc apply -f gitops-operator.yaml

# Expected output:
namespace/openshift-gitops-operator created
operatorgroup.operators.coreos.com/openshift-gitops-operator created
subscription.operators.coreos.com/openshift-gitops-operator created
```

3. Verify operator pods are running:

```shell
oc get pods -n openshift-gitops-operator

# Example output:
NAME                                                            READY   STATUS    RESTARTS   AGE
openshift-gitops-operator-controller-manager-759b6c9ff9-752sk   2/2     Running   0          2d18h
```

##### 4.3.4 NVIDIA Maintenance Operator

**Installation Instructions:**

1. Create the operator helm values file named `maintenance-operator-values.yaml`:

```shell
operatorConfig:
  maxParallelOperations: 60%
operator:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: "node-role.kubernetes.io/master"
                operator: Exists
          - matchExpressions:
              - key: "node-role.kubernetes.io/control-plane"
                operator: Exists
  tolerations:
    - key: node-role.kubernetes.io/master
      operator: Exists
      effect: NoSchedule
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
```

2. Install the operator with the values file using helm:

```shell
helm upgrade --install maintenance-operator oci://ghcr.io/mellanox/maintenance-operator-chart \
  --namespace dpf-operator-system \
  --create-namespace \
  --version 0.2.0 \
  --values maintenance-operator-values.yaml \
  --wait

# Expected output:
NAME: maintenance-operator
LAST DEPLOYED: Thu Jan  1 14:25:12 2026
NAMESPACE: dpf-operator-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

3. Verify operator pods are running:

```shell
oc get pods -n dpf-operator-system

# Example output:
maintenance-operator-585767f779-kps9c   1/1     Running   0          2d23h
```

#### 4.4 Operators Configuration

##### 4.4.1 Configure NFD Operator

1. Define a set of cluster variables:

```
# OpenShift Cluster variables
export CLUSTER_NAME="doca-mgmt"  # Management cluster name 
export BASE_DOMAIN="example.com" # Management cluster base domain
export HOST_CLUSTER_API="api.${CLUSTER_NAME}.${BASE_DOMAIN}" # Management cluster API endpoint 
```

   **Note:** those variables are defined as well in sections 5,6.

2. Create the NodeFeatureDiscovery resource file. Notice the **HOST\_CLUSTER\_API** variable is used in the resource file.

```shell
apiVersion: nfd.openshift.io/v1
kind: NodeFeatureDiscovery
metadata:
  name: nfd-instance
  namespace: openshift-nfd
spec:
  operand:
    workerEnvs:
      - name: KUBERNETES_SERVICE_HOST
        value: $HOST_CLUSTER_API
      - name: KUBERNETES_SERVICE_PORT
        value: "6443"
  workerConfig:
    configData: |
      sources:
        pci:
          deviceClassWhitelist:
            - "0200"
            - "03"
            - "12"
            - "0207"
          deviceLabelFields:
            - "vendor"
            - "device"
            - "class"
```

   

3. Apply the file using the following command:

```shell
envsubst < nfd-instance.yaml | oc apply -f -

# Expected output:
nodefeaturediscovery.nfd.openshift.io/nfd-instance configured
```

4. Create a NodeFeatureRule resource file for detecting worker nodes with DPU and label it with “dpu-enabled” label:

```shell
apiVersion: nfd.openshift.io/v1alpha1
kind: NodeFeatureRule
metadata:
  name: dpu-detection-rule
  namespace: openshift-nfd
spec:
  rules:
    - labels:
        dpu-enabled: ""
      matchFeatures:
        - feature: pci.device
          matchExpressions:
            device:
              op: In
              value:
                - a2d6
                - a2dc
            vendor:
              op: In
              value:
                - 15b3
      name: DPU-detection-rule
```

   

5. Apply the file:

```shell
oc apply -f nfd-rule.yaml

# Expected output:
nodefeaturerule.nfd.openshift.io/dpu-detection-rule created
```

##### 4.4.2 Configure MetalLB Operator 

1. Define the following hosted cluster variables used by MetalLB:

```
export HOSTED_CLUSTER_VIP=<HOSTED_CLUSTER_VIP> # Virtual IP for hosted DPU cluster allocated from the management cluster subnet - i.e 10.0.110.200/32
export HOSTED_CLUSTER_NAME="dpf-hosted" # DPU hosted cluster name
```

   **Note:** those variables are defined as well in sections 5\.

2. Create the MetalLB resource file with the following CRs:   
* **MetalLB**: This CR deploys the MetalLB load balancer in the cluster to provide external IP addresses for services.  
* **IPAddressPool**: This CR defines a pool of IP addresses (in this case a single VIP) that MetalLB can assign to services in the specified namespace.  
* **L2Advertisement**: This CR configures MetalLB to advertise the IP addresses from the specified pool using Layer 2 mode.

```shell
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: openshift-operators
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-network
  namespace: openshift-operators
spec:
  addresses:
    - $HOSTED_CLUSTER_VIP  # Virtual IP for hosted DPU cluster allocated from the management cluster subnet 
  serviceAllocation:
    namespaces:
      - clusters-$HOSTED_CLUSTER_NAME # DPU hosted cluster name
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: advertise-lab-network
  namespace: openshift-operators
spec:
  ipAddressPools:
    - lab-network
```


3. Apply the resource file:

```shell
envsubst < metallb-config.yaml | oc apply -f -

# Expected output:
metallb.metallb.io/metallb created
ipaddresspool.metallb.io/lab-network created
l2advertisement.metallb.io/advertise-lab-network created
```

##### 4.4.3 Configure GitOps Operator:

1. Create ArgoCD resource file   
* This CR creates ArgoCD instance in the **dpf-operator-system** namespace to manage the DPF components lifecycle

```shell
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: argocd
  namespace: dpf-operator-system
spec:
  nodePlacement:
    nodeSelector:
      node-role.kubernetes.io/control-plane: ""
    tolerations:
    - key: node-role.kubernetes.io/master
      operator: Exists
      effect: NoSchedule
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
  server:
    route:
      enabled: true
    labels:
      ovn.dpu.nvidia.com/skip-injection: ""
  controller:
    labels:
      ovn.dpu.nvidia.com/skip-injection: ""
  repo:
    labels:
      ovn.dpu.nvidia.com/skip-injection: ""
  applicationSet:
    enabled: false
  sso:
    dex:
      openShiftOAuth: true
  notifications:
    enabled: false
```

2. Apply the resource file:

```shell
oc apply -f argocd-instance.yaml

# Expected output:
argocd.argoproj.io/argocd created

# Wait for the ArgoCD pods to be ready
oc wait deployment argocd-redis -n dpf-operator-system \
 --for=condition=Available --timeout=120s

# Add the skip-injection label to the Redis deployment:
oc patch deployment argocd-redis -n dpf-operator-system \
 --type=merge -p '{"spec":{"template":{"metadata":{"labels":{"ovn.dpu.nvidia.com/skip-injection":""}}}}}'

# Expected Output: 
deployment.apps/argocd-redis patched

# Important: The OpenShift GitOps operator ArgoCD CR does not support custom labels on the Redis component. This label must be applied manually to prevent the OVN resource injector from modifying Redis pods. The ArgoCD operator will not override this patch.
```

##### 4.4.4 Configure Cluster Network Operator

This command is enabling global IP forwarding for OVN-Kubernetes in the OpenShift cluster.  
It enables IP packet forwarding between different networks managed by OVN-Kubernetes.

```shell
oc patch network.operator.openshift.io cluster --type=merge -p \ '{"spec":{"defaultNetwork":{ "ovnKubernetesConfig":{"gatewayConfig":{"ipForwarding":"Global"}}}}}'

# Expected output:
network.operator.openshift.io/cluster patched
```

#### 4.5  Custom SR-IOV Device Plugin Installation

A lightweight, standalone SR-IOV device plugin is used instead of the full SR-IOV Network Operator.   
This custom device plugin includes an intelligent supervisor script that ensures enhanced initialization sequence:

* PF Discovery: Automatically detects NVIDIA BlueField-3 devices by polling the system until the physical function (PF) is available  
* VF Readiness Gate: Blocks the device plugin startup until all VFs are physically present in the system.   
* Dynamic Configuration: Automatically generates per-node configuration by introspecting actual PCI addresses and separates VFs into two resource pools:  
  * openshift.io/bf3-p0-vfs-mgmt   
  * openshift.io/bf3-p0-vfs   
* Resilient Operation: If hardware is not ready within the timeout period (600 seconds), the pod restarts and retries discovery automatically  
    
1. Create the SR-IOV device plugin resources file:

```shell
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-sriov-network-operator
  labels:
    openshift.io/cluster-monitoring: "true"
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sriov-device-plugin
  namespace: openshift-sriov-network-operator
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sriov-device-plugin-scc
  namespace: openshift-sriov-network-operator
rules:
- apiGroups:
  - security.openshift.io
  resourceNames:
  - privileged
  resources:
  - securitycontextconstraints
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sriov-device-plugin-scc-binding
  namespace: openshift-sriov-network-operator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: sriov-device-plugin-scc
subjects:
- kind: ServiceAccount
  name: sriov-device-plugin
  namespace: openshift-sriov-network-operator
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriov-supervisor-script
  namespace: openshift-sriov-network-operator
data:
  entrypoint.sh: |
    #!/bin/bash
    set -e

    TARGET_VENDOR="15b3"
    TARGET_DEVICE="a2dc"
    NODE_NAME="${NODE_NAME:-$(hostname)}"
    
    CONFIG_DIR="/etc/pcidp"
    CONFIG_FILE="${CONFIG_DIR}/${NODE_NAME}"
    mkdir -p "$CONFIG_DIR"

    SYS_PREFIX="/host/sys"
    MGMT_VF_INDEX=1
    DATA_VF_START=2

    # Timeout settings
    MAX_WAIT_SECONDS=600
    POLL_INTERVAL=5

    log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

    log "=== SR-IOV Supervisor Started: $NODE_NAME ==="
    log "Status: Blocking Device Plugin start until VFs are physically present."

    find_pf() {
      for dev in ${SYS_PREFIX}/class/net/*; do
        [ -f "$dev/device/sriov_numvfs" ] || continue
        v=$(cat "$dev/device/vendor" 2>/dev/null | sed 's/0x//')
        d=$(cat "$dev/device/device" 2>/dev/null | sed 's/0x//')
        if [ "$v" = "$TARGET_VENDOR" ] && [ "$d" = "$TARGET_DEVICE" ]; then
          basename "$dev" 
          return 0
        fi
      done
      return 1
    }

    PF=""
    ELAPSED=0
    while [ -z "$PF" ]; do
      PF=$(find_pf)
      if [ -z "$PF" ]; then
        if [ "$ELAPSED" -ge "$MAX_WAIT_SECONDS" ]; then
          log "ERROR: Timeout waiting for PF. Exiting to restart pod."
          exit 1
        fi
        log "Waiting for PF..."
        sleep $POLL_INTERVAL
        ELAPSED=$((ELAPSED + POLL_INTERVAL))
      fi
    done
    
    PF_PATH="${SYS_PREFIX}/class/net/$PF"
    log "Found PF: $PF"

    get_vf_pci() {
      local idx=$1
      if [ -L "$PF_PATH/device/virtfn$idx" ]; then
        basename "$(readlink "$PF_PATH/device/virtfn$idx")"
        return 0
      fi
      return 1
    }

    # Loop indefinitely until hardware appears.
    # While looping, the main binary is NOT running.
    # Kubelet sees 0 resources.
    log "Waiting for VFs..."
    MGMT_PCI=""
    DATA_PCI_LIST=""
    VF_DEVICE_ID=""
    VF_ELAPSED=0

    while true; do
      TOTAL_VFS=$(cat "$PF_PATH/device/sriov_numvfs" 2>/dev/null || echo 0)

      if [ "$TOTAL_VFS" -le "$DATA_VF_START" ]; then
        if [ "$VF_ELAPSED" -ge "$MAX_WAIT_SECONDS" ]; then
          log "ERROR: Timeout waiting for VFs. Exiting."
          exit 1
        fi
        sleep $POLL_INTERVAL
        VF_ELAPSED=$((VF_ELAPSED + POLL_INTERVAL))
        continue
      fi

      LAST_VF_INDEX=$((TOTAL_VFS - 1))
      MISSING=0

      # Check Mgmt VF
      pci=$(get_vf_pci $MGMT_VF_INDEX)
      if [ -z "$pci" ]; then MISSING=1; else
        MGMT_PCI="\"$pci\""
        [ -z "$VF_DEVICE_ID" ] && VF_DEVICE_ID=$(cat "${SYS_PREFIX}/bus/pci/devices/$pci/device" 2>/dev/null | sed 's/0x//')
      fi

      # Check Data VFs
      TEMP_DATA_LIST=""
      for (( i=$DATA_VF_START; i<=$LAST_VF_INDEX; i++ )); do
        pci=$(get_vf_pci $i)
        if [ -z "$pci" ]; then MISSING=1; break; fi
        [ -n "$TEMP_DATA_LIST" ] && TEMP_DATA_LIST="$TEMP_DATA_LIST, "
        TEMP_DATA_LIST="$TEMP_DATA_LIST\"$pci\""
      done

      if [ "$MISSING" -eq 0 ] && [ -n "$VF_DEVICE_ID" ]; then
        DATA_PCI_LIST="$TEMP_DATA_LIST"
        log "All VFs resolved."
        break
      fi
      sleep $POLL_INTERVAL
    done

    log "Generating configuration..."
    cat > "$CONFIG_FILE" <<EOF
    {
      "resourceList": [
        {
          "resourceName": "bf3-p0-vfs",
          "selectors": {
            "vendors": ["$TARGET_VENDOR"],
            "devices": ["$VF_DEVICE_ID"],
            "pciAddresses": [$DATA_PCI_LIST],
            "linkTypes": ["ether"],
            "IsRdma": true,
            "NeedVhostNet": false
          }
        },
        {
          "resourceName": "bf3-p0-vfs-mgmt",
          "selectors": {
            "vendors": ["$TARGET_VENDOR"],
            "devices": ["$VF_DEVICE_ID"],
            "pciAddresses": [$MGMT_PCI],
            "linkTypes": ["ether"],
            "IsRdma": true,
            "NeedVhostNet": false
          }
        }
      ]
    }
    EOF
    log "Configuration written."

    # 4. LAUNCH PLUGIN
    log "=== Starting SR-IOV Device Plugin ==="
    exec /usr/bin/sriovdp -v 10 -logtostderr --resource-prefix=openshift.io --config-file="$CONFIG_FILE"
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: sriov-device-plugin
  namespace: openshift-sriov-network-operator
  labels:
    app: sriov-device-plugin
spec:
  selector:
    matchLabels:
      app: sriov-device-plugin
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 33%
      maxSurge: 0
  template:
    metadata:
      labels:
        app: sriov-device-plugin
    spec:
      serviceAccountName: sriov-device-plugin
      priorityClassName: system-node-critical
      hostNetwork: true
      nodeSelector:
        feature.node.kubernetes.io/network-sriov.capable: "true"
      tolerations:
      - operator: Exists
      
      containers:
      - name: sriov-device-plugin
        image: registry.redhat.io/openshift4/ose-sriov-network-device-plugin-rhel9@sha256:1e3d8511559b72315b71065927c8228900636af25026638c3ff0c1b84698f65a
        command: ["/bin/bash", "/scripts/entrypoint.sh"]
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          privileged: true
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
        volumeMounts:
        - name: supervisor-script
          mountPath: /scripts
        - name: devicesock
          mountPath: /var/lib/kubelet/device-plugins
        - name: config-volume
          mountPath: /etc/pcidp
        - name: host-sys
          mountPath: /host/sys
          readOnly: true
      
      volumes:
      - name: supervisor-script
        configMap:
          name: sriov-supervisor-script
          defaultMode: 0755
      - name: devicesock
        hostPath:
          path: /var/lib/kubelet/device-plugins
          type: Directory
      - name: config-volume
        emptyDir: {}
      - name: host-sys
        hostPath:
          path: /sys
          type: Directory
```

2. Apply the resources file:

```shell
oc apply -f sriov-dp.yaml

# Expected output:
namespace/openshift-sriov-network-operator created
serviceaccount/sriov-device-plugin created
role.rbac.authorization.k8s.io/sriov-device-plugin-scc created
rolebinding.rbac.authorization.k8s.io/sriov-device-plugin-scc-binding created
configmap/sriov-supervisor-script created
Warning: would violate PodSecurity "restricted:latest": host namespaces (hostNetwork=true), privileged (container "sriov-device-plugin" must not set securityContext.privileged=true), allowPrivilegeEscalation != false (container "sriov-device-plugin" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "sriov-device-plugin" must set securityContext.capabilities.drop=["ALL"]), restricted volume types (volumes "devicesock", "host-sys" use restricted volume type "hostPath"), runAsNonRoot != true (pod or container "sriov-device-plugin" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "sriov-device-plugin" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
daemonset.apps/sriov-device-plugin created


```

## 

## **Chapter 5:** Hosted Cluster for DPU Control-Plane

#### 5.1 Define Environment Variables

* From machine with access to the cluster define the following variables: 

```shell
# Define cluster variables (make sure you have no hidden spaces in values)
export HOSTED_CLUSTER_NAME="dpf-hosted" # Name of the DPU hosted cluster
export HOSTED_CONTROL_PLANE_NAMESPACE="clusters-${HOSTED_CLUSTER_NAME}"  # Control plane namespace
export BASE_DOMAIN="example.com"                          # Your base domain
export ETCD_STORAGE_CLASS="lvms-vg1"  # Storage class for etcd (adjust as needed)
export SSH_KEY="$HOME/.ssh/id_rsa.pub" # Path to your SSH public key
export OPENSHIFT_PULL_SECRET="$HOME/openshift_pull.json" # Path to OpenShift pull secret
export HOSTED_CLUSTER_IMAGE="quay.io/openshift-release-dev/ocp-release:4.20.4-multi" # Hosted Cluster OCP Image 
```

* **Note:** Download the OpenShift pull secret from Red Hat website using [this](https://console.redhat.com/openshift/install/pull-secret) link


#### 5.2 Create a DPU Hosted Cluster

In this section we will create a Hosted Cluster using the HyperShift Operator.  
For more details you can refer to the [official Red Hat documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/hosted_control_planes/deploying-hosted-control-planes#hcp-bm-create-infra-console_hcp-deploy-bm).

Follow the steps below to create the Hosted Cluster:

* Verify that all the HyperShift operator pods are running (recreated) before proceeding to next step:

```shell
oc get pods -n hypershift
```

* Create hosted Hypershift Cluster: 


```shell
hypershift create cluster none \
  --name="${HOSTED_CLUSTER_NAME}" \
  --base-domain="${BASE_DOMAIN}" \
  --release-image="${HOSTED_CLUSTER_IMAGE}" \
  --ssh-key="${SSH_KEY}" \
  --pull-secret="${OPENSHIFT_PULL_SECRET}" \    --disable-cluster-capabilities=ImageRegistry,Insights,Console,openshift-samples,Ingress,NodeTuning \
  --network-type=Other \
  --etcd-storage-class="${ETCD_STORAGE_CLASS}" \
  --node-selector='node-role.kubernetes.io/control-plane=""' \
  --node-pool-replicas=0 \
  --node-upgrade-type=Replace \
  --expose-through-load-balancer \
  --control-plane-availability-policy=HighlyAvailable \
  --infra-availability-policy=HighlyAvailable
```

 

```shell
# Expected output:

{"level":"info","ts":"2026-01-04T19:36:48+02:00","msg":"Applied Kube resource","kind":"Namespace","namespace":"","name":"clusters"}
{"level":"info","ts":"2026-01-04T19:36:48+02:00","msg":"Applied Kube resource","kind":"Secret","namespace":"clusters","name":"dpf-hosted-pull-secret"}
{"level":"info","ts":"2026-01-04T19:36:48+02:00","msg":"Applied Kube resource","kind":"Secret","namespace":"clusters","name":"dpf-hosted-ssh-key"}
{"level":"info","ts":"2026-01-04T19:36:48+02:00","msg":"Applied Kube resource","kind":"Secret","namespace":"clusters","name":"dpf-hosted-etcd-encryption-key"}
{"level":"info","ts":"2026-01-04T19:36:48+02:00","msg":"Applied Kube resource","kind":"","namespace":"clusters","name":"dpf-hosted"}
{"level":"info","ts":"2026-01-04T19:36:48+02:00","msg":"Applied Kube resource","kind":"NodePool","namespace":"clusters","name":"dpf-hosted"}

```

**Notes**

* The \--network-type=Other is used because DPF will provide custom networking  
* The \--node-selector ensures control plane pods run on master nodes  
* The nodepool is set to 0 replicas because DPU nodes will be managed separately  
* The kubeconfig secret is automatically created once the hosted cluster is ready

#### 5.3 Hosted Cluster creation verification:

```shell
# Check all pods are running in the control plane namespace
oc get pods -n clusters-$HOSTED_CLUSTER_NAME

# Check HyperShift cluster status
oc get hostedcluster $HOSTED_CLUSTER_NAME -n clusters

# Expected output:
NAME         VERSION   KUBECONFIG                    PROGRESS   AVAILABLE   PROGRESSING   MESSAGE
dpf-hosted             dpf-hosted-admin-kubeconfig   Partial    True        False         The hosted control plane is available

```

#### 5.4 Extract the Hosted Cluster kubeconfig

* **Note:** The Hosted Cluster kubeconfig will be used in later steps of this document.

```shell
oc get secret -n clusters $HOSTED_CLUSTER_NAME-admin-kubeconfig -o jsonpath='{.data.kubeconfig}' | base64 -d > $HOSTED_CLUSTER_NAME.kubeconfig
```

#### 5.5 Add DPU IPAM Controller Manager to the Hosted Cluster Namespace

The hosted cluster's **kube-controller-manager** does not include the node IPAM controller.   
This additional deployment runs a dedicated **kube-controller-manager** instance with only the **node-ipam-controller** enabled, responsible for allocating pod CIDR ranges to DPU worker nodes as they join the hosted cluster.  
Without it, DPU worker nodes do not receive pod CIDR assignments and pods on those nodes cannot obtain IP addresses.

1. Create the resources file:

```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dpu-cluster-node-ipam-controller-manager
  namespace: $HOSTED_CONTROL_PLANE_NAMESPACE
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: dpu-cluster-node-ipam
  strategy:
    rollingUpdate:
      maxSurge: 3
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: dpu-cluster-node-ipam
    spec:
      automountServiceAccountToken: false
      containers:
      - args:
        - --openshift-config=/etc/kubernetes/config/config.json
        - --kubeconfig=/etc/kubernetes/secrets/svc-kubeconfig/kubeconfig
        - --authentication-kubeconfig=/etc/kubernetes/secrets/svc-kubeconfig/kubeconfig
        - --authorization-kubeconfig=/etc/kubernetes/secrets/svc-kubeconfig/kubeconfig
        - --controllers=-bootstrap-signer-controller,-certificatesigningrequest-approving-controller,-certificatesigningrequest-cleaner-controller,-certificatesigningrequest-signing-controller,-cloud-node-lifecycle-controller,-clusterrole-aggregation-controller,-cronjob-controller,-daemonset-controller,-deployment-controller,-disruption-controller,-endpoints-controller,-endpointslice-controller,-endpointslice-mirroring-controller,-ephemeral-volume-controller,-garbage-collector-controller,-horizontal-pod-autoscaler-controller,-job-controller,-kube-apiserver-serving-clustertrustbundle-publisher-controller,-legacy-serviceaccount-token-cleaner-controller,-namespace-controller,-node-lifecycle-controller,-node-route-controller,-persistentvolume-attach-detach-controller,-persistentvolume-binder-controller,-persistentvolume-expander-controller,-persistentvolume-protection-controller,-persistentvolumeclaim-protection-controller,-pod-garbage-collector-controller,-replicaset-controller,-replicationcontroller-controller,-resourceclaim-controller,-resourcequota-controller,-root-ca-certificate-publisher-controller,-selinux-warning-controller,-service-cidr-controller,-service-lb-controller,-serviceaccount-controller,-serviceaccount-token-controller,-statefulset-controller,-storage-version-migrator-controller,-storageversion-garbage-collector-controller,-taint-eviction-controller,-token-cleaner-controller,-ttl-after-finished-controller,-ttl-controller,-validatingadmissionpolicy-status-controller,-volumeattributesclass-protection-controller,node-ipam-controller
        - --allocate-node-cidrs=true
        - --cluster-cidr=10.244.0.0/14
        - --node-cidr-mask-size=24
        - --service-cluster-ip-range=172.31.0.0/16
        - --leader-elect=true
        - --leader-elect-renew-deadline=12s
        - --leader-elect-retry-period=3s
        - --leader-elect-resource-name=dpu-cluster-node-ipam-controller
        - --root-ca-file=/etc/kubernetes/certs/root-ca/ca.crt
        - --secure-port=10257
        - --service-account-private-key-file=/etc/kubernetes/certs/service-signer/service-account.key
        - --service-cluster-ip-range=172.31.0.0/16
        - --use-service-account-credentials=true
        - --cluster-signing-duration=17520h
        - --tls-cert-file=/etc/kubernetes/certs/server/tls.crt
        - --tls-private-key-file=/etc/kubernetes/certs/server/tls.key
        - --v=2
        command:
        - hyperkube
        - kube-controller-manager
        image: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:c4778d7ebdcfdcb188c9f7c45c5934d1e2d8f93f3f6e794214c334ade1fe67f3
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: healthz
            port: 10257
            scheme: HTTPS
          initialDelaySeconds: 45
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 10
        name: kube-controller-manager
        ports:
        - containerPort: 10257
          name: client
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: healthz
            port: 10257
            scheme: HTTPS
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 10
        resources:
          requests:
            cpu: 60m
            memory: 400Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/run/kubernetes
          name: certs
        - mountPath: /etc/kubernetes/certs/cluster-signer
          name: cluster-signer
        - mountPath: /etc/kubernetes/config
          name: kcm-config
        - mountPath: /etc/kubernetes/secrets/svc-kubeconfig
          name: kubeconfig
        - mountPath: /var/log/kube-controller-manager
          name: logs
        - mountPath: /etc/kubernetes/recycler-config
          name: recycler-config
        - mountPath: /etc/kubernetes/certs/root-ca
          name: root-ca
        - mountPath: /etc/kubernetes/certs/server
          name: server-crt
        - mountPath: /etc/kubernetes/certs/service-signer
          name: service-signer
        - mountPath: /etc/kubernetes/certs/service-ca
          name: service-serving-ca
      dnsPolicy: ClusterFirst
      initContainers:
      - command:
        - /usr/bin/control-plane-operator
        - availability-prober
        - --target
        - https://kube-apiserver:6443/readyz
        image: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:e2bd614b4eac679dccbdcb1db8a5451273f1fc4fedcb797bfa5d3320584b6cb3
        imagePullPolicy: IfNotPresent
        name: availability-prober
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      priorityClassName: hypershift-control-plane
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: hypershift.openshift.io/control-plane
        operator: Equal
        value: "true"
      - effect: NoSchedule
        key: hypershift.openshift.io/cluster
        operator: Equal
        value: $HOSTED_CLUSTER_NAME
      volumes:
      - configMap:
          defaultMode: 420
          name: kcm-config
        name: kcm-config
      - configMap:
          defaultMode: 420
          name: root-ca
        name: root-ca
      - emptyDir: {}
        name: logs
      - name: kubeconfig
        secret:
          defaultMode: 416
          secretName: kube-controller-manager-kubeconfig
      - name: cluster-signer
        secret:
          defaultMode: 416
          secretName: cluster-signer-ca
      - emptyDir: {}
        name: certs
      - name: service-signer
        secret:
          defaultMode: 416
          secretName: sa-signing-key
      - name: server-crt
        secret:
          defaultMode: 416
          secretName: kcm-server
      - configMap:
          defaultMode: 420
          name: recycler-config
        name: recycler-config
      - configMap:
          defaultMode: 420
          optional: true
          name: service-serving-ca
        name: service-serving-ca

```

2. Apply the resources file:

```shell
envsubst < dpu-node-ipam-controller.yaml | oc apply -f -

# Expected output:
Warning: spec.template.spec.nodeSelector[node-role.kubernetes.io/master]: use "node-role.kubernetes.io/control-plane" instead
deployment.apps/dpu-cluster-node-ipam-controller-manager created


```

3. Verify that  the **dpu-cluster-node-ipam-controller-manager** pod is running: 

```shell
oc get pods -n $HOSTED_CONTROL_PLANE_NAMESPACE |grep dpu-cluster-node-ipam-controller

# Expected output:
dpu-cluster-node-ipam-controller-manager-57f6c4ff76-kgjlb   1/1     Running     0             2m
```

## 

## **Chapter 6:** DPF Operator Installation and Configuration

#### 6.1 Environment Variables

* Define the following variables:

```shell
# OpenShift Infrastructure
export CLUSTER_NAME="doca-mgmt" # Management cluster name (The OpenShift where HyperShift operator runs)
export BASE_DOMAIN="example.com" # Management cluster base domain
export HOST_CLUSTER_API="api.${CLUSTER_NAME}.${BASE_DOMAIN}" # The OpenShift Management cluster API endpoint 

# NVIDIA DPF 
export TAG="v25.7.1" # DPF operator version
export TARGETCLUSTER_API_SERVER_PORT="6443" # Management Cluster API port
export TARGETCLUSTER_NODE_CIDR=<TARGETCLUSTER_NODE_CIDR> # IP address range for hosts in the management cluster on which DPF operator is installed (e.g., 10.0.110.0/24)
export DPU_P0=<DPU_P0> # Physical DPU port 0 interface name on worker nodes (e.g., ens7f0np0)
export NFS_SERVER_IP=<NFS_SERVER_IP> # NFS server IP for BFB image storage (e.g., 10.0.110.253)
export NFS_BFB_PATH=<NFS_BFB_PATH> # Path for BFB in the NFS server with RW access e.g., ״/״

## Network Configuration
export POD_CIDR="10.128.0.0/14" # Pod network CIDR of the Management Cluster - This is default OCP configuration, if it was changed please update this value accordingly
export SERVICE_CIDR="172.31.0.0/16" # Service network CIDR of the Management Cluster
export VTEP_CIDR=<VTEP_CIDR> # Worker OVN tunnel network range (The management network and the VTEP CIDR network must be routable to each other). e.g., 10.0.120.0/22
export NUM_VFS="46" # Number of VFs to create on DPU interfaces

## Registry URLs 
export REGISTRY="https://helm.ngc.nvidia.com/nvidia/doca" # DPF operator Helm registry
export HELM_REGISTRY_REPO_URL="https://helm.ngc.nvidia.com/nvidia/doca" # NVIDIA DOCA registry
export HBN_NGC_IMAGE_URL="nvcr.io/nvidia/doca/doca_hbn" # HBN container image registry


## BFB Configuration
export BFB_URL=<BFB_URL> # URL to the RHCOS-based BFB image for DPU provisioning (default: "http://bfb.okoyl.xyz/from-ci/rhcos-9.6.20250707-1.6-nvidiabfb.aarch64.bfb")

## OVN_KUBERNETE (4.20.4)
export OVN_TEMPLATE_CHART_URL="oci://ghcr.io/mellanox/charts"
export  OVN_KUBERNETES_IMAGE_REPO=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256
export  OVN_KUBERNETES_IMAGE_TAG=780d11fac73412276b312b3f7c879b5e63da9687c7c8e79fc142e9c6e2f7c4cf

```

* **Notes:**   
  * For more information about the environment variables visit [NVIDIA Official Doc](https://docs.nvidia.com/networking/display/dpf2571/ovn+kubernetes+with+host+based+networking#src-4397531144_OVNKuberneteswithHostBasedNetworking-0.Requiredvariables)  
  * The DPU\_P0 parameter can be discovered during “Add Hosts” operation in Assisted Installer UI. Follow step 8.2.1 to obtain it.

#### 

#### 6.2 Create DPU Image Storage Objects

1. Create the following PV/PVC resource file:

```shell
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: bfb-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  nfs: 
    path: $NFS_BFB_PATH
    server: $NFS_SERVER_IP
  persistentVolumeReclaimPolicy: Delete
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bfb-pvc
  namespace: dpf-operator-system
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeMode: Filesystem
  storageClassName: ""
```

* Note: This PVC will be referenced by bfbPVCName in the DPFOperatorConfig resource created in upcoming steps.  
    
2. Apply the resource file:

```shell
envsubst < bfb-nfs.yaml | oc apply -f -

# Expected output:
persistentvolume/bfb-pv created
persistentvolumeclaim/bfb-pvc created
```

3. Verify the PV/PVC objects are created and Bound:

```shell
oc get persistentvolume  |grep bfb-pv

# Expected output:
bfb-pv                                     10Gi       RWX            Delete           Bound    dpf-operator-system/bfb-pvc

# Check the status of the PVC 
oc get persistentvolumeclaims -n dpf-operator-system

# Expected output:
bfb-pvc   Bound    bfb-pv   10Gi       RWX                           <unset>                 6m42s
```

#### 6.3 Install DPF Operator

1. Install the DPF operator using helm:

```shell
helm repo add --force-update dpf-repository ${REGISTRY}

helm repo update

helm upgrade --install dpf-operator dpf-repository/dpf-operator --namespace dpf-operator-system --version "${TAG}" --set kamajiEtcdDefrag.enabled=false --set isOpenshift=true --set enableNodeFeatureRules=false --wait

# Example output
Release "dpf-operator" does not exist. Installing it now.
NAME: dpf-operator
LAST DEPLOYED: Tue Nov 18 02:44:57 2025
NAMESPACE: dpf-operator-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

   

2. Verify the operator components state:

```shell
# Run the following command:
oc rollout status deployment --namespace dpf-operator-system dpf-operator-controller-manager

# Example output:
deployment "dpf-operator-controller-manager" successfully rolled out

# Run the following command:
oc wait --for=condition=ready --namespace dpf-operator-system pods --all
# Example output:
pod/argocd-application-controller-0 condition met
pod/argocd-redis-b4f94bb8d-wr86b condition met
pod/argocd-repo-server-96765f997-79k9q condition met
pod/argocd-server-648c7ff85f-7frtg condition met
pod/dpf-operator-controller-manager-7bf9744c5f-cwrgc condition met
pod/maintenance-operator-585767f779-8k2lx condition met
```

#### 6.4 Generate BF.cfg ConfigMap

* The BF.cfg ConfigMap contains the boot configuration template that DPU nodes use when joining the hosted cluster. Generate the ConfigMap using a dedicated script and apply it:

```shell
# Download the template generator python script
curl -O https://raw.githubusercontent.com/rh-ecosystem-edge/openshift-dpf/refs/heads/main/scripts/gen_template.py

# Make it executable
chmod +x gen_template.py

# Generate the template as a ConfigMap resource
python3 gen_template.py \
  --cluster "${HOSTED_CLUSTER_NAME}" \
  --hosted-clusters-namespace "clusters" \
  --output-file custom-bfb-template.yaml

# Apply the generated ConfigMap
oc apply -f custom-bfb-template.yaml

# Expected output:
configmap/custom-bfb.cfg created
```

* Note: This  ConfigMap will be referenced by bfCFGTemplateConfigMap in the DPFOperatorConfig resource created in upcoming steps.

#### 6.5 Create DPF Resources

##### 6.5.1 DPFOperatorConfig 

1. Create DPFOperatorConfig resource file:

```shell
apiVersion: operator.dpu.nvidia.com/v1alpha1
kind: DPFOperatorConfig
metadata:
  name: dpfoperatorconfig
  namespace: dpf-operator-system
spec:
  kamajiClusterManager:
    disable: true
  multus:
    disable: true
  networking:
    controlPlaneMTU: 9000
    highSpeedMTU: 9000
  overrides:
    dpuCNIBinPath: /var/lib/cni/bin/
    dpuCNIPath: /run/multus/cni/net.d/
    dpuOpenvSwitchSystemSharedLib64Path: /lib64
    flannelSkipCNIConfigInstallation: false
    kubernetesAPIServerPort: $TARGETCLUSTER_API_SERVER_PORT
    kubernetesAPIServerVIP: $HOST_CLUSTER_API
  provisioningController:
    bfCFGTemplateConfigMap: custom-bfb.cfg 
    bfbPVCName: bfb-pvc 
    dmsTimeout: 900
  staticClusterManager:
    disable: false
```

   

2. Apply the resource file:

```shell
envsubst < dpfoperatorconfig.yaml | oc apply -f -

# Example output
dpfoperatorconfig.operator.dpu.nvidia.com/dpfoperatorconfig created
```

3. Follow the verification steps:

```shell
## Ensure the provisioning and DPUService controller manager deployments are available.
oc rollout status deployment --namespace dpf-operator-system dpf-provisioning-controller-manager
# Example output:
deployment "dpf-provisioning-controller-manager" successfully rolled out

oc rollout status deployment --namespace dpf-operator-system dpuservice-controller-manager
# Example output
deployment "dpuservice-controller-manager" successfully rolled out

## Ensure all pods in the DPF Operator system are Available
oc get pods -n dpf-operator-system

# Example output
NAME                                                   READY   STATUS              RESTARTS   AGE
argocd-application-controller-0                        1/1     Running             0          13h
argocd-redis-7565dcc7fc-nz92w                          1/1     Running             0          13h
argocd-repo-server-5cd89d9dd8-l9s58                    1/1     Running             0          13h
argocd-server-6cbf77db84-gzrl9                         1/1     Running             0          13h
dpf-dpu-detector-kwqtm                                 1/1     Running             0          2m46s
dpf-dpu-detector-mqbn4                                 1/1     Running             0          2m46s
dpf-dpu-detector-vt968                                 1/1     Running             0          2m46s
dpf-operator-controller-manager-7bf9744c5f-zdvhw       1/1     Running             0          6m46s
dpf-provisioning-controller-manager-57d4767fdc-ftm7q   1/1     Running             0          2m47s
dpuservice-controller-manager-7459985c77-bxsqp         1/1     Running             0          2m46s
maintenance-operator-585767f779-4qvvn                  1/1     Running             0          21h
servicechainset-controller-manager-c47f7769-t76xt      0/1     ContainerCreating   0          2m24s
static-cm-controller-manager-6d977d5f84-756pw          1/1     Running             0          2m47s
```

* **Note:** The **servicechainset** pod is expected to show as not running at this phase.


##### 6.5.2 DPUCluster

The DPUCluster custom resource defines the hosted cluster configuration and references the kubeconfig secret for DPU node management.  
 

1. Create a secret in **dpf-operator-system** namespace with the hosted cluster kubeconfig file extracted in section 5.4:

```shell
oc create secret generic doca-admin-kubeconfig \
  -n dpf-operator-system \
  --from-file=super-admin.conf=./$HOSTED_CLUSTER_NAME.kubeconfig \
  --type=Opaque
```

   

2. Create the DPUCluster resource file:

```shell
apiVersion: provisioning.dpu.nvidia.com/v1alpha1
kind: DPUCluster
metadata:
  name: cluster
  namespace: dpf-operator-system
spec:
  type: static
  maxNodes: 10
  kubeconfig: doca-admin-kubeconfig
```

   

3. Apply the resource file:

```shell
oc apply -f dpucluster.yaml

# Expected output:
dpucluster.provisioning.dpu.nvidia.com/cluster created
```

   

4. Verify the DPUCluster is Ready:

```c#
oc get dpucluster -n dpf-operator-system

# Example output
NAME      READY   PHASE   TYPE     MAXNODES   VERSION   AGE
cluster   True    Ready   static   10                   12s
```

##### 

##### 6.5.3 DPUFlavor

DPUFlavor custom resource is an immutable configuration profile for DPUs that specifies system settings (grub, Firmware, OVS), resource allocations, and operational parameters to be applied during DPU provisioning.

1. Create the DPUFlavor resource file:

```shell
apiVersion: provisioning.dpu.nvidia.com/v1alpha1
kind: DPUFlavor
metadata:
  name: flavor
  namespace: dpf-operator-system
spec:
  bfcfgParameters:
    - UPDATE_ATF_UEFI=yes
    - UPDATE_DPU_OS=yes
    - WITH_NIC_FW_UPDATE=yes
  grub:
    kernelParameters:
      - console=hvc0
      - console=ttyAMA0
      - earlycon=pl011,0x13010000
      - fixrttc
      - net.ifnames=0
      - biosdevname=0
      - iommu.passthrough=1
      - cgroup_no_v1=net_prio,net_cls
      - hugepagesz=2048kB
      - hugepages=8072
  nvconfig:
    - device: '*'
      parameters:
        - PF_BAR2_ENABLE=0
        - PER_PF_NUM_SF=1
        - PF_TOTAL_SF=20
        - PF_SF_BAR_SIZE=10
        - NUM_PF_MSIX_VALID=0
        - PF_NUM_PF_MSIX_VALID=1
        - PF_NUM_PF_MSIX=228
        - INTERNAL_CPU_MODEL=1
        - INTERNAL_CPU_OFFLOAD_ENGINE=0
        - SRIOV_EN=1
        - NUM_OF_VFS=$NUM_VFS
        - LAG_RESOURCE_ALLOCATION=1
        - NUM_VF_MSIX=48
  ovs:
    rawConfigScript: IyEvYmluL2Jhc2gKCl9vdnMtdnNjdGwoKSB7CiAgb3ZzLXZzY3RsIC0tbm8td2FpdCAtLXRpbWVvdXQgMTUgIiRAIgp9Cgpfb3ZzLXZzY3RsIHNldCBPcGVuX3ZTd2l0Y2ggLiBvdGhlcl9jb25maWc6ZG9jYS1pbml0PXRydWUKX292cy12c2N0bCBzZXQgT3Blbl92U3dpdGNoIC4gb3RoZXJfY29uZmlnOmRwZGstbWF4LW1lbXpvbmVzPTUwMDAwCl9vdnMtdnNjdGwgc2V0IE9wZW5fdlN3aXRjaCAuIG90aGVyX2NvbmZpZzpody1vZmZsb2FkPXRydWUKX292cy12c2N0bCBzZXQgT3Blbl92U3dpdGNoIC4gb3RoZXJfY29uZmlnOnBtZC1xdWlldC1pZGxlPXRydWUKX292cy12c2N0bCBzZXQgT3Blbl92U3dpdGNoIC4gb3RoZXJfY29uZmlnOm1heC1pZGxlPTIwMDAwCl9vdnMtdnNjdGwgc2V0IE9wZW5fdlN3aXRjaCAuIG90aGVyX2NvbmZpZzptYXgtcmV2YWxpZGF0b3I9NTAwMApfb3ZzLXZzY3RsIC0taWYtZXhpc3RzIGRlbC1iciBvdnNicjEKX292cy12c2N0bCAtLWlmLWV4aXN0cyBkZWwtYnIgb3ZzYnIyCl9vdnMtdnNjdGwgLS1tYXktZXhpc3QgYWRkLWJyIGJyLXNmYwpfb3ZzLXZzY3RsIHNldCBicmlkZ2UgYnItc2ZjIGRhdGFwYXRoX3R5cGU9bmV0ZGV2Cl9vdnMtdnNjdGwgc2V0IGJyaWRnZSBici1zZmMgZmFpbF9tb2RlPXNlY3VyZQpfb3ZzLXZzY3RsIC0tbWF5LWV4aXN0IGFkZC1wb3J0IGJyLXNmYyBwMApfb3ZzLXZzY3RsIHNldCBJbnRlcmZhY2UgcDAgdHlwZT1kcGRrCl9vdnMtdnNjdGwgc2V0IEludGVyZmFjZSBwMCBtdHVfcmVxdWVzdD05MjE2Cl9vdnMtdnNjdGwgc2V0IFBvcnQgcDAgZXh0ZXJuYWxfaWRzOmRwZi10eXBlPXBoeXNpY2FsCl9vdnMtdnNjdGwgLS1tYXktZXhpc3QgYWRkLXBvcnQgYnItc2ZjIHAxCl9vdnMtdnNjdGwgc2V0IEludGVyZmFjZSBwMSB0eXBlPWRwZGsKX292cy12c2N0bCBzZXQgSW50ZXJmYWNlIHAxIG10dV9yZXF1ZXN0PTkyMTYKX292cy12c2N0bCBzZXQgUG9ydCBwMSBleHRlcm5hbF9pZHM6ZHBmLXR5cGU9cGh5c2ljYWwKCiMgQWN0aXZhdGUgRE9DQSBmb3IgT1ZOSwpfb3ZzLXZzY3RsIHNldCBPcGVuX3ZTd2l0Y2ggLiBleHRlcm5hbC1pZHM6b3ZuLWJyaWRnZS1kYXRhcGF0aC10eXBlPW5ldGRldgpfb3ZzLXZzY3RsIHNldCBPcGVuX3ZTd2l0Y2ggLiBvdGhlcl9jb25maWc6aHctb2ZmbG9hZC1jdC11bmlkaXItdWRwLWVuYWJsZWQ9dHJ1ZQoKIyBzZXR1cCBvdm5rdWJlIG1hbmFnZWQgYnJpZGdlLCBici1kcHUgKHRoaXMgY29ycmVzcG9uZHMgdG8gYnItZXggb24gb3ZuayBkb2NzKQpfb3ZzLXZzY3RsIC0tbWF5LWV4aXN0IGFkZC1iciBici1kcHUKX292cy12c2N0bCBici1zZXQtZXh0ZXJuYWwtaWQgYnItZHB1IGJyaWRnZS1pZCBici1kcHUKX292cy12c2N0bCBici1zZXQtZXh0ZXJuYWwtaWQgYnItZHB1IGJyaWRnZS11cGxpbmsgcGJyZHB1dG9icm92bgpfb3ZzLXZzY3RsIHNldCBicmlkZ2UgYnItZHB1IGRhdGFwYXRoX3R5cGU9bmV0ZGV2Cl9vdnMtdnNjdGwgc2V0IEludGVyZmFjZSBici1kcHUgbXR1X3JlcXVlc3Q9OTAwMApfb3ZzLXZzY3RsIC0tbWF5LWV4aXN0IGFkZC1wb3J0IGJyLWRwdSBwZjBocGYKX292cy12c2N0bCBzZXQgSW50ZXJmYWNlIHBmMGhwZiBtdHVfcmVxdWVzdD05MjE2Cl9vdnMtdnNjdGwgc2V0IEludGVyZmFjZSBwZjBocGYgdHlwZT1kcGRrCgojIENyZWF0ZSBzd2l0Y2hpbmcgT1ZTIGJyaWRnZSBpbiBiZXR3ZWVuIHRoZSBTQyBtYW5hZ2VkIGJyaWRnZSBhbmQgT1ZOSwpfb3ZzLXZzY3RsIC0tbWF5LWV4aXN0IGFkZC1iciBici1vdm4KX292cy12c2N0bCBzZXQgYnJpZGdlIGJyLW92biBkYXRhcGF0aF90eXBlPW5ldGRldgpfb3ZzLXZzY3RsIHNldCBJbnRlcmZhY2UgYnItb3ZuIG10dV9yZXF1ZXN0PTkwMDAKX292cy12c2N0bCAtLW1heS1leGlzdCBhZGQtcG9ydCBici1vdm4gcGJyb3ZudG9icmRwdQpfb3ZzLXZzY3RsIC0tbWF5LWV4aXN0IGFkZC1wb3J0IGJyLWRwdSBwYnJkcHV0b2Jyb3ZuCgojIFBhdGNoIGJyLW92biBhbmQgYnItZHB1IHRvZ2V0aGVyCl9vdnMtdnNjdGwgc2V0IGludGVyZmFjZSBwYnJvdm50b2JyZHB1IHR5cGU9cGF0Y2ggb3B0aW9uczpwZWVyPXBicmRwdXRvYnJvdm4KX292cy12c2N0bCBzZXQgaW50ZXJmYWNlIHBicmRwdXRvYnJvdm4gdHlwZT1wYXRjaCBvcHRpb25zOnBlZXI9cGJyb3ZudG9icmRwdQo=
```

   

* **Note:** The applied DPUFlavor resource includes an encoded OVS configuration. Here is a decoded description of the applied OVS configuration:

```shell
#!/bin/bash
_ovs-vsctl() {
  ovs-vsctl --no-wait --timeout 15 "$@"
}
_ovs-vsctl set Open_vSwitch . other_config:doca-init=true
_ovs-vsctl set Open_vSwitch . other_config:dpdk-max-memzones=50000
_ovs-vsctl set Open_vSwitch . other_config:hw-offload=true
_ovs-vsctl set Open_vSwitch . other_config:pmd-quiet-idle=true
_ovs-vsctl set Open_vSwitch . other_config:max-idle=20000
_ovs-vsctl set Open_vSwitch . other_config:max-revalidator=5000
_ovs-vsctl --if-exists del-br ovsbr1
_ovs-vsctl --if-exists del-br ovsbr2
_ovs-vsctl --may-exist add-br br-sfc
_ovs-vsctl set bridge br-sfc datapath_type=netdev
_ovs-vsctl set bridge br-sfc fail_mode=secure
_ovs-vsctl --may-exist add-port br-sfc p0
_ovs-vsctl set Interface p0 type=dpdk
_ovs-vsctl set Interface p0 mtu_request=9216
_ovs-vsctl set Port p0 external_ids:dpf-type=physical
_ovs-vsctl --may-exist add-port br-sfc p1
_ovs-vsctl set Interface p1 type=dpdk
_ovs-vsctl set Interface p1 mtu_request=9216
_ovs-vsctl set Port p1 external_ids:dpf-type=physical

# Activate DOCA for OVNK
_ovs-vsctl set Open_vSwitch . external-ids:ovn-bridge-datapath-type=netdev
_ovs-vsctl set Open_vSwitch . other_config:hw-offload-ct-unidir-udp-enabled=true

# setup ovnkube managed bridge, br-dpu (this corresponds to br-ex on ovnk docs)
_ovs-vsctl --may-exist add-br br-dpu
_ovs-vsctl br-set-external-id br-dpu bridge-id br-dpu
_ovs-vsctl br-set-external-id br-dpu bridge-uplink pbrdputobrovn
_ovs-vsctl set bridge br-dpu datapath_type=netdev
_ovs-vsctl set Interface br-dpu mtu_request=9000
_ovs-vsctl --may-exist add-port br-dpu pf0hpf
_ovs-vsctl set Interface pf0hpf mtu_request=9216
_ovs-vsctl set Interface pf0hpf type=dpdk

# Create switching OVS bridge in between the SC managed bridge and OVNK
_ovs-vsctl --may-exist add-br br-ovn
_ovs-vsctl set bridge br-ovn datapath_type=netdev
_ovs-vsctl set Interface br-ovn mtu_request=9000
_ovs-vsctl --may-exist add-port br-ovn pbrovntobrdpu
_ovs-vsctl --may-exist add-port br-dpu pbrdputobrovn

# Patch br-ovn and br-dpu together
_ovs-vsctl set interface pbrovntobrdpu type=patch options:peer=pbrdputobrovn
_ovs-vsctl set interface pbrdputobrovn type=patch options:peer=pbrovntobrdpu
```

2. Apply the resource file:

```shell
envsubst < dpu-flavor.yaml | oc apply -f -                
# Example output               
dpuflavor.provisioning.dpu.nvidia.com/flavor created  
```

##### 6.5.4 BFB

The BFB resource defines the DPU image (BFB \- Bluefield Bitstream) to download and placed on a shared storage for future DPU provisioning.

1. Create the BFB resource file:

```shell
apiVersion: provisioning.dpu.nvidia.com/v1alpha1
kind: BFB
metadata:
  name: bf-bundle
  namespace: dpf-operator-system
spec:
  url: $BFB_URL 
```

2. Apply the resource file:

```shell
envsubst < bfb.yaml | oc apply -f -

# Expected output:
bfb.provisioning.dpu.nvidia.com/bf-bundle created
```

3. Verify the BFB image was downloaded successfully and Ready:

```shell
oc get bfbs.provisioning.dpu.nvidia.com -n dpf-operator-system bf-bundle -o yaml |grep phase

# Example output
phase: Ready
```

##### 6.5.5 DPUDeployment

DPUDeployment CR is the main orchestration object that connects DPUServices with specific BFBs and DPUFlavors, defines DPUSets for DPU provisioning, and configures service chains to deploy services across DPUs.

1. Create the DPUDeployment resource file:

```shell
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUDeployment
metadata:
  name: dpudeployment
  namespace: dpf-operator-system
spec:
  dpus:
    bfb: bf-bundle
    flavor: flavor
    dpuSets:
    - nameSuffix: "dpuset1"
      nodeSelector:
        matchLabels:
          feature.node.kubernetes.io/dpu-enabled: ""
  services:
    hbn:
      serviceTemplate: hbn
      serviceConfiguration: hbn
    ovn:
      serviceTemplate: ovn
      serviceConfiguration: ovn
  serviceChains:
    switches:
      - ports:
        - serviceInterface:
            matchLabels:
              uplink: p0
        - service:
            name: hbn
            interface: p0_if
      - ports:
        - serviceInterface:
            matchLabels:
              uplink: p1
        - service:
            name: hbn
            interface: p1_if
      - ports:
        - serviceInterface:
            matchLabels:
              port: ovn
        - service:
            name: hbn
            interface: pf2dpu2_if
```

   

2. Apply the resource file:

```shell
oc apply -f dpudeployment.yaml

# Expected output:
dpudeployment.svc.dpu.nvidia.com/dpudeployment created
```

   

3. Verify DPUDeployment state:

```shell
oc get DPUDeployment -n dpf-operator-system

# Example output
NAME            READY   PHASE     AGE
dpudeployment   False   Pending   2m32s
```

* **Note:** “Pending” state is expected at this phase.

#### 

#### 6.6 Create DPU Services Resources

##### 6.6.1 HBN DPU Service

1. Create HBN DPU Service resources file:

```shell
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceTemplate
metadata:
  name: hbn
  namespace: dpf-operator-system
spec:
  deploymentServiceName: "hbn"
  helmChart:
    source:
      repoURL: $HELM_REGISTRY_REPO_URL
      version: "1.0.3"
      chart: doca-hbn
    values:
      image:
        repository: $HBN_NGC_IMAGE_URL
        tag: 3.2.0-doca3.2.0
      resources:
        memory: 6Gi
        nvidia.com/bf_sf: 3
---

apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceConfiguration
metadata:
  name: hbn
  namespace: dpf-operator-system
spec:
  deploymentServiceName: "hbn"
  serviceConfiguration:
    serviceDaemonSet:
      annotations:
        k8s.v1.cni.cncf.io/networks: |-
          [
          {"name": "iprequest", "interface": "ip_lo", "cni-args": {"poolNames": ["loopback"], "poolType": "cidrpool"}},
          {"name": "iprequest", "interface": "ip_pf2dpu2", "cni-args": {"poolNames": ["pool1"], "poolType": "cidrpool", "allocateDefaultGateway": true}}
          ]
    helmChart:
      values:
        configuration:
          perDPUValuesYAML: |
            - hostnamePattern: "*"
              values:
                bgp_peer_group: hbn
          startupYAMLJ2: |
            - header:
                model: BLUEFIELD
                nvue-api-version: nvue_v1
                rev-id: 1.0
                version: HBN 2.4.0
            - set:
                interface:
                  lo:
                    ip:
                      address:
                        {{ ipaddresses.ip_lo.ip }}/32: {}
                    type: loopback
                  p0_if,p1_if:
                    type: swp
                    link:
                      mtu: 9216
                  pf2dpu2_if:
                    ip:
                      address:
                        {{ ipaddresses.ip_pf2dpu2.cidr }}: {}
                    type: swp
                    link:
                      mtu: 9216
                router:
                  bgp:
                    autonomous-system: {{ ( ipaddresses.ip_lo.ip.split(".")[3] | int ) + 65101 }}
                    enable: on
                    graceful-restart:
                      mode: full
                    router-id: {{ ipaddresses.ip_lo.ip }}
                vrf:
                  default:
                    router:
                      bgp:
                        address-family:
                          ipv4-unicast:
                            enable: on
                            redistribute:
                              connected:
                                enable: on
                          ipv6-unicast:
                            enable: on
                            redistribute:
                              connected:
                                enable: on
                        enable: on
                        neighbor:
                          p0_if:
                            peer-group: {{ config.bgp_peer_group }}
                            type: unnumbered
                          p1_if:
                            peer-group: {{ config.bgp_peer_group }}
                            type: unnumbered
                        path-selection:
                          multipath:
                            aspath-ignore: on
                        peer-group:
                          {{ config.bgp_peer_group }}:
                            remote-as: external
  interfaces:
    - name: p0_if
      network: mybrhbn
    - name: p1_if
      network: mybrhbn
    - name: pf2dpu2_if
      network: mybrhbn
```

### 

2. Apply the resources file:

```shell
envsubst < hbn.yaml | oc apply -f -

# Example output
dpuservicetemplate.svc.dpu.nvidia.com/hbn created
dpuserviceconfiguration.svc.dpu.nvidia.com/hbn created
```

##### 6.6.2 OVN-Kubernetes DPU Service

1. Create OVN-Kubernetes DPU Service resources file:

```shell
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceTemplate
metadata:
  name: ovn
  namespace: dpf-operator-system
spec:
  deploymentServiceName: "ovn"
  helmChart:
    source:
      repoURL: $OVN_TEMPLATE_CHART_URL
      chart: ovn-kubernetes-chart
      version: $TAG-ocp
    values:
      commonManifests:
        enabled: true
      dpuManifests:
        enabled: true
        cniBinDir: /var/lib/cni/bin/
        cniConfDir: /run/multus/cni/net.d
        image:
          repository: $OVN_KUBERNETES_IMAGE_REPO
          tag: $OVN_KUBERNETES_IMAGE_TAG
      leaseNamespace: "openshift-ovn-kubernetes"
      gatewayOpts: "--gateway-interface=br-dpu"

---

apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceConfiguration
metadata:
  name: ovn
  namespace: dpf-operator-system
spec:
  deploymentServiceName: "ovn"
  serviceConfiguration:
    helmChart:
      values:
        k8sAPIServer: https://$HOST_CLUSTER_API:$TARGETCLUSTER_API_SERVER_PORT
        podNetwork: $POD_CIDR/23
        serviceNetwork: $SERVICE_CIDR
        mtu: 8940
        dpuManifests:
          kubernetesSecretName: "ovn-dpu"
          vtepCIDR: $VTEP_CIDR
          hostCIDR: $TARGETCLUSTER_NODE_CIDR
          ipamPool: "pool1"
          ipamPoolType: "cidrpool"
          ipamVTEPIPIndex: 0
          ipamPFIPIndex: 1
          cniBinDir: "/var/lib/cni/bin/"
          cniConfDir: "/run/multus/cni/net.d"
```

### 

2. Apply the resources file:

```shell
envsubst < ovn-k.yaml | oc apply -f -

# Example output

dpuservicetemplate.svc.dpu.nvidia.com/ovn created
dpuserviceconfiguration.svc.dpu.nvidia.com/ovn created
```

##### 6.6.3: OVN-Kubernetes DPUServiceCredentialRequest 

This resource enables the OVN-Kubernetes DPU service on the hosted cluster to authenticate with the management cluster's API server. The ClusterRoleBinding grants the required permissions for OVN node network operations.                                    

1. Create DPUServiceCredentialRequest resource file:

```shell
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceCredentialRequest
metadata:
  name: ovn-dpu
  namespace: dpf-operator-system 
spec:
  serviceAccount:
    name: ovn-kubernetes-node-dpu-service
    namespace: openshift-ovn-kubernetes
  duration: 24h
  type: tokenFile
  secret:
    name: ovn-dpu
    namespace: dpf-operator-system
  metadata:
    labels:
      dpu.nvidia.com/image-pull-secret: ""
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ovn-kubernetes-node-limited-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-ovn-kubernetes-node-limited
subjects:
- kind: ServiceAccount
  name: ovn-kubernetes-node-dpu-service
  namespace: openshift-ovn-kubernetes
```

2. Apply the resource file:

```shell
oc apply -f dpucredentialreq.yaml 

# Expected output:
dpuservicecredentialrequest.svc.dpu.nvidia.com/ovn-dpu created
clusterrolebinding.rbac.authorization.k8s.io/ovn-kubernetes-node-limited-binding created
```

##### 6.6.4: DPUServiceInterfaces: Physical 

The DPUServiceInterfaces defines interface objects that can be specified in Service Chains, in this case the physical ports on the DPU.

1. Create DPUServiceInterfaces resource file:

```shell
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceInterface
metadata:
  name: p0
  namespace: dpf-operator-system
spec:
  template:
    spec:
      template:
        metadata:
          labels:
            uplink: "p0"
        spec:
          interfaceType: physical
          physical:
            interfaceName: p0
---
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceInterface
metadata:
  name: p1
  namespace: dpf-operator-system
spec:
  template:
    spec:
      template:
        metadata:
          labels:
            uplink: "p1"
        spec:
          interfaceType: physical
          physical:
            interfaceName: p1
```

2. Apply the resource file:

```shell
oc apply -f physical-if.yaml 

# Expected output:
dpuserviceinterface.svc.dpu.nvidia.com/p0 created
dpuserviceinterface.svc.dpu.nvidia.com/p1 created
```

##### 6.6.5: DPUServiceInterface: OVN-Kubernetes 

The DPUServiceInterfaces defines interface objects that can be specified in Service Chains, in this case the interface used by OVN-Kubernetes on Host workloads.

1. Create DPUServiceInterfaces resource file:  

```shell
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceInterface
metadata:
  name: ovn
  namespace: dpf-operator-system
spec:
  template:
    spec:
      template:
        metadata:
          labels:
            port: ovn
        spec:
          interfaceType: ovn
```

2. Apply the resource file:

```shell
oc apply -f ovnk-if.yaml

# Expected output: 
dpuserviceinterface.svc.dpu.nvidia.com/ovn created
```

##### 6.6.6: DPUServiceNADs 

DPUServiceNAD resources define the network attachments available to DPU services on the hosted cluster. Each NAD maps to an OVS bridge on the DPU and specifies the resource type, IPAM mode, and MTU configuration.

* **mybrhbn** \- Maps to the br-hbn bridge, used by the HBN service. IPAM is disabled as IP allocation is handled by DPUServiceIPAM.  
* **mybrsfc** \- Maps to the br-sfc bridge, used for service function chaining. IPAM is enabled.  
* Both use sub-function (sf) resource type and Jumbo MTU (9216).

1. Create DPUServiceNAD resources file:

```shell
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceNAD
metadata:
  name: mybrhbn
  namespace: dpf-operator-system
spec:
  resourceType: sf
  ipam: false
  bridge: "br-hbn"
  serviceMTU: 9216
--- 
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceNAD
metadata:
  name: mybrsfc
  namespace: dpf-operator-system
spec:
  resourceType: sf
  ipam: true
  bridge: "br-sfc"
  serviceMTU: 9216
```

   

2. Apply the resource file:

```shell
oc apply -f dpuservice-nad.yaml

# Expected output: 
dpuservicenad.svc.dpu.nvidia.com/mybrhbn created
dpuservicenad.svc.dpu.nvidia.com/mybrsfc created
```

##### 6.6.7: DPUServiceIPAM 

The DPUServiceIPAM CR is used to set up IP Address Management for DPU Services.

1. Create the DPUServiceIPAM resource file:

```shell
---
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceIPAM
metadata:
  name: pool1
  namespace: dpf-operator-system
spec:
  ipv4Network:
    network: $VTEP_CIDR
    gatewayIndex: 3
    prefixSize: 29
---
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceIPAM
metadata:
  name: loopback
  namespace: dpf-operator-system
spec:
  ipv4Network:
    network: "11.0.0.0/24"
    prefixSize: 32

```

2. Apply the resource file:

```shell
envsubst < dpuservice-ipam.yaml | oc apply -f -

# Expected output:
dpuserviceipam.svc.dpu.nvidia.com/pool1 created
dpuserviceipam.svc.dpu.nvidia.com/loopback created
```

#### 6.7 Verify DPU Services Resources

Follow the steps to verify DPU Services Resources state:

```shell
## Ensure the DPUServices are created and have been reconciled.
oc wait --for=condition=ApplicationsReconciled --namespace dpf-operator-system dpuservices -l svc.dpu.nvidia.com/owned-by-dpudeployment=dpf-operator-system_dpudeployment                                                                                  

# Example output
dpuservice.svc.dpu.nvidia.com/hbn-v4ffl condition met                                                                                              
dpuservice.svc.dpu.nvidia.com/ovn-xmkjj condition met     

## Ensure the DPUServiceIPAMs have been reconciled
oc wait --for=condition=DPUIPAMObjectReconciled --namespace dpf-operator-system dpuserviceipam --all    

# Example output
dpuserviceipam.svc.dpu.nvidia.com/loopback condition met                                                                                           
dpuserviceipam.svc.dpu.nvidia.com/pool1 condition met
 
## Ensure the DPUServiceInterfaces have been reconciled
oc wait --for=condition=ServiceInterfaceSetReconciled --namespace dpf-operator-system dpuserviceinterface --all  
                                                                                                                                     
# Example output
dpuserviceinterface.svc.dpu.nvidia.com/hbn-p0-if-qqlws condition met                                                                               
dpuserviceinterface.svc.dpu.nvidia.com/hbn-p1-if-9x29m condition met                                                                               
dpuserviceinterface.svc.dpu.nvidia.com/hbn-pf2dpu2-if-vnjtl condition met                                                                          
dpuserviceinterface.svc.dpu.nvidia.com/ovn condition met                                                                                           
dpuserviceinterface.svc.dpu.nvidia.com/p0 condition met                                                                                            
dpuserviceinterface.svc.dpu.nvidia.com/p1 condition met  

## Ensure the DPUServiceChains have been reconciled
oc wait --for=condition=ServiceChainSetReconciled --namespace dpf-operator-system dpuservicechain --all

# Example output
dpuservicechain.svc.dpu.nvidia.com/dpudeployment-sqx9h condition met

##  Ensure DPUServices status as below (nvidia-k8s-ipam is expected to be Pending at this phase)
oc get dpuservice -n dpf-operator-system 

NAME                            READY   PHASE     AGE
flannel                         True    Success   26h
hbn-gffmv                       True    Success   25m
nvidia-k8s-ipam                 False   Pending   26h
ovn-f49zx                       True    Success   17m
ovs-cni                         True    Success   26h
servicechainset-controller      True    Success   26h
servicechainset-rbac-and-crds   True    Success   138m
sfc-controller                  True    Success   26h
sriov-device-plugin             True    Success   26h
```

#### 

## **Chapter 7:** OVN-Kubernetes CNI Adjustments 

Few operational steps are required to adjust the cluster CNI for supporting DPU acceleration.

#### 7.1 Enable OVN-Kubernetes Resource Injector

The OVN Kubernetes resource injector uses a MutatingAdmissionPolicy to automatically inject SR-IOV VF resource requests and network attachment annotations into each pod scheduled to a worker node. This policy-based admission control mechanism is part of the same helm chart as the other OVN-Kubernetes CNI components.   
**Note:** This step requires the MutatingAdmissionPolicy FeatureGate to be enabled and control plane nodes to be patched with resource capacity (Cluster Configuration section 4.2).

1. Install the OVN Kubernetes resource injector using the helm:

```shell
helm install ovn-kubernetes-resource-injector \
  $OVN_TEMPLATE_CHART_URL/ovn-kubernetes-chart \
  --namespace openshift-ovn-kubernetes \
  --version ${TAG}-ocp \
  --set ovn-kubernetes-resource-injector.enabled=true \
  --set resourceName=openshift.io/bf3-p0-vfs \
  --set nodeWithDPUManifests.enabled=false \
  --set nodeWithoutDPUManifests.enabled=false \
  --set dpuManifests.enabled=false \
  --set controlPlaneManifests.enabled=false \
  --set commonManifests.enabled=false \
  --wait

# Example output

NAME: ovn-kubernetes-resource-injector
LAST DEPLOYED: Sun Nov  2 17:10:29 2025
NAMESPACE: openshift-ovn-kubernetes
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

2. Verify the resource injector admission policy was applied:

```shell
oc get mutatingadmissionpolicies

# Example output
NAME                                                 MUTATIONS   PARAMKIND   AGE
ovn-kubernetes-resource-injector-resource-injector   2           <unset>     3m13s
```

#### 

#### 7.2 Enable OVN-Kubernetes DPU-Host Mode 

Worker nodes with DPUs that are using accelerated OVN-Kubernetes CNI must be running in a mode named “DPU-Host”. To enable this mode on the worker nodes, configMap resources applying the following configurations should be created on the Management Cluster:

* Disable network-node-identity  
* Enable DPU acceleration support

1. Create the configMap resources file:

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: network-node-identity
  namespace: openshift-network-operator
data:
  enabled: "false"
---
apiVersion: v1
kind: ConfigMap
metadata:
    name: hardware-offload-config
    namespace: openshift-network-operator
data:
    dpu-host-mode-label: "feature.node.kubernetes.io/dpu-enabled="
    mgmt-port-resource-name: "openshift.io/bf3-p0-vfs-mgmt"
```

   

2. Apply the resource file:

```shell
oc apply -f ovnk-acc.yaml

# Expected output: 
configmap/network-node-identity created
configmap/hardware-offload-config created
```

## **Chapter 8:** Adding Worker Nodes and Provision DPUs

#### 8.1 Prerequisites

* Before adding worker nodes, ensure:  
  * Management cluster is fully operational  
  * Hosted cluster is created and running  
  * DPF operator is installed  
  * Cluster OVN-Kubernetes CNI is adjusted for DPU acceleration  
  * Worker node Hardware and Network prerequisites are met (See Prerequisites Chapter)  
  * DPU Secure Boot is disabled (See Prerequisites Chapter)  
  * iDRAC/BMC access to Worker node is available

#### 8.2 Add Worker Nodes

You can follow two different methods to add a worker node to your management OpenShift cluster:

* Using Assisted Installer (User Interface)  
* Using Baremetal Operator (Automated using cluster APIs)

##### 8.2.1 **Option 1:** Adding Worker Nodes Using Assisted Installer 

1. Log into [https://console.redhat.com/openshift](https://console.redhat.com/openshift)   
2. Select your cluster  
3. Click "Add hosts"   
4. Create and download a Discovery ISO   
5. Upload the ISO to the the worker node (with DPU) using IDRAC/BMC and boot the server from it  
6. Wait for the node to boot from the discovery ISO  
7. Once the node is discovered, collect the DPU Interface name, as it is used to populate DPU\_P0 variable in previous steps  
   ![][image2]  
8. Select “Install ready host” to start the installation of the worker from Assisted Installer UI and monitor the installation progress   
9. Once the worker nodes are “Installed” proceed to next step

##### 8.2.2 **Option 2:** Adding Worker Nodes Using BareMetal Operator  

1. References: [https://github.com/openshift/cluster-baremetal-operator](https://github.com/openshift/cluster-baremetal-operator)  
2. Prerequisites  
   1. Baremetal Operator installed on the management Openshift cluster  
   2. Physical Worker servers with BMC/iDRAC/iLO access (Redfish-compatible)  
   3. Network connectivity from the management cluster to worker BMC   
   4. BMC IP address and access credentials for each server  
   5. MAC address of the management network interface for each server  
   6. Name of the disk root device for each server  
3. Set the following environment variables

```shell
export BMC_IP= <x.x.x.x> # allocated from the management cluster subnet 
export BMC_USER=<username>  
export BMC_PASSWORD=<password> 
export WORKER_NAME=<worker-X> # worker-01
export BOOT_MAC=<mgmt_int_mac> # aa:bb:cc:dd:ee:01 - MAC of the oob management interface 
export ROOT_DEVICE=<root_device> # /dev/sda
```

4. Verify BMC Connectivity

```shell
# Test BMC access from one of the control plane nodes
ping $BMC_IP
curl -k https://$BMC_IP/redfish/v1/
# Test credentials
curl -k -u $BMC_USER:$BMC_PASSWORD https://$BMC_IP/redfish/v1/Systems
```

5. Verify Baremetal Operator is Available

```shell
# The Baremetal Operator is pre-installed in OpenShift. Verify it is available:
oc get clusteroperator baremetal
Expected output:
  NAME        VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
  baremetal   4.20.x    True        False         False      ...
```

6. Create and apply the Provisioning resource:

```shell
# Create or update the Provisioning configuration to disable the provisioning network (required for virtual media boot):
apiVersion: metal3.io/v1alpha1
kind: Provisioning
metadata:
  name: provisioning-configuration
spec:
  provisioningNetwork: "Disabled"
  watchAllNamespaces: false

# Apply the configuration:
oc apply -f provisioning.yaml
```

**Important:** When provisioningNetwork is set to Disabled, servers boot using Redfish virtual media instead of PXE.

7. Create and apply BMC Credentials Secret resource:

```shell
# For each worker node, create a Secret containing the BMC credentials:
apiVersion: v1
kind: Secret
metadata:
  name: ${WORKER_NAME}-bmc-secret
  namespace: openshift-machine-api
type: Opaque
stringData:
  username: $BMC_USER
  password: $BMC_PASSWORD 

# Apply the secret:
envsubst < bmc-secret.yaml | oc apply -f -
```

   

8. Create and apply the BaremetalHost resource:

```shell
# Create a BaremetalHost CR for each worker node:
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: $WORKER_NAME
  namespace: openshift-machine-api
  annotations:
    inspect.metal3.io: disabled
spec:
  online: true
  bootMACAddress: $BOOT_MAC
  bmc:
    address: redfish-virtualmedia+https://$BMC_IP
    credentialsName: $WORKER_NAME-bmc-secret
    disableCertificateVerification: true
  customDeploy:
    method: install_coreos
  rootDeviceHints:
    deviceName: $ROOT_DEVICE
  userData:
    name: worker-user-data-managed
    namespace: openshift-machine-api

# Apply the BaremetalHost:
envsubst < baremetalhost.yaml | oc apply -f -
```

9. Monitor provisioning progress:

```shell
# Watch the BaremetalHost status:
oc get bmh -n openshift-machine-api -w

# Expected progression:
NAME        STATE         CONSUMER   ONLINE   ERROR   AGE
worker-01   registering              true             10s
worker-01   available                true             30s
worker-01   provisioning             true             1m
worker-01   provisioned              true             10m
```

#### 8.3 Approve Worker Nodes Certificate Requests 

```shell
# Watch for pending CSRs 
oc get csr -w
# Approve all pending CSRs:
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve

# Example output:
certificatesigningrequest.certificates.k8s.io/csr-27bgq approved
certificatesigningrequest.certificates.k8s.io/csr-69g65 approved
certificatesigningrequest.certificates.k8s.io/csr-7r862 approved
certificatesigningrequest.certificates.k8s.io/csr-f5vk7 approved

# Repeat step 2 until no pending CSRs remain. Each node typically generates multiple CSRs.

# Verify the worker node joined successfully to the cluster
oc get nodes

# Example output:
NAME               STATUS     ROLES                         AGE     VERSION
host-worker1       NotReady   worker                        68s     v1.33.6
host-worker2       NotReady   worker                        75s     v1.33.6
master-0           Ready      control-plane,master,worker   4d22h   v1.33.6
master-1           Ready      control-plane,master,worker   4d21h   v1.33.6
master-2           Ready      control-plane,master,worker   4d22h   v1.33.6
```

**Notes**: 

* The worker node will become Ready only after the DPU provisioning process is fully completed and all OVN-K CNI components (on the host and on the DPU) are up.  
* Do NOT proceed to next steps until ALL pending CSRs are approved

#### 8.4 Verify DPU Provisioning is started automatically

```shell
# Watch for DPU object creation (happens automatically via DPUSet)
oc get dpu -n dpf-operator-system -w

# The DPUSet controller will:
# 1. Detect nodes with dpu-enabled="" label
# 2. Create a DPU object for each labeled node
# 3. Start the DPU provisioning process

# DPU Provisioning main stages (as reflected in DPU object state):
# - Initializing: DPU object created
# - OS Installing: BFB installation in progress 
# - Rebooting: Host and DPU reset
# - DPU Cluster Config: DPU kubernetes node join procedure in progress (manual csr approval is required - see instructions below)

#   ** Proceed to the next section to approve CSRs when this state is reached **

# - Host Network Configuration: Networking configuration adjustments on host 
# - Ready: DPU successfully provisioned and ready to use
# - Error: Provisioning failed (check events/conditions)

# Monitor DPU provisioning progress
oc -n dpf-operator-system exec deploy/dpf-operator-controller-manager -- /dpfctl describe dpudeployments

# For details you can check the DPU object state or follow provisioning logs
oc describe dpu -n dpf-operator-system <dpu-name>
oc logs -n dpf-operator-system -l dpf.nvidia.com/dpu=<dpu-name> -f
```

#### 8.5 Create Authorization Configuration for the Hosted Cluster Components

DPF services running on DPU nodes require privileged access to host networking and devices. These ClusterRoleBindings grant the privileged SCC to each DPF service account in the hosted cluster.

1. Switch to hosted cluster context:

```shell
export KUBECONFIG=$HOSTED_CLUSTER_NAME.kubeconfig
```

   

2. Create the authorization resources file: 

```shell
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: flannel-scc-rolebinding
  labels:
    app.kubernetes.io/component: rbac
    app.kubernetes.io/part-of: dpu-services
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
subjects:
  - kind: ServiceAccount
    name: flannel
    namespace: dpf-operator-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: doca-hbn-scc-rolebinding
  labels:
    app.kubernetes.io/component: rbac
    app.kubernetes.io/part-of: dpu-services
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
subjects:
  - kind: ServiceAccount
    name: default
    namespace: dpf-operator-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ovn-dpu-scc-rolebinding
  labels:
    app.kubernetes.io/component: rbac
    app.kubernetes.io/part-of: dpu-services
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
subjects:
  - kind: ServiceAccount
    name: ovn-dpu
    namespace: dpf-operator-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ovn-kubernetes-node-scc-rolebinding
  labels:
    app.kubernetes.io/component: rbac
    app.kubernetes.io/part-of: dpu-services
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
subjects:
  - kind: Group
    name: system:serviceaccounts:dpf-operator-system
    apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sriov-device-plugin-scc-rolebinding
  labels:
    app.kubernetes.io/component: rbac
    app.kubernetes.io/part-of: dpu-services
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
subjects:
  - kind: ServiceAccount
    name: sriov-device-plugin
    namespace: dpf-operator-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nvidia-k8s-ipam-scc-rolebinding
  labels:
    app.kubernetes.io/component: rbac
    app.kubernetes.io/part-of: dpu-services
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
subjects:
  - kind: ServiceAccount
    name: cluster-nvidia-k8s-ipam-node
    namespace: dpf-operator-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ovs-cni-scc-rolebinding
  labels:
    app.kubernetes.io/component: rbac
    app.kubernetes.io/part-of: dpu-services
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
subjects:
  - kind: ServiceAccount
    name: cluster-ovs-cni-marker
    namespace: dpf-operator-system
```

3. Apply the resources file on the hosted cluster:

```shell
oc apply -f dpu-cluster-scc.yaml 

# Example output:
clusterrolebinding.rbac.authorization.k8s.io/flannel-scc-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/doca-hbn-scc-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/ovn-dpu-scc-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/ovn-kubernetes-node-scc-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/sriov-device-plugin-scc-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/nvidia-k8s-ipam-scc-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/ovs-cni-scc-rolebinding create

```

#### 8.6 Approve DPU Nodes Certificate Requests to Complete Provisioning

When the DPU object state reaches the "DPU Cluster Config" phase, the DPU worker node will attempt to join the hosted cluster. Manual approval of its certificate requests is required to complete the provisioning process.

1. While you are still on hosted cluster context (kubeconfig), approve DPU nodes CSRs:

```shell
# Watch for CSRs from the DPU node
oc get csr -w
# The DPU node name typically follows pattern: <host-worker-node-name>-<dpu-serial-number>

# Approve the DPU CSRs (multiple per worker)
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
# Example output:
certificatesigningrequest.certificates.k8s.io/csr-6jx22 approved
certificatesigningrequest.certificates.k8s.io/csr-tb6nd approved

## Repeat approval for the newly created csrs until node is added to the cluster

# Verify the DPU node joined the hosted cluster and appear as Ready
oc get nodes 

# Example output:
NAME                            STATUS   ROLES    AGE     VERSION
host-worker1-mt2413xz0b65   Ready    worker   2m48s   v1.33.5
host-worker2-mt2413xz0aw6   Ready    worker   2m45s   v1.33.5
```

   **Note:** Once the DPU nodes join the hosted cluster, DPU Provisioning process will proceed.

#### 8.7 Verify Full System Readiness

At this point DPU Provisioning will proceed until completion. Follow the steps below to verify full system readiness.

1. Switch BACK to Management Cluster context (kubeconfig):

```shell
export KUBECONFIG=mgmt-kubeconfig
```

   

2. Verify the status of critical resources:

```shell
# Monitor worker nodes status (should transition from NotReady to Ready once all OVN-K CNI components are up on both host and DPU)
oc get node
# Example output:
NAME               STATUS   ROLES                         AGE     VERSION
host-worker1       Ready    worker                        57m     v1.33.6
host-worker2       Ready    worker                        57m     v1.33.6
master-0           Ready    control-plane,master,worker   4d23h   v1.33.6
master-1           Ready    control-plane,master,worker   4d22h   v1.33.6
master-2           Ready    control-plane,master,worker   4d23h   v1.33.6

# Verify SR-IOV VFs are registered as k8s node Resource on the worker nodes
oc get nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/control-plane' -o json | \
  jq '.items[] | {name: .metadata.name, capacity: .status.capacity."openshift.io/bf3-p0-vfs", allocatable: .status.allocatable."openshift.io/bf3-p0-vfs"}'
# Example output:
{
  "name": "host-worker1",
  "capacity": "44",
  "allocatable": "44"
}
{
  "name": "host-worker2",
  "capacity": "44",
  "allocatable": "44"
}


# Verify DPU Service status
oc get dpuservices -n dpf-operator-system
# Example output:
NAME                            READY   PHASE     AGE
flannel                         True    Success   4h31m
hbn-cmlph                       True    Success   3h47m
nvidia-k8s-ipam                 True    Success   4h31m
ovn-xcv67                       True    Success   3h47m
ovs-cni                         True    Success   4h31m
servicechainset-controller      True    Success   4h31m
servicechainset-rbac-and-crds   True    Success   4h30m
sfc-controller                  True    Success   4h31m
sriov-device-plugin             True    Success   4h31m


# For more detailed visibility of the DPU Services use
oc -n dpf-operator-system exec deploy/dpf-operator-controller-manager -- /dpfctl describe all --show-resources=dpuservice --grouping=false
```

## 

## **Chapter 9:** Traffic Validation Test

To verify the full deployment and the functionality of DPU Services and Service Chains we will execute a basic traffic test between test-workload pods and services deployed across the system on different nodes.

#### 9.1 Create Traffic Test Pods/Services 

1. Create a file defining test pods and services:

```shell
# 1. Namespace
---
apiVersion: v1
kind: Namespace
metadata:
  name: workload

# 2. SCC RoleBinding (grants 'default' ServiceAccount in 'workload' NS access to 'privileged' SCC)
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: privileged-scc-default-sa
  namespace: workload
subjects:
- kind: ServiceAccount
  name: default
  namespace: workload
roleRef:
  kind: ClusterRole
  name: system:openshift:scc:privileged
  apiGroup: rbac.authorization.k8s.io

# 3. Deployments and Services
# Deployment: traffic-test-master
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traffic-test-master
  namespace: workload
  labels:
    app: traffic-test-master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: traffic-test-master
  template:
    metadata:
      labels:
        app: traffic-test-master
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: traffic-test-master
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: nginx
        securityContext:
          privileged: true
          capabilities:
            add:
            - NET_ADMIN
        image: nicolaka/netshoot
        command: ["nc", "-kl", "5000"]
        ports:
        - containerPort: 5000
          name: tcp-server
        resources:
          requests:
            cpu: 1
            memory: 1Gi
          limits:
            cpu: 1
            memory: 1Gi
---
# Service: traffic-test-master
apiVersion: v1
kind: Service
metadata:
  name: traffic-test-master
  namespace: workload
  labels:
    app: traffic-test-master
spec:
  selector:
    app: traffic-test-master
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
---
# Service: traffic-test-master-nodeport
apiVersion: v1
kind: Service
metadata:
  name: traffic-test-master-nodeport
  namespace: workload
  labels:
    app: traffic-test-master
spec:
  type: NodePort
  selector:
    app: traffic-test-master
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
---
# Deployment: traffic-test-worker
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traffic-test-worker
  namespace: workload
  labels:
    app: traffic-test-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: traffic-test-worker
  template:
    metadata:
      labels:
        app: traffic-test-worker
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: traffic-test-worker
      nodeSelector:
        feature.node.kubernetes.io/dpu-enabled: ""
      containers:
      - name: nginx
        securityContext:
          privileged: true
          capabilities:
            add:
            - NET_ADMIN
        image: nicolaka/netshoot
        command: ["nc", "-kl", "5000"]
        ports:
        - containerPort: 5000
          name: tcp-server
        resources:
          requests:
            cpu: 16
            memory: 6Gi
          limits:
            cpu: 16
            memory: 6Gi
---
# Service: traffic-test-worker
apiVersion: v1
kind: Service
metadata:
  name: traffic-test-worker
  namespace: workload
  labels:
    app: traffic-test-worker
spec:
  selector:
    app: traffic-test-worker
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
---
# Service: traffic-test-worker-nodeport
apiVersion: v1
kind: Service
metadata:
  name: traffic-test-worker-nodeport
  namespace: workload
  labels:
    app: traffic-test-worker
spec:
  type: NodePort
  selector:
    app: traffic-test-worker
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
---
# Deployment: traffic-test-worker-hostnetwork
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traffic-test-worker-hostnetwork
  namespace: workload
  labels:
    app: traffic-test-worker-hostnetwork
spec:
  replicas: 2
  selector:
    matchLabels:
      app: traffic-test-worker-hostnetwork
  template:
    metadata:
      labels:
        app: traffic-test-worker-hostnetwork
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: traffic-test-worker-hostnetwork
      nodeSelector:
        feature.node.kubernetes.io/dpu-enabled: ""
      hostNetwork: true
      containers:
      - name: nginx
        securityContext:
          privileged: true
          capabilities:
            add:
            - NET_ADMIN
        image: nicolaka/netshoot
        command: ["nc", "-kl", "5000"]
        ports:
        - containerPort: 5000
          name: tcp-server
        resources:
          requests:
            cpu: 1
            memory: 1Gi
          limits:
            cpu: 1
            memory: 1Gi
---
# Service: traffic-test-worker-hostnetwork
apiVersion: v1
kind: Service
metadata:
  name: traffic-test-worker-hostnetwork
  namespace: workload
  labels:
    app: traffic-test-worker-hostnetwork
spec:
  selector:
    app: traffic-test-worker-hostnetwork
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
---
# Service: traffic-test-worker-hostnetwork-nodeport
apiVersion: v1
kind: Service
metadata:
  name: traffic-test-worker-hostnetwork-nodeport
  namespace: workload
  labels:
    app: traffic-test-worker-hostnetwork
spec:
  type: NodePort
  selector:
    app: traffic-test-worker-hostnetwork
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000

```

2. Apply the file:

```shell
oc apply -f traffic-pods.yaml

# Example output:
rolebinding.rbac.authorization.k8s.io/privileged-scc-default-sa created
deployment.apps/traffic-test-master created
service/traffic-test-master created
service/traffic-test-master-nodeport created
deployment.apps/traffic-test-worker created
service/traffic-test-worker created
service/traffic-test-worker-nodeport created
deployment.apps/traffic-test-worker-hostnetwork created
service/traffic-test-worker-hostnetwork created
service/traffic-test-worker-hostnetwork-nodeport created


# Verify the test pods and services are running on master and worker nodes
oc get pods -n workload -o wide 

# Example output: 
NAME                                               READY   STATUS    RESTARTS   AGE     IP             NODE               NOMINATED NODE   READINESS GATES
traffic-test-master-7448bb5cc-mdftd                1/1     Running   0          2m10s   10.129.0.145   master-2           <none>           <none>
traffic-test-worker-776486fb68-krz54               1/1     Running   0          2m10s   10.128.2.9     host-worker1       <none>           <none>
traffic-test-worker-776486fb68-lz8vf               1/1     Running   0          2m10s   10.131.0.9     host-worker2       <none>           <none>
traffic-test-worker-hostnetwork-596d569d99-cjpns   1/1     Running   0          2m10s   10.0.110.11    host-worker1       <none>           <none>
traffic-test-worker-hostnetwork-596d569d99-x6m7r   1/1     Running   0          2m10s   10.0.110.12    host-worker2       <none>           <none>


oc get svc -n workload
# Example output:
NAME                                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
traffic-test-master                        ClusterIP   172.30.102.123   <none>        5000/TCP         13m
traffic-test-master-nodeport               NodePort    172.30.98.22     <none>        5000:31368/TCP   13m
traffic-test-worker                        ClusterIP   172.30.187.147   <none>        5000/TCP         13m
traffic-test-worker-hostnetwork            ClusterIP   172.30.122.242   <none>        5000/TCP         13m
traffic-test-worker-hostnetwork-nodeport   NodePort    172.30.122.214   <none>        5000:30209/TCP   13m
traffic-test-worker-nodeport               NodePort    172.30.108.72    <none>        5000:32570/TCP   13m
```

   

3. Verify test pods and services are running:

```shell
oc get pods -n workload -o wide 

# Example output: 
NAME                                               READY   STATUS    RESTARTS   AGE     IP             NODE               NOMINATED NODE   READINESS GATES
traffic-test-master-7448bb5cc-mdftd                1/1     Running   0          2m10s   10.129.0.145   master-2           <none>           <none>
traffic-test-worker-776486fb68-krz54               1/1     Running   0          2m10s   10.128.2.9     host-worker1       <none>           <none>
traffic-test-worker-776486fb68-lz8vf               1/1     Running   0          2m10s   10.131.0.9     host-worker2       <none>           <none>
traffic-test-worker-hostnetwork-596d569d99-cjpns   1/1     Running   0          2m10s   10.0.110.11    host-worker1       <none>           <none>
traffic-test-worker-hostnetwork-596d569d99-x6m7r   1/1     Running   0          2m10s   10.0.110.12    host-worker2       <none>           <none>


oc get svc -n workload
# Example output:
NAME                                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
traffic-test-master                        ClusterIP   172.30.102.123   <none>        5000/TCP         13m
traffic-test-master-nodeport               NodePort    172.30.98.22     <none>        5000:31368/TCP   13m
traffic-test-worker                        ClusterIP   172.30.187.147   <none>        5000/TCP         13m
traffic-test-worker-hostnetwork            ClusterIP   172.30.122.242   <none>        5000/TCP         13m
traffic-test-worker-hostnetwork-nodeport   NodePort    172.30.122.214   <none>        5000:30209/TCP   13m
traffic-test-worker-nodeport               NodePort    172.30.108.72    <none>        5000:32570/TCP   13m


```

#### 9.2 Run Traffic Validation Tests

Run a basic connectivity test by pinging between the different pods and services deployed in the “workload” namespace.   
A successful test will indicate that the DPU Services and Chains are configured correctly and working as expected.

1. An example of a ping connectivity test between pods running on different worker nodes:

```
oc -n workload exec -it traffic-test-worker-776486fb68-krz54 -- ping -c 4 10.131.0.9

# Example output: 
PING 10.131.0.9 (10.131.0.9) 56(84) bytes of data.
64 bytes from 10.131.0.9: icmp_seq=1 ttl=62 time=1.61 ms
64 bytes from 10.131.0.9: icmp_seq=2 ttl=62 time=0.876 ms
64 bytes from 10.131.0.9: icmp_seq=3 ttl=62 time=0.510 ms
64 bytes from 10.131.0.9: icmp_seq=4 ttl=62 time=0.421 ms

--- 10.131.0.9 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3028ms
rtt min/avg/max/mdev = 0.421/0.853/1.606/0.466 ms
```

2. An example of a service connectivity test from a pod running on worker node to a service hosted on control plane node:

```
oc -n workload exec -it traffic-test-worker-776486fb68-krz54 -- nc -vz 172.30.102.123 5000

# Example output: 
Connection to 172.30.102.123 5000 port [tcp/*] succeeded!
```

## **Chapter 10:** Basic Troubleshooting

#### 10.1 DPU Provisioning

In case DPU provisioning is not started immediately after adding worker nodes to the cluster, follow the steps below:

* Verify all worker CSRs were approved


```shell
oc get csr 
```


* Verify all DPF Controller pods are running


```shell
oc get pod -n dpf-operator-system
```

* Verify Workers nodes were labeled for DPU provisioning by NFD:

```shell
# Verify the dpu-enabled label is present
oc get nodes -l feature.node.kubernetes.io/dpu-enabled=""


NAME                                       STATUS     ROLES    AGE   VERSION
nvd-srv-22.nvidia.eng.rdu2.dc.redhat.com   NotReady   worker   62s   v1.33.5
nvd-srv-23.nvidia.eng.rdu2.dc.redhat.com   NotReady   worker   66s   v1.33.5
```

* Check BFB object to make sure the BFB image was downloaded and ready


```shell
oc describe bfb -n dpf-operator-system   bf-bundle
```

* Check DPUDeployment object status, look for information about  
  * BFB object state  
  * DPUServiceTemplate objects state  
  * DPUServiceConfiguration objects state


```shell
oc get dpudeployments -n dpf-operator-system   dpudeployment -o yaml

oc -n dpf-operator-system exec deploy/dpf-operator-controller-manager -- /dpfctl describe dpudeployments
```

#### 10.2 DPU Object State 

In case DPU Objects stay in "DPU Cluster Config" state, check for pending CSRs in hosted cluster:

```shell
# Switch to hosted cluster context
export KUBECONFIG=/path/to/hosted-cluster.kubeconfig
# Check for Pending CSRs and approve it 
oc get csr -A
```

#### 10.3 Management cluster nodes state 

In case management cluster nodes do not become “Ready” after DPU provisioning is completed, check all OVN-K CNI components are up:

```shell
# Switch to management cluster context
export KUBECONFIG=/path/to/management-cluster.kubeconfig
# Check OVN-K pods on the x86 workers and control plane nodes 
oc get pods -n openshift-ovn-kubernetes -o wide 

# Switch to hosted cluster context
export KUBECONFIG=/path/to/hosted-cluster.kubeconfig
# Check OVN-K pods on the DPU workers 
oc get pods -n openshift-ovn-kubernetes -o wide 
```

## **Chapter 11:** Appendix

#### 11.1 Unsupported OVN-Kubernetes Features  

**IMPORTANT:** The following OVN-K Features are currently not supported as part of the CNI used in this Technical Preview:

* UDN  
* Egress IP  
* ANP  
* Hybrid Overlay  
* Egress Firewall  
* QoS  
* BGP  
* Multicast  
* Local Gw Mode

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>