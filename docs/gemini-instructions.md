Integrating a Data Processing Unit (DPU) worker node—like an NVIDIA BlueField-3 running NVIDIA's Data Platform Framework (DPF)—into a Single Node OpenShift (SNO) baremetal environment involves shifting the networking layer from the host CPU to the DPU.  

When you target this setup with OpenShift Virtualization (KubeVirt), the goal is to pass Single Root I/O Virtualization (SR-IOV) Virtual Functions (VFs) from the DPU straight into your Virtual Machine for near-native, hardware-accelerated networking.

Here is the high-level step-by-step workflow to configure your SNO cluster, join the DPU worker, and launch a KubeVirt VM utilizing it.

Step 1: Install OpenShift Virtualization on your SNO
Before bringing the DPU worker node online, you need to ensure your primary SNO node has virtualization enabled.

Log into your OpenShift Web Console and navigate to Operators -> OperatorHub.

Search for and install the OpenShift Virtualization Operator.

Once the operator is installed, navigate to the Installed Operators page, click OpenShift Virtualization, and create the HyperConverged cluster deployment instance.

Verify that the KubeVirt pods are running seamlessly on your single master node.

Step 2: Prepare the SNO Control Plane for DPF
To orchestrate a BlueField-3 DPU, you will use NVIDIA’s Data Platform Framework (DPF) combined with OpenShift’s networking operators.  

Install the required infrastructure operators from the OperatorHub:

Node Feature Discovery (NFD) Operator (To detect hardware features).  

SR-IOV Network Operator (To manage physical/virtual functions on the DPU card).  

NVIDIA DPF Operator (The main framework coordinating DPU life cycles).  

Configure a specific MachineConfig targeting the future worker node pool to disable local OVS (Open vSwitch) services. Since the BlueField-3 will process OVN-Kubernetes natively on its own Arm cores, the host server's CPU must yield network stack processing to the DPU.  

Step 3: Add the DPU Server as a Worker Node
You need to add your baremetal server containing the DPU to your existing SNO cluster.

Go to the Red Hat Hybrid Cloud Console or use the Assisted Installer via Day-2 operations to add a host.

Configure NMState networking schemas during the host discovery phase. DPF requires a dedicated bridge (commonly named br-dpu) to be built on the host's management interface.

Boot your baremetal worker node using the generated Discovery ISO.

Once the worker node completes installation and joins the SNO control plane, apply the mandatory OVN label to the node:

Bash
oc label node <worker-node-name> k8s.ovn.org/dpu-host=""
Step 4: Provision the BlueField-3 DPU via DPF
With the worker node joined, the DPF operator will detect the card via NFD but will report it as unprovisioned. You must push the BlueField Bootstream (BFB) image to flash the DPU's onboard Arm processor.

Define a DPUFlavor Custom Resource (CR) specifying how the card's resources should be split (e.g., partitioning memory, network speeds).

Define a DPUSet CR that links to your DPUFlavor and includes the URL of the .bfb operating system image file.

The DPF operator will automatically push the image to the card, spin up an isolated Kubernetes Control Plane inside the DPU's Arm cores, and configure OVN-Kubernetes in DPU offload mode.

Step 5: Configure SR-IOV for KubeVirt
To pass high-speed network interfaces to a KubeVirt VM, you must configure the SR-IOV operator to manage the Virtual Functions exposed by the BlueField-3 card.  

Create an SriovNetworkNodePolicy. Set the deviceType to netdevice or vfio-pci.

Note: Set externallyManaged: true within the policy. This crucial flag tells OpenShift's SR-IOV operator to step back and let the NVIDIA DPF Operator manage the low-level virtual interfaces.

Create an SriovNetwork resource targeting the network namespace of your choice. This resource will generate a custom network attachment definition (NAD) that KubeVirt can hook into.

Step 6: Deploy a VM targeting the DPU Worker
Now you can schedule a Virtual Machine Instance (VMI) that relies on the DPU for networking. You must ensure the VM is forced onto the DPU worker node and configured to pick up the SR-IOV network device.

Apply a YAML configuration similar to this:

YAML
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: dpu-accelerated-vm
  namespace: my-vm-workloads
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/domain: dpu-accelerated-vm
    spec:
      # Force the VM to land exclusively on your DPU worker node
      nodeSelector:
        k8s.ovn.org/dpu-host: "" 
      domain:
        cpu:
          cores: 4
        memory:
          guest: 8Gi
        devices:
          interfaces:
          - name: dpu-net
            # Injecting the SR-IOV network directly into the virtual machine
            sriov: {} 
          disks:
          - disk:
              bus: virtio
            name: containerdisk
      networks:
      - name: dpu-net
        multus:
          # References the NAD created via the SR-IOV Network resource in Step 5
          networkName: my-sriov-dpu-network 
      volumes:
      - containerDisk:
          image: quay.io/containerdisks/ubuntu:22.04
        name: containerdisk