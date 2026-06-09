# SSH into a KubeVirt VM on a DPU Worker Node

## The Problem

On DPU-accelerated worker nodes, the VM's pod network traffic flows through the DPU's OVS bridge, not through the host's network stack. This means:

- `virtctl console` hangs (websocket can't reach the virt-launcher pod)
- `oc port-forward` to the VM fails (pod IP unreachable from the node)
- NodePort services don't route to the VM
- Direct SSH from your laptop to the pod IP doesn't work

## Solution: Jump Pod on the Same DPU Worker

Deploy a pod on the same DPU worker node as the VM. Pod-to-pod traffic on the same DPU worker flows through the DPU's OVS bridge and works correctly.

### Step 1: Create a Jump Pod

```bash
oc run jump --image=nicolaka/netshoot -n <vm-namespace> --restart=Never \
  --overrides='{
    "spec": {
      "nodeSelector": {
        "kubernetes.io/hostname": "<dpu-worker-node>"
      },
      "tolerations": [
        {"key": "nvidia.com/gpu", "operator": "Exists", "effect": "NoSchedule"}
      ],
      "securityContext": {
        "runAsNonRoot": true,
        "seccompProfile": {"type": "RuntimeDefault"}
      },
      "containers": [{
        "name": "jump",
        "image": "nicolaka/netshoot",
        "command": ["sleep", "infinity"],
        "securityContext": {
          "allowPrivilegeEscalation": false,
          "capabilities": {"drop": ["ALL"]}
        }
      }]
    }
  }'
```

**Example** (VM `test-vm-udn` on srv23 in `iperf3-test` namespace):

```bash
oc run jump --image=nicolaka/netshoot -n iperf3-test --restart=Never \
  --overrides='{
    "spec": {
      "nodeSelector": {
        "kubernetes.io/hostname": "nvd-srv-23.nvidia.eng.rdu2.dc.redhat.com"
      },
      "tolerations": [
        {"key": "nvidia.com/gpu", "operator": "Exists", "effect": "NoSchedule"}
      ],
      "securityContext": {
        "runAsNonRoot": true,
        "seccompProfile": {"type": "RuntimeDefault"}
      },
      "containers": [{
        "name": "jump",
        "image": "nicolaka/netshoot",
        "command": ["sleep", "infinity"],
        "securityContext": {
          "allowPrivilegeEscalation": false,
          "capabilities": {"drop": ["ALL"]}
        }
      }]
    }
  }'
```

### Step 2: Verify the Jump Pod Is Running on the Correct Node

```bash
oc get pod jump -n <vm-namespace> -o wide
```

Confirm the `NODE` column shows the same DPU worker as the VM.

### Step 3: Get the VM's Pod IP

```bash
oc get vmi <vm-name> -n <vm-namespace> -o jsonpath='{.status.interfaces[0].ipAddress}'
```

### Step 4: SSH into the VM

```bash
oc exec -it jump -n <vm-namespace> -- ssh -o StrictHostKeyChecking=no <user>@<vm-pod-ip>
```

**Example:**

```bash
oc exec -it jump -n iperf3-test -- ssh -o StrictHostKeyChecking=no fedora@10.130.2.16
# password: fedora
```

### Cleanup

When done, delete the jump pod:

```bash
oc delete pod jump -n <vm-namespace>
```

## Alternative: Guest Agent Commands (No SSH Needed)

If you just need to run commands inside the VM without an interactive session, the QEMU guest agent works over a virtio-serial channel (no network required):

```bash
# Run a command
oc exec <virt-launcher-pod> -n <vm-namespace> -c compute -- \
  virsh -c qemu+unix:///session qemu-agent-command <domain-name> \
  '{"execute":"guest-exec","arguments":{"path":"<command>","arg":["<args>"],"capture-output":true}}'

# Get the result (use the PID from the previous command's output)
oc exec <virt-launcher-pod> -n <vm-namespace> -c compute -- \
  virsh -c qemu+unix:///session qemu-agent-command <domain-name> \
  '{"execute":"guest-exec-status","arguments":{"pid":<pid>}}'
```

The output is base64-encoded in the `out-data` field. Decode with `echo "<base64>" | base64 -d`.

**Example** (check listening ports inside the VM):

```bash
# Step 1: Execute the command
oc exec virt-launcher-test-vm-udn-s9qgl -n iperf3-test -c compute -- \
  virsh -c qemu+unix:///session qemu-agent-command iperf3-test_test-vm-udn \
  '{"execute":"guest-exec","arguments":{"path":"/usr/bin/ss","arg":["-tlnp"],"capture-output":true}}'
# Returns: {"return":{"pid":1234}}

# Step 2: Fetch the result
oc exec virt-launcher-test-vm-udn-s9qgl -n iperf3-test -c compute -- \
  virsh -c qemu+unix:///session qemu-agent-command iperf3-test_test-vm-udn \
  '{"execute":"guest-exec-status","arguments":{"pid":1234}}'
# Returns: {"return":{"exitcode":0,"out-data":"<base64>", ...}}

# Step 3: Decode the output
echo "<base64>" | base64 -d
```

## Why This Happens

In a DPF-accelerated cluster, the DPU's eSwitch operates in switchdev mode. Pod traffic is handled by VF representors on the DPU's OVS bridge (`br-sfc` → `br-ovn` → `br-dpu`). The host node's kernel network stack is not in the datapath for pod-to-pod traffic. This means:

| Path | Works? | Why |
|------|--------|-----|
| Pod → Pod (same DPU worker) | Yes | Both VF representors are on the same OVS bridge |
| Pod → Pod (different DPU workers) | Yes | OVN-K tunnels through the DPU uplinks |
| Node → Pod (on DPU worker) | No | Node traffic doesn't go through the DPU's OVS bridge |
| External → NodePort → Pod | No | NodePort uses kube-proxy/iptables on the node, which can't reach the pod |
| `virtctl console` | Hangs | Websocket proxy ultimately needs network path to virt-launcher |
| `oc port-forward` | Fails | Same as node → pod — traffic can't reach the pod IP |
| Guest agent (`virsh qemu-agent-command`) | Yes | Uses virtio-serial channel, bypasses the network entirely |
