# VM Networking with DPF, UDN, and SR-IOV

This document describes three configuration options for running KubeVirt VMs on DPU-accelerated worker nodes with different combinations of DPF (DataPath Framework), UDN (User Defined Networks), and SR-IOV passthrough.

## Background: The Integration Challenge

DPF, UDN, and KubeVirt SR-IOV passthrough each work independently, but combining all three is not yet a supported upstream pattern. The NVIDIA DOCA Platform documentation covers:

- **DPF + OVN-K acceleration for pods** (primary network via `dpf-ovn-kubernetes` NAD)
- **DPF + OVN-K secondary networks for pods** (via `enableMultiNetwork: "true"` and secondary NADs)

However, **none of the upstream docs** (doca-platform, network-operator, nic-configuration-operator) address KubeVirt VM integration with DPF-accelerated networks. The three options below represent practical approaches given the current state of the tooling.

## Cluster Prerequisites

All options assume:

- DPU worker nodes with BlueField-3 cards provisioned via DPF
- OVN-Kubernetes as the primary CNI with DPU offload enabled
- The `dpf-ovn-kubernetes` NetworkAttachmentDefinition in `openshift-ovn-kubernetes`
- DPF's resource injection webhook automatically attaching VFs to pods on DPU workers

### NodeSRIOVDevicePluginConfig

The `NodeSRIOVDevicePluginConfig` defines three VF pools from PF0 on the DPU:

```yaml
apiVersion: noderesources.dpu.nvidia.com/v1alpha1
kind: NodeSRIOVDevicePluginConfig
metadata:
  name: bf3-p0-vfs
  namespace: dpf-operator-system
spec:
  devicePluginResources:
    - name: bf3-p0-vfs-mgmt         # VFs 1-5: management (OVN-K mgmt port, provisioning)
      ranges:
        - pfIndex: 0
          start: 1
          end: 5
      type: vf
    - name: bf3-p0-vfs-udn          # VFs 6-8: dedicated for UDN/secondary networks
      options:
        isRdma: true
      ranges:
        - pfIndex: 0
          start: 6
          end: 8
      type: vf
    - name: bf3-p0-vfs               # VFs 9-45: general workload (primary network injection)
      options:
        isRdma: true
      ranges:
        - pfIndex: 0
          start: 9
          end: 45
      type: vf
```

These appear on host nodes as allocatable resources:

| Resource | VF Range | Purpose |
|----------|----------|---------|
| `openshift.io/bf3-p0-vfs-mgmt` | 1-5 | Management and OVN-K mgmt port |
| `openshift.io/bf3-p0-vfs-udn` | 6-8 | UDN / secondary network VFs |
| `openshift.io/bf3-p0-vfs` | 9-45 | Primary network (auto-injected by DPF webhook) |

---

## Option A: UDN as Secondary Network on a VM (OVN-K Accelerated via DPF)

### Concept

The VM gets its primary network through the default DPF-accelerated pod network, and a **secondary** network through an OpenShift UserDefinedNetwork that requests a VF from the DPU. Both networks are managed by OVN-Kubernetes and benefit from DPU hardware offload.

### Networks

| Network | Type | Interface in VM | IP Range | VF Pool |
|---------|------|-----------------|----------|---------|
| Primary (default) | Pod network via `dpf-ovn-kubernetes` | `eth0` | Cluster pod CIDR | `openshift.io/bf3-p0-vfs` (auto-injected) |
| Secondary (UDN) | UserDefinedNetwork via OVN-K overlay | `net1` | 192.168.64.0/20 | `openshift.io/bf3-p0-vfs-udn` |

### How It Works

1. The DPF resource injection webhook automatically adds a VF from `bf3-p0-vfs` for the primary network
2. The UDN controller creates a NetworkAttachmentDefinition with `ovn-k8s-cni-overlay` CNI plugin
3. The VM requests a second VF from `bf3-p0-vfs-udn` for the secondary interface
4. OVN-Kubernetes manages both interfaces, providing overlay networking, IP allocation, and network policy

### SR-IOV Configuration

**UserDefinedNetwork** (creates the NAD automatically):

```yaml
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: accelerated-network
  namespace: <vm-namespace>
  annotations:
    k8s.v1.cni.cncf.io/resourceName: openshift.io/bf3-p0-vfs-udn
spec:
  topology: Layer3
  layer3:
    role: Secondary
    subnets:
      - cidr: 192.168.64.0/20
        hostSubnet: 24
```

The annotation `k8s.v1.cni.cncf.io/resourceName: openshift.io/bf3-p0-vfs-udn` tells the system to allocate VFs from the dedicated UDN pool rather than the general pool.

### VM Manifest

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-udn-secondary
  namespace: <vm-namespace>
  labels:
    app: test-vm
  annotations:
    kubevirt.io/allow-pod-bridge-network-live-migration: "true"
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-udn-secondary
    spec:
      nodeSelector:
        kubernetes.io/hostname: nvd-srv-23.nvidia.eng.rdu2.dc.redhat.com
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Exists"
          effect: "NoSchedule"
      domain:
        devices:
          autoattachSerialConsole: true
          interfaces:
            - name: default
              bridge: {}
            - name: udn-net
              bridge: {}
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
        resources:
          requests:
            memory: 1Gi
            cpu: "1"
      networks:
        - name: default
          pod: {}
        - name: udn-net
          multus:
            networkName: accelerated-network
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/containerdisks/fedora:latest
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |
              #cloud-config
              user: fedora
              password: fedora
              chpasswd: { expire: False }
              ssh_pwauth: True
```

### Pros and Cons

**Pros:**
- Full OVN-K overlay features on both networks (network policy, DNS, east-west routing)
- DPU hardware acceleration on both interfaces
- IP management handled by OVN-K

**Cons:**
- UDN + DPF integration is not documented upstream — the UDN controller may not be fully DPF-aware
- Overlay overhead on the secondary network (encapsulation)
- Limited to 3 VFs in the `bf3-p0-vfs-udn` pool (current config)

### Current Status

**Not fully validated.** The UDN NAD uses `ovn-k8s-cni-overlay` with `cniVersion: 1.1.0` which follows a different codepath than the upstream DOCA secondary network NAD (`cniVersion: 0.4.0`, `topology: layer2`). Requires further testing to confirm DPU offload works correctly for UDN-created secondary interfaces on VMs.

---

## Option B: Direct SR-IOV VF Passthrough (Bypasses OVN-K)

### Concept

The VM gets **only** a direct SR-IOV VF passthrough interface — no pod network, no OVN-K overlay. The VF is passed directly to the VM's guest OS as a PCI device, giving near-native network performance. The VM communicates on the raw L2 network attached to the DPU's physical port.

### Networks

| Network | Type | Interface in VM | IP Range | VF Pool |
|---------|------|-----------------|----------|---------|
| SR-IOV passthrough | Direct VF (no overlay) | `eth0` (PCI device) | Manually configured or external DHCP | `openshift.io/bf3-p0-vfs-udn` |

### How It Works

1. A NetworkAttachmentDefinition is created with the `sriov` CNI plugin (not `ovn-k8s-cni-overlay`)
2. The VM requests a VF from the `bf3-p0-vfs-udn` pool
3. The VF is passed through to the VM as a PCI device via VFIO
4. The VM sees a Mellanox NIC directly — no virtio, no overlay, no OVN-K involvement
5. IP configuration is manual (static) or via external DHCP on the physical network

### SR-IOV Configuration

**NetworkAttachmentDefinition** (manual, not UDN-managed):

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: dpu-sriov-net
  namespace: <vm-namespace>
  annotations:
    k8s.v1.cni.cncf.io/resourceName: openshift.io/bf3-p0-vfs-udn
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "sriov",
      "vlan": 0,
      "ipam": {}
    }
```

Setting `"ipam": {}` means no automatic IP assignment — the VM must configure its own IP (static or DHCP from the physical network).

### VM Manifest

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-sriov-passthrough
  namespace: <vm-namespace>
  labels:
    app: test-vm
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-sriov-passthrough
    spec:
      nodeSelector:
        kubernetes.io/hostname: nvd-srv-23.nvidia.eng.rdu2.dc.redhat.com
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Exists"
          effect: "NoSchedule"
      domain:
        devices:
          autoattachSerialConsole: true
          interfaces:
            - name: dpu-net
              sriov: {}
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
        resources:
          requests:
            memory: 1Gi
            cpu: "1"
      networks:
        - name: dpu-net
          multus:
            networkName: dpu-sriov-net
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/containerdisks/fedora:latest
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |
              #cloud-config
              user: fedora
              password: fedora
              chpasswd: { expire: False }
              ssh_pwauth: True
```

Key differences from other options:
- `interfaces` uses `sriov: {}` binding — the VF is passed as a PCI device, not a virtio NIC
- No `pod: {}` default network — the VM has no connection to the cluster pod network
- The VF resource (`openshift.io/bf3-p0-vfs-udn`) is **not** requested in `resources.requests` — KubeVirt handles this automatically when `sriov: {}` is used with a Multus NAD that has a `resourceName` annotation

### Pros and Cons

**Pros:**
- Maximum performance — near-native line rate, lowest latency
- The VM sees a real Mellanox NIC (RDMA capable if `isRdma: true` in the VF pool)
- No overlay overhead
- Simplest DPF integration — no dependency on OVN-K for this interface

**Cons:**
- No cluster networking features (no network policy, no DNS, no service discovery)
- No pod network means no `oc exec`, no `virtctl console` (unless you add a second interface)
- Manual IP management required
- The VM is isolated from the cluster network — only reachable from the physical L2 network

### When to Use

Best for dedicated high-performance workloads where the VM needs raw network access (RDMA, storage traffic, benchmarking) and doesn't need cluster integration. **Not recommended as the only interface** — combine with Option C instead.

---

## Option C: Combined — Primary DPF-Accelerated + Secondary SR-IOV Passthrough (Recommended)

### Concept

The VM gets **two interfaces**: a primary DPF-accelerated pod network (for cluster connectivity, management, SSH) and a secondary SR-IOV VF passthrough (for high-performance data traffic). This gives the best of both worlds.

### Networks

| Network | Type | Interface in VM | IP Range | VF Pool |
|---------|------|-----------------|----------|---------|
| Primary (default) | Pod network via `dpf-ovn-kubernetes` | `eth0` (virtio) | Cluster pod CIDR (auto) | `openshift.io/bf3-p0-vfs` (auto-injected) |
| Secondary (SR-IOV) | Direct VF passthrough | `enp1s0` or similar (PCI device) | Manual / external DHCP | `openshift.io/bf3-p0-vfs-udn` |

### How It Works

1. The DPF resource injection webhook automatically adds a VF from `bf3-p0-vfs` for the primary pod network — this gives the VM full cluster connectivity via OVN-K
2. A second VF from `bf3-p0-vfs-udn` is passed through directly to the VM as a PCI device via the `sriov` CNI plugin
3. Inside the VM, `eth0` provides cluster access (SSH, DNS, services) and the SR-IOV NIC provides a high-performance data path

### SR-IOV Configuration

**NetworkAttachmentDefinition** for the secondary SR-IOV interface:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: dpu-sriov-net
  namespace: <vm-namespace>
  annotations:
    k8s.v1.cni.cncf.io/resourceName: openshift.io/bf3-p0-vfs-udn
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "sriov",
      "vlan": 0,
      "ipam": {}
    }
```

### VM Manifest

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-dpf-sriov
  namespace: <vm-namespace>
  labels:
    app: test-vm
  annotations:
    kubevirt.io/allow-pod-bridge-network-live-migration: "true"
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-dpf-sriov
    spec:
      nodeSelector:
        kubernetes.io/hostname: nvd-srv-23.nvidia.eng.rdu2.dc.redhat.com
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Exists"
          effect: "NoSchedule"
      domain:
        devices:
          autoattachSerialConsole: true
          interfaces:
            - name: default
              bridge: {}
            - name: dpu-net
              sriov: {}
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
        resources:
          requests:
            memory: 1Gi
            cpu: "1"
      networks:
        - name: default
          pod: {}
        - name: dpu-net
          multus:
            networkName: dpu-sriov-net
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/containerdisks/fedora:latest
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |
              #cloud-config
              user: fedora
              password: fedora
              chpasswd: { expire: False }
              ssh_pwauth: True
              bootcmd:
                - [ sh, -c, 'grubby --update-kernel=ALL --args="console=tty0 console=ttyS0,115200n8"' ]
              runcmd:
                - systemctl enable serial-getty@ttyS0.service
                - systemctl start serial-getty@ttyS0.service
```

Key points:
- `default` network with `bridge: {}` — primary pod network, auto-injected with a DPF-accelerated VF
- `dpu-net` with `sriov: {}` — secondary SR-IOV passthrough, direct VF access
- Both `pod: {}` and `multus` networks are defined
- Serial console is enabled for `virtctl console` access

### What the VM Sees

```
eth0        - virtio NIC (primary, DPF-accelerated OVN-K pod network)
              IP: assigned by OVN-K (e.g., 10.130.2.x/23)
              Use: SSH, cluster services, DNS, management

enp1s0      - Mellanox ConnectX VF (SR-IOV passthrough)
              IP: manually configured or external DHCP
              Use: high-performance data traffic, iperf3, RDMA
```

### Pros and Cons

**Pros:**
- Full cluster connectivity via the primary network (SSH, DNS, services, `virtctl console`)
- Near-native performance on the secondary SR-IOV interface
- Clean separation of management and data traffic
- Works with current DPF tooling — no dependency on unvalidated UDN+DPF integration
- RDMA capable on the SR-IOV interface (if `isRdma: true` in the VF pool)

**Cons:**
- No OVN-K features on the SR-IOV interface (no network policy, no overlay routing)
- Consumes 2 VFs per VM (one auto-injected for primary, one for SR-IOV)
- IP management on the SR-IOV interface is manual
- The SR-IOV interface is raw L2 — VMs on different nodes need physical L2 connectivity between DPU ports

### Why This Is the Recommended Option

This is the most practical approach because:

1. The primary network is already working and proven in the current cluster
2. SR-IOV passthrough to VMs is a well-established KubeVirt feature
3. It avoids the unvalidated UDN + DPF codepath (Option A)
4. It provides both management access and high-performance data path (unlike Option B)
5. It matches the pattern from the DOCA Platform's secondary network docs, adapted for KubeVirt

---

## Comparison Summary

| Feature | Option A (UDN Secondary) | Option B (SR-IOV Only) | Option C (Combined) |
|---------|--------------------------|------------------------|---------------------|
| Primary network | Pod (DPF-accelerated) | None | Pod (DPF-accelerated) |
| Secondary network | UDN (OVN-K overlay) | SR-IOV passthrough | SR-IOV passthrough |
| Network policy | Both interfaces | Neither | Primary only |
| DNS / services | Yes | No | Yes (primary) |
| Data-plane performance | Good (overlay) | Best (native) | Best (native) on secondary |
| `virtctl console` / SSH | Yes | No | Yes |
| RDMA support | No (overlay) | Yes | Yes (secondary) |
| IP management | Automatic (OVN-K) | Manual | Primary auto, secondary manual |
| VFs consumed per VM | 2 | 1 | 2 |
| Upstream validation | Not validated | Partial | Most practical |
| **Recommendation** | Wait for upstream | Dedicated workloads only | **Use this** |

## Test Results and Hardware Offload Findings (2026-06-07)

### What Was Tested

Option C was deployed on srv23 (VM) → srv27 (test pod) using the secondary OVN-K layer2
network (192.168.200.0/24) with VFs from the `bf3-p0-vfs-udn` pool.

**iperf3 result: 43.5 Gbits/sec** (50.6 GBytes in 10 seconds, 44 retransmits).

### Hardware Offload Status

**No hardware offload was achieved for VM traffic.** The DPU's OVS showed zero flows
(offloaded or software) on the VM's VF representor ports. Root cause:

KubeVirt's `bridge: {}` interface binding creates a Linux bridge inside the virt-launcher pod
that connects the VM's tap device to the VF. Traffic arrives on a different OVS ingress port
than the VF representor, preventing the DPU's eSwitch from matching and offloading flows.

The same issue affects `masquerade: {}` binding (which also broke outbound connectivity entirely).

The 43.5 Gbps was achieved via OVS **DPDK software datapath** on the DPU's ARM cores — fast,
but not true eSwitch hardware offload.

### Blocking Issues for True Hardware Offload

1. **`sriov` CNI plugin not installed** — DPF clusters use their own SR-IOV device plugin
   (via `NodeSRIOVDevicePluginConfig`), not the OpenShift SR-IOV operator. The `sriov` CNI
   binary is absent from `/var/lib/cni/bin`, preventing `sriov: {}` VM binding.

2. **`bridge: {}` binding incompatible with eSwitch offload** — The Linux bridge inside
   virt-launcher changes the traffic's ingress port as seen by OVS on the DPU.

3. **No upstream KubeVirt + DPF integration** — None of the DOCA Platform, Network Operator,
   or NIC Configuration Operator docs address KubeVirt VM networking.

### What Would Be Needed

To achieve true hardware-offloaded VM networking with DPF:
- Install the `sriov` CNI binary on DPU worker nodes, OR
- Use `vhostuser: {}` binding with DPDK passthrough, OR
- Wait for upstream NVIDIA/Red Hat integration of DPF + KubeVirt

## References

- [DOCA Platform - OVN-K Use Case](https://github.com/NVIDIA/doca-platform/tree/public-main/docs/public/user-guides/host-trusted/use-cases/ovnk) — DPF + OVN-K deployment guide
- [DOCA Platform - Secondary Networks](https://github.com/NVIDIA/doca-platform/tree/public-main/docs/public/advanced-configuration/secondary-networks) — Secondary network support for HBN-OVNK (pods only)
- [DOCA Platform - Multi-DPU OVN-K](https://github.com/NVIDIA/doca-platform/blob/public-main/docs/public/advanced-configuration/multi-dpu-ovnk-hbn.md) — SR-IOV device plugin and multi-DPU architecture
- [NVIDIA Network Operator](https://github.com/Mellanox/network-operator) — Manages Multus, SR-IOV device plugin, NIC configuration
- [NIC Configuration Operator](https://github.com/Mellanox/nic-configuration-operator) — Firmware-level NIC configuration (VF count, link type, performance tuning)
