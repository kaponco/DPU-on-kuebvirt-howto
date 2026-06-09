# SR-IOV Gap Analysis: What's Missing for VM Hardware Offload

## Current Setup (Installed 2026-05-19)

The DPF operator manages SR-IOV through two components:

**1. DPU Provisioning (DPUFlavor)** — configures VFs at the firmware level:
- `SRIOV_EN=1`, `NUM_OF_VFS=46` in nvconfig
- eSwitch multiport enabled
- OVS with `hw-offload=true` and DOCA mode

**2. SR-IOV Device Plugin (`dpf-sriov-device-plugin`)** — runs on each DPU host worker:
- Image: `nvcr.io/nvidia/mellanox/sriov-network-device-plugin:network-operator-v26.1.0`
- Discovers VFs and registers them as Kubernetes resources via `NodeSRIOVDevicePluginConfig`
- Exposes three VF pools: `bf3-p0-vfs` (9-45), `bf3-p0-vfs-udn` (6-8), `bf3-p0-vfs-mgmt` (1-5)

**Pod-to-pod traffic IS hardware-offloaded** (62 Gbps observed). The OVN-K resource injector
attaches VFs directly to pod network namespaces, and the DPU's eSwitch matches flows on VF
representors for hardware offload.

**Note:** The DPF Installation Manual (Technical Preview) Chapter 4.5 describes a "Custom SR-IOV
Device Plugin Installation" using a standalone DaemonSet with a supervisor script. This was
**not deployed** on this cluster — DPF's own `nodeSRIOVDevicePluginController` handles device
plugin management instead. The manual's custom plugin also does not install the `sriov` CNI binary.

## What's Missing

Three components are needed for KubeVirt `sriov: {}` VM binding. All three are absent:

### 1. `sriov` CNI Binary

**Status: Installed via DaemonSet (2026-06-08)**

The `sriov` CNI binary was not included by DPF. We deployed it via a standalone DaemonSet
(`sriov-cni-installer` in `dpf-operator-system`) that copies the binary from
`ghcr.io/k8snetworkplumbingwg/sriov-cni:v2.8.1` to `/var/lib/cni/bin/` on DPU host workers.

The binary is now present and functional — pods using `"type": "sriov"` NADs can be created.

### 2. `vfio-pci` Driver Binding for VFs

**Status: NOT DONE**

All VFs on srv23 are currently bound to the `mlx5_core` kernel driver. This is correct for
the OVN-K pod acceleration use case (pods use VFs as netdevs in the kernel).

KubeVirt's `sriov: {}` binding passes VFs as PCI devices to the VM via **VFIO** (Virtual
Function I/O). This requires VFs to be bound to the `vfio-pci` userspace driver so the kernel
releases the device and QEMU can claim it.

DPF's VF pool would need to be split:
- VFs for pods: remain on `mlx5_core` (used by OVN-K)
- VFs for VMs: bind to `vfio-pci` (used by KubeVirt VFIO passthrough)

This is normally handled by the SR-IOV Network Operator's config daemon via
`SriovNetworkNodePolicy` with `deviceType: vfio-pci`. Without the operator, manual driver
binding or a custom DaemonSet would be needed.

### 3. `network-resources-injector` Webhook

**Status: NOT INSTALLED**

KubeVirt reads the PCI address of the SR-IOV device from the pod's `k8s.v1.cni.cncf.io/network-status`
annotation. Specifically, it expects a `device-info` field:

```json
{
  "name": "sriov-test/dpu-sriov-passthrough",
  "interface": "net1",
  "device-info": {
    "type": "pci",
    "version": "1.0.0",
    "pci": {
      "pci-address": "0000:b5:01.1"
    }
  }
}
```

The `network-resources-injector` is a mutating admission webhook that:
- Reads the `resourceName` annotation from the NetworkAttachmentDefinition
- Injects the corresponding resource requests/limits into the pod spec
- Populates the `device-info` with the PCI address of the allocated VF

Without it, the `sriov` CNI creates the network attachment but KubeVirt's virt-handler fails
with: `"failed to create PCI address pool with network status from file: context deadline
exceeded: file is not populated with network-info"`

The `network-resources-injector` is a component of the SR-IOV Network Operator. It can also
be deployed standalone from https://github.com/k8snetworkplumbingwg/network-resources-injector.

## Why DPF Doesn't Include These

- DPF's `cniInstaller` component only installs the **RDMA CNI** binary, not sriov
- DPF's `dpf-ovs-cni` (which handles SR-IOV for pods) is deployed on the **DPU ARM cluster**, not host nodes
- The DPF architecture uses `mlx5_core` for all VFs — pods access them as kernel netdevs, not VFIO devices
- DPF has its own resource injection mechanism (OVN-K MutatingAdmissionPolicy) for pods, not the `network-resources-injector` webhook
- There is no upstream support for KubeVirt VM SR-IOV passthrough in DPF

## Test Results (2026-06-08)

| Test | Result |
|------|--------|
| sriov CNI binary installed via DaemonSet | Binary deployed, pod sandbox created successfully |
| VM with `sriov: {}` binding | **Failed** — virt-handler error: `file is not populated with network-info` |
| Root cause #1 | Missing `device-info` in network-status (no `network-resources-injector`) |
| Root cause #2 | VFs bound to `mlx5_core`, not `vfio-pci` |
| vfio-pci binding attempt | **Failed** — DPU VFs have no IOMMU groups on the host |
| VFs restored to mlx5_core | Yes — VFs 6-8 back on mlx5_core after failed binding |

### IOMMU Blocker (Fundamental)

DPU VFs behind the BlueField-3 PCIe switch **do not have IOMMU groups** exposed to the host.
VFIO requires IOMMU group isolation per device. This is by design — the DPU's eSwitch
manages device isolation, not the host's IOMMU.

```
# Verified on srv23:
$ ls /sys/bus/pci/devices/0000:b5:01.1/iommu_group
ls: cannot access: No such file or directory
```

**This means KubeVirt `sriov: {}` VFIO passthrough is not possible for DPU VFs.**

## The vDPA Path Forward (Recommended)

### What is vDPA?

**vDPA (virtio DataPath Acceleration)** is a technology that exposes SR-IOV VFs as virtio-net
devices. The VM connects via standard virtio (no VFIO/IOMMU needed), while the data plane
goes directly through the DPU's eSwitch for hardware offload.

```
VM virtio-net → vhost-vdpa → SR-IOV VF → DPU eSwitch (hardware offload)
                                        → VF representor on OVS br-int (flow matching)
```

### Why vDPA Solves the Problem

| Issue | VFIO Approach | vDPA Approach |
|-------|---------------|---------------|
| IOMMU groups | Required (missing on DPU VFs) | Not required |
| VM interface | PCI passthrough (Mellanox NIC) | virtio-net (standard) |
| Driver in VM | mlx5_core (vendor-specific) | virtio-net (generic) |
| OVS flow matching | Via VF representor | Via VF representor (same) |
| Hardware offload | Yes (if VFIO works) | Yes |
| Live migration | Complex (PCI device state) | Easier (virtio standard) |

### vDPA Implementation Steps and Verification (2026-06-08)

The four steps to implement vDPA and their status on this cluster:

#### Step 1: Enable vDPA on the Host — DONE

All kernel modules are available and loaded on srv23:

```
$ lsmod | grep vdpa
vhost_vdpa             32768  0
virtio_vdpa            16384  0
mlx5_vdpa              86016  0
vdpa                   40960  3 virtio_vdpa,vhost_vdpa,mlx5_vdpa
```

Modules are not persistent across reboots — would need a MachineConfig or DaemonSet to
load them at boot.

#### Step 2: Configure DPU VFs for vDPA — DONE (Hardware Ready)

BF3 VFs are vDPA-capable. Verified by creating a vDPA device and binding to `vhost-vdpa`:

```bash
# VF6 is a vDPA management device
$ vdpa mgmtdev show pci/0000:b5:01.1
pci/0000:b5:01.1:
  supported_classes net
  max_supported_vqs 65
  dev_features CSUM GUEST_CSUM MTU MAC HOST_TSO4 HOST_TSO6 ...

# Create vDPA device (VF stays on mlx5_core — no unbinding needed)
$ vdpa dev add name vdpa6 mgmtdev pci/0000:b5:01.1
# SUCCESS — vdpa6: type network mgmtdev pci/0000:b5:01.1 vendor_id 5555

# Rebind from virtio-vdpa (containers) to vhost-vdpa (VMs)
$ echo vhost_vdpa > /sys/bus/vdpa/devices/vdpa6/driver_override
$ echo vdpa6 > /sys/bus/vdpa/drivers/virtio_vdpa/unbind
$ echo vdpa6 > /sys/bus/vdpa/drivers/vhost_vdpa/bind

# Result: /dev/vhost-vdpa-0 created — this is what QEMU uses
$ ls -la /dev/vhost-vdpa-0
crw-------. 1 root root 498, 0 Jun  8 10:03 /dev/vhost-vdpa-0
```

**Key finding:** The VF stays bound to `mlx5_core` while the vDPA device is created on top
of it. No IOMMU groups needed. The `mlx5_vdpa` kernel module provides the vDPA management
layer through `mlx5_core`.

#### Step 3: SR-IOV Device Plugin with vdpaType — BLOCKED

The upstream `sriov-network-device-plugin` supports `"vdpaType": "vhost"` in its raw JSON
config:

```json
{
  "resourceName": "bf3_vdpa_vhost",
  "selectors": [{
    "vendors": ["15b3"],
    "devices": ["<VF_DEVICE_ID>"],
    "drivers": ["mlx5_core"],
    "vdpaType": "vhost"
  }]
}
```

**However**, DPF's `NodeSRIOVDevicePluginConfig` CRD does NOT support `vdpaType`:
- The CRD schema only has `isRdma` in the `options` field
- The DPF controller generates the device plugin config from this CRD
- No way to pass `vdpaType` through DPF's abstraction layer

**Workaround deployed:** A standalone `sriov-network-device-plugin` with `vdpaType: "vhost"`
and `pfNames: ["ens7f0np0#6-8"]` runs alongside DPF's device plugin. VFs 6-8 were removed
from DPF's `NodeSRIOVDevicePluginConfig` to avoid conflicts.

#### Step 4: KubeVirt VM vDPA Binding — DEPLOYED

The official KubeVirt vDPA Network Binding Plugin:
**https://github.com/kubevirt/vdpa-network-binding-plugin**

**Deployment issues encountered:**
- Official images at `quay.io/kubevirt/` are **not published** (unauthorized)
- Community images at `ghcr.io/k8snetworkplumbingwg/` also **not published** (403)
- **Solution:** Built from source and pushed to `quay.io/rh-ee-skapon/vdpa-network-binding-sidecar:v1-amd64`
- Must build with `--platform linux/amd64` on ARM Macs to avoid `Exec format error`

**On OpenShift with HCO:** Patch the HyperConverged CR at `spec.networkBinding` — NOT the
KubeVirt CR directly (HCO reverts direct KubeVirt patches).

```bash
oc patch hyperconverged kubevirt-hyperconverged -n openshift-cnv --type merge --patch '
spec:
  networkBinding:
    vdpa:
      sidecarImage: quay.io/rh-ee-skapon/vdpa-network-binding-sidecar:v1-amd64
      downwardAPI: device-info
      migration:
        method: ""
'
```

**Result:** Sidecar runs (4/4 containers in virt-launcher pod), but VM fails at
`failed to read network-info: context deadline exceeded` — see Step 5 below.

#### Step 5: CNI Plugin Selection — RESOLVED (Use `tap` CNI)

The `sriov` and `host-device` CNI plugins both fail with `"device or resource busy"` because
they try to move the VF kernel netdev into the pod namespace. With vDPA, the VF is occupied
by the vDPA subsystem and can't be moved.

**Solution:** Use `"type": "tap"` in the NAD as a no-op CNI. The device plugin handles
`/dev/vhost-vdpa-X` passthrough via the Kubernetes device plugin API (not CNI).

#### Step 6: device-info Plumbing — BLOCKED (Final Blocker)

The vDPA sidecar needs device-info from the downward API (`downwardAPI: "device-info"` in
the binding config). virt-handler reads this to know which `/dev/vhost-vdpa-X` to pass to QEMU.

The `network-resources-injector` webhook is needed to populate device-info in the pod's
`k8s.v1.cni.cncf.io/network-status` annotation. Without it, virt-handler times out.

**Next action:** Deploy `network-resources-injector` v1.8.0 from:
https://github.com/k8snetworkplumbingwg/network-resources-injector

### Complete Status Summary

| Component | Status | Blocker? |
|-----------|--------|----------|
| Kernel modules (vdpa, mlx5-vdpa, vhost-vdpa) | Loaded and working | No |
| BF3 VF vDPA hardware support | Verified — `/dev/vhost-vdpa-{0,1,2}` created | No |
| vDPA device creation (`vdpa dev add`) | Works via lifecycle DaemonSet | No |
| vhost-vdpa driver binding | Works | No |
| Standalone vDPA device plugin | **Deployed** — `bf3-vdpa-vhost: 3` on srv23 | No |
| KubeVirt vDPA sidecar | **Deployed** — built from source, runs in virt-launcher (4/4) | No |
| HCO vDPA binding registration | **Done** — `spec.networkBinding.vdpa` | No |
| CNI plugin (`tap` as no-op) | Works — pod sandbox created | No |
| sriov CNI binary | Installed via DaemonSet | No |
| **network-resources-injector** | **Not deployed** | **Yes** — device-info not injected, virt-handler times out |

### What's Required to Close the Gap (Updated 2026-06-08)

**All components have been deployed and verified. The final blocker is in the device plugin code.**

The `sriov-network-device-plugin v3.9.0` discovers vDPA devices and passes `/dev/vhost-vdpa-X`
to pods, but leaves the `vdpa` struct empty in its `PCIDEVICE_*_INFO` env var. This breaks the
device-info chain: device plugin → NRI → annotations → downward API → virt-launcher → sidecar.

**What would fix it:**
- Upstream patch to `sriov-network-device-plugin` to populate vDPA device-info
- Custom admission webhook to inject device-info annotations from env vars
- KubeVirt sidecar enhancement to read PCIDEVICE env vars directly

**Deployed infrastructure (all working individually):**
- vDPA lifecycle DaemonSet — creates devices, binds `vhost-vdpa`, writes devinfo JSON
- Standalone vDPA device plugin — `vdpaType: "vhost"`, `pfNames: ["ens7f0np0#6-8"]`
- KubeVirt vDPA sidecar — built from source, registered via HCO `spec.networkBinding`
- network-resources-injector — v1.8.0, `-insecure` flag for OpenShift
- noop-cni — shell script CNI that returns success without moving devices
- devinfo files — written to `/var/run/k8s.cni.cncf.io/devinfo/dp/` on host

## Options Summary (Final)

### Option 1: Fix Device Plugin vDPA Info (Development Required)

Patch `sriov-network-device-plugin` to populate the `vdpa` struct in `PCIDEVICE_*_INFO`,
or write a custom webhook to bridge the gap. All other components are deployed and working.

- Effort: Moderate — device plugin code change or custom webhook
- Risk: Low — targeted change, rest of stack verified
- Reward: Complete vDPA hardware offload

### Option 2: Accept Software-Path Performance (Current Fallback)

Keep `bridge: {}` binding with `ovn-k8s-cni-overlay`. Traffic goes through DPDK software path
on DPU ARM cores at 43.5 Gbps. No hardware offload. Working in `iperf3-test` namespace.

### Option 3: Wait for Upstream Integration

NVIDIA and Red Hat will eventually provide integrated vDPA support with published images
and proper device-info plumbing.

## References

- [NVIDIA - Virtio Acceleration through Hardware vDPA](https://docs.nvidia.com/doca/sdk/virtio-acceleration-through-hardware-vdpa/index.html)
- [NVIDIA - OVS-DPDK Hardware Acceleration](https://docs.nvidia.com/doca/sdk/ovs-dpdk-hardware-acceleration/index.html)
- [Red Hat - Running containerized workloads with virtio/vDPA](https://www.redhat.com/en/blog/running-containerized-workloads-virtiovdpa)
- [OVN-Kubernetes - OVS Acceleration with kernel datapath](https://ovn-kubernetes.io/features/hardware-offload/ovs-kernel/)
- [OVN-Kubernetes - DPU Support](https://ovn-kubernetes.io/features/hardware-offload/dpu-support/)
- [Red Hat - DPU-enabled networking with OpenShift and NVIDIA DPF](https://developers.redhat.com/articles/2025/03/20/dpu-enabled-networking-openshift-and-nvidia-dpf)
- [KubeVirt vDPA Network Binding Plugin](https://github.com/kubevirt/vdpa-network-binding-plugin) — Official plugin (sidecar + webhook)
- [KubeVirt Summit 2024 - vDPA Workflow](https://www.youtube.com/watch?v=ZFEq6F5p-fU) — SK Telecom demo of KubeVirt + vDPA
- [KubeVirt Network Binding Plugins](https://kubevirt.io/user-guide/network/network_binding_plugins/) — Plugin framework docs
- [SR-IOV Network Device Plugin](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin) — vdpaType selector support
- [KubeVirt Issue #13432](https://github.com/kubevirt/kubevirt/issues/13432) — virt-launcher `network-info` timeout error
- [network-resources-injector](https://github.com/k8snetworkplumbingwg/network-resources-injector) — Standalone webhook for PCI device-info injection
- [sriov-cni](https://github.com/k8snetworkplumbingwg/sriov-cni) — SR-IOV CNI plugin binary
