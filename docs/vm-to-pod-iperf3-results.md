# VM-to-Pod iperf3 Test Results — Secondary OVN-K Network

## Test Date

2026-06-07

## Objective

Measure network throughput from a KubeVirt VM to a pod across two DPU host worker nodes over a secondary OVN-Kubernetes Layer 2 network with hardware-accelerated VFs.

## Test Setup

### Cluster

- **Platform**: OpenShift 4.20 with DPF v26.4.0
- **DPU Hardware**: NVIDIA BlueField-3

### Nodes

| Role | Node | IP |
|------|------|----|
| VM (iperf3 client) | nvd-srv-23.nvidia.eng.rdu2.dc.redhat.com | 10.6.135.2 |
| Pod (iperf3 server) | nvd-srv-27.nvidia.eng.rdu2.dc.redhat.com | 10.6.135.6 |

### Network

- **Type**: Secondary OVN-Kubernetes Layer 2 UserDefinedNetwork
- **Name**: `accelerated-network`
- **Subnet**: 192.168.200.0/24
- **SR-IOV VF Pool**: `openshift.io/bf3-p0-vfs-udn` (BlueField-3 Virtual Functions)

### VM Configuration

- **Name**: `vm-srv23-sriov` (namespace: `sriov-test`)
- **Image**: Fedora (quay.io/containerdisks/fedora:latest)
- **Resources**: 1 vCPU, 1 GiB memory
- **Network Binding**: `bridge: {}` (virtio-net, software datapath on DPU ARM cores)
- **VM IP**: 192.168.200.2 (interface `enp2s0`)
- **Node Pinning**: `kubernetes.io/hostname: nvd-srv-23`

### Pod Configuration

- **Name**: `test-pod-srv27` (namespace: `sriov-test`)
- **Image**: `networkstatic/iperf3`
- **Pod IP**: 192.168.200.4 (interface `net1`)
- **Mode**: iperf3 server (`iperf3 -s`)

## What We Ran

1. Deployed a secondary OVN-K Layer 2 UDN (`accelerated-network`) in the `sriov-test` namespace
2. Deployed a Fedora VM on srv23 with `bridge: {}` binding and UDN connectivity
3. Deployed an iperf3 server pod on srv27 attached to the same UDN
4. Installed iperf3 inside the VM via `dnf install`
5. Ran a 10-second single-stream TCP test from the VM to the pod:

```bash
# Inside the VM (virtctl console)
iperf3 -c 192.168.200.4
```

## Output

```
Connecting to host 192.168.200.4, port 5201
[  5] local 192.168.200.2 port 40632 connected to 192.168.200.4 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  5.37 GBytes  46.0 Gbits/sec   44   4.16 MBytes
[  5]   1.00-2.00   sec  5.51 GBytes  47.3 Gbits/sec    0   4.16 MBytes
[  5]   2.00-3.00   sec  5.42 GBytes  46.5 Gbits/sec    0   4.16 MBytes
[  5]   3.00-4.00   sec  4.59 GBytes  39.4 Gbits/sec    0   4.16 MBytes
[  5]   4.00-5.00   sec  3.40 GBytes  29.3 Gbits/sec    0   4.16 MBytes
[  5]   5.00-6.00   sec  5.45 GBytes  46.8 Gbits/sec    0   4.16 MBytes
[  5]   6.00-7.00   sec  5.49 GBytes  47.1 Gbits/sec    0   4.16 MBytes
[  5]   7.00-8.00   sec  5.39 GBytes  46.3 Gbits/sec    0   4.16 MBytes
[  5]   8.00-9.00   sec  4.93 GBytes  42.3 Gbits/sec    0   4.16 MBytes
[  5]   9.00-10.00  sec  5.07 GBytes  43.6 Gbits/sec    0   4.16 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  50.6 GBytes  43.5 Gbits/sec   44            sender
[  5]   0.00-10.00  sec  50.6 GBytes  43.5 Gbits/sec                  receiver

iperf Done.
```

## Results Summary

| Metric | Value |
|--------|-------|
| **Average Throughput** | 43.5 Gbits/sec |
| **Peak Throughput** | 47.3 Gbits/sec |
| **Total Data Transferred** | 50.6 GBytes |
| **Test Duration** | 10 seconds |
| **Retransmissions** | 44 (first second only) |
| **Streams** | 1 (single stream) |

## Comparison with Pod-to-Pod Baseline

| Test | Throughput | Streams | Binding |
|------|-----------|---------|---------|
| Pod-to-Pod (default network) | 62.0 Gbps | 1 | N/A (direct OVN-K) |
| Pod-to-Pod (default network) | 48.4 Gbps | 4 | N/A (direct OVN-K) |
| **VM-to-Pod (secondary UDN)** | **43.5 Gbps** | **1** | **bridge: {}** |

## Traffic Path

```
VM (192.168.200.2)
  │ virtio-net
  ▼
virt-launcher pod (bridge binding)
  │ tap device → Linux bridge → OVS
  ▼
BlueField-3 DPU (srv23)
  │ DPDK software path on DPU ARM cores
  │ Geneve encapsulation
  ▼
Physical network (100 GbE)
  │
  ▼
BlueField-3 DPU (srv27)
  │ Geneve decap → OVS → pod netns
  ▼
iperf3 server pod (192.168.200.4)
```

## Conclusions

1. **43.5 Gbps is a strong result for software-path VM networking.** The VM achieves ~70% of the pod-to-pod single-stream baseline (62 Gbps), with the gap attributable to the virtio + Linux bridge overhead in the `bridge: {}` binding.

2. **No hardware offload for VM traffic in this configuration.** The `bridge: {}` binding routes VM traffic through a Linux bridge inside the virt-launcher pod, which changes the OVS ingress port and prevents the DPU from matching it to hardware-offloaded flows. The DPU ARM cores handle packet processing in software (DPDK), not in the hardware pipeline.

3. **Retransmissions are minimal.** Only 44 retransmissions occurred in the first second (connection ramp-up), with zero retransmissions for the remaining 9 seconds — indicating a stable, well-behaved path.

4. **Hardware offload requires vDPA binding.** To achieve true hardware-offloaded VM networking (where the DPU ASIC processes packets at line rate), the VM needs a `vdpa` network binding instead of `bridge`. This was attempted in the [vDPA investigation](../custom-resources/secondary_sriov/README.md) but is currently blocked by the SR-IOV device plugin not populating the `vdpa` struct in device-info.

## Raw Output

Full iperf3 output: [`outputs/vm-to-pod-secondary-network-iperf3.out`](../outputs/vm-to-pod-secondary-network-iperf3.out)
