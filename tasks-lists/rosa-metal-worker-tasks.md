# Add Metal Instance Worker to ROSA Cluster

This checklist covers adding an AWS metal instance worker to an existing ROSA cluster for KVM/virtualization support.

## Prerequisites

- [x ] ROSA cluster is running and accessible
- [x ] AWS account has capacity for metal instances in the region
- [x ] rosa CLI installed and configured
- [x ] oc CLI authenticated to the cluster

## Phase 1: Verify Current Cluster State

- [x ] Check current machine pools
  ```bash
  rosa list machinepools --cluster=<cluster-name>
  ```

- [x ] Verify current worker instance types
  ```bash
  # ROSA doesn't expose Machine API - use rosa CLI instead
  rosa list machinepools --cluster=<cluster-name>
  ```
  Shows: workers (m5.xlarge), gpu-workers (g6.xlarge)

- [x ] Check CNV operator is installed (if planning to use VMs)
  ```bash
  oc get csv -n openshift-cnv
  oc get hyperconverged -n openshift-cnv
  ```

## Phase 2: Create Metal Instance Machine Pool

- [x ] List available metal instance types for your region
  Common options:
  - `m5.metal` - 96 vCPU, 384 GiB RAM (general purpose)
  - `m5zn.metal` - 48 vCPU, 192 GiB RAM (high frequency)
  - `c5.metal` - 96 vCPU, 192 GiB RAM (compute optimized)
  - `r5.metal` - 96 vCPU, 768 GiB RAM (memory optimized)
  
  Check pricing and availability: https://aws.amazon.com/ec2/instance-types/

- [x ] Create new machine pool with metal instance type
  ```bash
  rosa create machinepool \
    --cluster=<cluster-name> \
    --name=metal-workers \
    --replicas=1 \
    --instance-type=m5.metal
  ```
  
  **Note**: Metal instances are expensive. Start with `--replicas=1` for testing.

- [x ] Wait for machine pool creation to complete
  ```bash
  rosa list machinepools --cluster=<cluster-name>
  ```

- [x ] Monitor node provisioning
  ```bash
  # Watch for new nodes to appear (ROSA doesn't expose Machine API)
  oc get nodes -w
  ```
  Or check machine pool status:
  ```bash
  rosa describe machinepool metal-workers --cluster=<cluster-name>
  ```

## Phase 3: Verify Metal Worker Node

- [x ] Wait for new node to appear and reach Ready state
  ```bash
  oc get nodes -w
  ```

- [x ] Identify the metal instance node
  ```bash
  oc get nodes -l node.openshift.io/os_id=rhcos -o custom-columns=NAME:.metadata.name,INSTANCE:.metadata.labels.'beta\.kubernetes\.io/instance-type'
  ```
  Look for the node with instance type `m5.metal` (or your chosen type)

- [x ] Verify node has worker role
  ```bash
  oc get node <metal-node-name> --show-labels | grep worker
  ```

- [x ] Check virt-handler is running on metal node (if CNV installed)
  ```bash
  oc get pods -n openshift-cnv -o wide | grep virt-handler | grep <metal-node-name>
  ```§

- [x ] Verify KVM devices are available
  ```bash
  oc describe node <metal-node-name> | grep devices.kubevirt.io/kvm
  ```
  Expected output: `devices.kubevirt.io/kvm: 110` or `1k` (non-zero value)

## Phase 4: Configure Metal Worker for VMs (Optional)

- [x ] Label the metal node for VM workloads
  ```bash
  oc label node <metal-node-name> workload-type=virtualization
  ```

- [ ] Add taint to dedicate node for VMs only (optional)
  ```bash
  oc adm taint node <metal-node-name> workload-type=virtualization:NoSchedule
  ```
  
  **Note**: If you taint the node, you'll need to add tolerations to VM manifests.

- [x ] Verify node capacity and allocatable resources
  ```bash
  oc describe node <metal-node-name> | grep -A 15 "Capacity:"
  ```

## Phase 5: Deploy Test VM to Metal Worker

- [x ] Update VM manifest to target metal worker
  Create `custom-resources/virtual-machine-metal.yaml`:
  ```bash
  cp custom-resources/virtual-machine.yaml custom-resources/virtual-machine-metal.yaml
  ```

- [x ] Add nodeSelector to target metal worker
  Edit `custom-resources/virtual-machine-metal.yaml` and add under `spec.template.spec`:
  ```yaml
  nodeSelector:
    workload-type: virtualization
  # Or use hostname directly:
  # nodeSelector:
  #   kubernetes.io/hostname: <metal-node-name>
  ```

- [ ] If node is tainted, add toleration
  Add under `spec.template.spec`:
  ```yaml
  tolerations:
    - key: "workload-type"
      operator: "Equal"
      value: "virtualization"
      effect: "NoSchedule"
  ```

- [x ] Deploy VM to metal worker
  ```bash
  oc apply -f custom-resources/virtual-machine-metal.yaml
  ```

- [x ] Verify VM is scheduled on metal worker
  ```bash
  oc get vm -n vm-workloads
  oc get vmi -n vm-workloads
  oc get pod -n vm-workloads -o wide | grep virt-launcher
  ```

- [x ] Wait for VM to reach Running state
  ```bash
  oc get vmi test-vm -n vm-workloads -w
  ```

## Phase 6: Validation

- [x ] Access VM console
  ```bash
  virtctl console test-vm -n vm-workloads
  ```

- [x ] Verify VM has IP from UDN
  Inside VM console:
  ```bash
  ip addr show
  ```
  Confirm IP is in `10.100.0.0/24` range

- [x ] Test VM networking
  ```bash
  ping 8.8.8.8
  ping <another-vm-or-pod-ip>
  ```

- [x ] Exit console (Ctrl+] or Ctrl+5)

- [x ] Check VM is using KVM (not emulation)
  ```bash
  # Verify the VM pod is running on the metal worker node
  oc get pod -n vm-workloads -l kubevirt.io/vm=test-vm -o wide
  
  # Check the node has KVM devices available
  oc describe node <metal-node-name> | grep devices.kubevirt.io/kvm
  
  # Verify domain type is 'kvm' not 'qemu'
  oc get vmi test-vm -n vm-workloads -o yaml | grep -A 5 "domain:"
  ```
  Expected: Pod on metal node, KVM devices > 0, CPU model shows `host-model` (KVM acceleration)

## Success Criteria

- [ ] Metal instance worker is in Ready state
- [ ] `devices.kubevirt.io/kvm` shows non-zero allocatable devices on metal node
- [ ] VM successfully deploys and runs on metal worker
- [ ] VM reaches Running state without `ErrorUnschedulable`
- [ ] VM networking is functional via UDN

## Cost Optimization

- [x ] **IMPORTANT**: Metal instances are expensive (~$4-6/hour for m5.metal)
  
- [x ] To stop incurring costs, scale down the machine pool when not in use:
  ```bash
  rosa edit machinepool metal-workers --cluster=<cluster-name> --replicas=0
  ```

- [ ] To resume testing, scale back up:
  ```bash
  rosa edit machinepool metal-workers --cluster=<cluster-name> --replicas=1
  ```

- [ ] To permanently remove the metal worker:
  ```bash
  rosa delete machinepool metal-workers --cluster=<cluster-name>
  ```

## Troubleshooting

### Machine pool creation fails
- Check AWS service quotas for metal instances in your region
- Verify instance type is available: `aws ec2 describe-instance-types --instance-types m5.metal --region <region>`
- Check ROSA cluster has capacity for additional workers

### Node doesn't reach Ready state
- Check machine pool status: `rosa describe machinepool metal-workers --cluster=<cluster-name>`
- Check EC2 instance status in AWS console
- Check node events: `oc describe node <metal-node-name>`

### No KVM devices on metal node
- Verify instance type is actually a `.metal` type: `oc get node <node> -o jsonpath='{.metadata.labels.beta\.kubernetes\.io/instance-type}'`
- Check virt-handler logs: `oc logs -n openshift-cnv <virt-handler-pod-on-metal-node>`
- Verify KVM module: `oc debug node/<metal-node-name> -- chroot /host lsmod | grep kvm`

### VM still shows ErrorUnschedulable
- Verify nodeSelector matches the metal node labels
- Check for node taints: `oc describe node <metal-node-name> | grep Taints`
- If tainted, ensure VM has appropriate tolerations

## Notes

- Metal instances are significantly more expensive than standard instances
- Metal instances provide bare-metal performance for VMs (no nested virtualization penalty)
- ROSA machine pools automatically handle instance lifecycle and replacement
- Machine Config Operator manages OS configuration for all workers (including metal)
- Consider using auto-scaling with min=0, max=1 to automatically stop when not in use
