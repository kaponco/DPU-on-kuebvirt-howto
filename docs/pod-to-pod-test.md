# Pod-to-Pod iperf3 Test with DPU Offload

## Overview

This document describes how to run a pod-to-pod network performance test between two DPU worker nodes using iperf3, verifying DPU hardware offload.

## Prerequisites

- OpenShift cluster with DPU workers provisioned
- DPF operator installed and configured
- `oc` CLI authenticated to the cluster
- Two DPU worker nodes available (e.g., nvd-srv-23 and nvd-srv-27)
- Access to `kubeconfig_dpu` for offload verification

## Setup

### 1. Create the Namespace

```bash
oc create namespace iperf3-test
```

### 2. Deploy the Server Pod

Deploy the iperf3 server on the first DPU worker node:

```bash
oc apply -f custom-resources/iperf3-server-pod-srv27.yaml
```

The server pod uses:
- Image: `networkstatic/iperf3`
- Mode: `iperf3 -s` (server mode, listening on port 5201)
- nodeSelector targeting the first DPU worker

### 3. Deploy the Client Pod

Deploy the iperf3 client on the second DPU worker node:

```bash
oc apply -f custom-resources/iperf3-client-pod-srv23.yaml
```

The client pod uses:
- Image: `networkstatic/iperf3`
- Mode: `sleep infinity` (keeps running for manual test execution)
- nodeAffinity targeting the second DPU worker
- podAntiAffinity to ensure it runs on a different node than the server

### 4. Verify Pod Placement

Wait for both pods to be running on separate DPU worker nodes:

```bash
oc get pods -n iperf3-test -o wide
```

Expected output:
```
NAME                  READY   STATUS    IP            NODE
iperf3-client         1/1     Running   10.130.X.X    <dpu-worker-1>
iperf3-server-srv27   1/1     Running   10.128.X.X    <dpu-worker-2>
```

### Network Configuration

Both pods use the **default cluster network** (OVN-Kubernetes). No UserDefinedNetwork annotations or explicit VF resource requests are needed. On DPU host workers, the default OVN-Kubernetes network traffic is automatically offloaded through the DPU hardware.

## Test Steps

### 1. Get the Server IP

```bash
SERVER_IP=$(oc get pod iperf3-server-srv27 -n iperf3-test -o jsonpath='{.status.podIP}')
echo "Server IP: $SERVER_IP"
```

### 2. Run a Basic TCP Throughput Test (10 seconds)

```bash
oc exec -n iperf3-test iperf3-client -- iperf3 -c $SERVER_IP -t 10
```

### 3. Run a Multi-Stream Test (optional)

```bash
oc exec -n iperf3-test iperf3-client -- iperf3 -c $SERVER_IP -t 10 -P 4
```

### 4. Verify DPU Offload

Confirm the traffic is hardware-offloaded by checking the DPU connection details annotation on each pod:

```bash
oc get pod iperf3-client -n iperf3-test \
  -o jsonpath='{.metadata.annotations.k8s\.ovn\.org/dpu\.connection-details}' | jq '.'

oc get pod iperf3-server-srv27 -n iperf3-test \
  -o jsonpath='{.metadata.annotations.k8s\.ovn\.org/dpu\.connection-details}' | jq '.'
```

Each pod should show a VF assignment:
```json
{
  "default": {
    "pfId": "0",
    "vfId": "<vf-number>",
    "sandboxId": "<container-id>",
    "vfNetdevName": "<device-name>"
  }
}
```

To verify hardware-offloaded flows during an active test, use the DPU cluster kubeconfig:

```bash
export KUBECONFIG=kubeconfig_dpu

# Find the OVN pod on the relevant DPU node
OVN_POD=$(oc get pods -n dpf-operator-system -l app.kubernetes.io/name=ovn-kubernetes \
  -o jsonpath='{.items[0].metadata.name}')

# Dump hardware-offloaded flows and filter by pod IP
oc exec -n dpf-operator-system $OVN_POD -c ovn-controller -- \
  ovs-appctl dpctl/dump-flows type=offloaded | grep "<pod-ip>"
```

Offloaded flows will show:
- `type=offloaded` confirming hardware acceleration
- Flow flags with `S` (seen by hardware)
- High packet/byte counts with sub-second `used` times
- Geneve tunnel encapsulation between DPU VTEP IPs

## Cleanup

```bash
oc delete pod iperf3-client -n iperf3-test
oc delete pod iperf3-server-srv27 -n iperf3-test
oc delete namespace iperf3-test
```

Verify cleanup:
```bash
oc get pods -n iperf3-test
# Should return: No resources found in iperf3-test namespace.
```

---

## Test Results (2026-06-07)

- **Cluster**: OpenShift 4.20 with DPF v26.4.0
- **DPU Hardware**: NVIDIA BlueField-3
- **DPU Workers**: nvd-srv-23 (client), nvd-srv-27 (server)
- **Network**: Default cluster network (OVN-Kubernetes with DPU offload)

### iperf3 Output

```
Connecting to host 10.128.2.14, port 5201
[  5] local 10.130.2.11 port 33826 connected to 10.128.2.14 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  6.14 GBytes  52.8 Gbits/sec  195   3.67 MBytes
[  5]   1.00-2.00   sec  7.42 GBytes  63.8 Gbits/sec    0   3.82 MBytes
[  5]   2.00-3.00   sec  7.24 GBytes  62.2 Gbits/sec    0   3.84 MBytes
[  5]   3.00-4.00   sec  7.43 GBytes  63.9 Gbits/sec    0   3.84 MBytes
[  5]   4.00-5.00   sec  7.33 GBytes  62.9 Gbits/sec    0   3.84 MBytes
[  5]   5.00-6.00   sec  7.36 GBytes  63.2 Gbits/sec    0   3.84 MBytes
[  5]   6.00-7.00   sec  7.36 GBytes  63.2 Gbits/sec    0   3.84 MBytes
[  5]   7.00-8.00   sec  7.28 GBytes  62.5 Gbits/sec   27   3.78 MBytes
[  5]   8.00-9.00   sec  7.40 GBytes  63.6 Gbits/sec    0   3.81 MBytes
[  5]   9.00-10.00  sec  7.25 GBytes  62.3 Gbits/sec    0   3.84 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  72.2 GBytes  62.0 Gbits/sec  223             sender
[  5]   0.00-10.00  sec  72.2 GBytes  62.0 Gbits/sec                  receiver
```

**Result: 62.0 Gbits/sec** average throughput, 72.2 GBytes transferred, 223 retransmits.

### DPU Offload Evidence

Pod VF assignments:

| Pod | Node | VF ID | VF Netdev |
|-----|------|-------|-----------|
| iperf3-client | nvd-srv-23 | 32 | dev44 |
| iperf3-server-srv27 | nvd-srv-27 | 44 | dev147 |

Hardware-offloaded flow statistics captured during the test on srv23's DPU:
- **2.85 million packets / 25.3 GB** processed in hardware (client → server direction)
- Traffic encapsulated in **Geneve tunnels** between DPU VTEP IPs (10.6.156.201 ↔ 10.6.156.193)
- VF representor ports (`pf0vf32`, `pf0vf44`) attached to `br-int` as DPDK ports

### Traffic Path

```
┌─────────────────────────────────────┐     Geneve Tunnel      ┌─────────────────────────────────────┐
│  nvd-srv-23 (Host)                  │  (DPU VTEP 10.6.156.x) │  nvd-srv-27 (Host)                  │
│                                     │                        │                                     │
│  ┌──────────────┐                   │                        │                   ┌──────────────┐  │
│  │ iperf3-client│                   │                        │                   │iperf3-server │  │
│  │ 10.130.2.11  │                   │                        │                   │ 10.128.2.14  │  │
│  └──────┬───────┘                   │                        │                   └──────┬───────┘  │
│         │ VF32                      │                        │                     VF44 │          │
│  ┌──────▼───────────────────────┐   │                        │   ┌──────────────────────▼───────┐  │
│  │  BlueField-3 DPU             │   │                        │   │  BlueField-3 DPU             │  │
│  │  pf0vf32 → br-int → Geneve ─┼───┼────────────────────────┼───┼─ Geneve → br-int → pf0vf44   │  │
│  │  (hardware offload)          │   │                        │   │  (hardware offload)           │  │
│  └──────────────────────────────┘   │                        │   └──────────────────────────────┘  │
└─────────────────────────────────────┘                        └─────────────────────────────────────┘
```

## Troubleshooting

### Pods Stuck in ContainerCreating

If pods are stuck in ContainerCreating on a DPU worker node, check for DOCA flow API errors on the DPU:

```bash
export KUBECONFIG=kubeconfig_dpu
oc exec -n dpf-operator-system <ovn-pod-on-affected-node> -c ovn-controller -- \
  bash -c 'grep "DOCA init flow api error" /var/log/openvswitch/ovs-vswitchd.log | tail -5'
```

If you see `Operation not permitted` errors, restart OVS on the DPU:

```bash
oc exec -n dpf-operator-system <ovn-pod-on-affected-node> -c ovn-controller -- \
  ovs-appctl exit --cleanup
```

The DaemonSet will restart the pod automatically with a fresh OVS/DOCA state.

### Primary UDN Namespace Constraint

Do not use namespaces that have a Primary UserDefinedNetwork configured. The namespace label `k8s.ovn.org/primary-user-defined-network` is immutable once set, and forces all pods to require VF resources for the primary network, which does not work for regular pods. Use a clean namespace without a Primary UDN for pod-to-pod testing.
