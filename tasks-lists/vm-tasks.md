# VM Deployment Task List

This checklist covers all steps from an existing OpenShift cluster to running a VM on a worker node.

## Phase 1: CNV Operator Installation

- [x] Install OpenShift Virtualization Operator from OperatorHub
  - Navigate to OperatorHub in the OpenShift Console
  - Search for "OpenShift Virtualization"
  - Install the operator

- [x] Create HyperConverged Custom Resource (CR)
  - Create the CR to deploy virtualization components
  - Wait for deployment to complete

- [x] Verify virt-handler pods are running on all eligible worker nodes
  ```bash
  oc get pods -n openshift-cnv | grep virt-handler
  ```
  - Confirm at least one virt-handler pod per worker node
  - All pods should be in Running state

- [x] Verify overall CNV operator health
  ```bash
  oc get csv -n openshift-cnv
  oc get hyperconverged -n openshift-cnv
  ```

## Phase 2: User Defined Network (UDN) Configuration

- [x] Create namespace for VM workloads
  ```bash
  oc create namespace vm-workloads
  ```

- [x] Create UserDefinedNetwork Custom Resource
  - Define network topology (Layer2 or Layer3)
  - Specify subnet configuration
  - Apply the CR to target namespace
  ```bash
  oc apply -f custom-resources/user-defined-network.yaml
  ```

- [x] Verify UDN is created successfully
  ```bash
  oc get userdefinednetwork -n vm-workloads
  oc describe userdefinednetwork vm-udn -n vm-workloads
  ```

- [x] Validate OVN-Kubernetes components recognize the UDN
  ```bash
  oc get network-attachment-definition -n vm-workloads
  ```

## Phase 3: Virtual Machine Deployment

- [x] Create Virtual Machine manifest
  - Configure VM specs (CPU, memory, storage)
  - Reference the UDN in network interface configuration
  - Set node affinity if targeting specific workers (optional for now)
  - Template available at: `custom-resources/virtual-machine.yaml`

- [x] Apply VM manifest
  ```bash
  oc apply -f custom-resources/virtual-machine.yaml
  ```

- [x] Verify VM is created and starting
  ```bash
  oc get vm -n vm-workloads
  oc get vmi -n vm-workloads
  ```

- [ ] Wait for VM to reach Running state
  ```bash
  oc get vmi test-vm -n vm-workloads -w
  ```

- [ ] Verify VM pod is scheduled on a worker node
  ```bash
  oc get pod -n vm-workloads -o wide | grep virt-launcher
  ```

## Phase 4: Network Validation

- [ ] Access VM console
  ```bash
  virtctl console test-vm -n vm-workloads
  ```

- [ ] Verify VM has IP address from UDN subnet
  - Check IP configuration inside VM
  - Confirm IP is within the UDN subnet range

- [ ] Test network connectivity within the UDN
  - Ping gateway (if applicable)
  - Test connectivity to other VMs/pods on same UDN

- [ ] Verify OVN pipeline configuration
  ```bash
  oc get pod -n openshift-ovn-kubernetes -o wide
  ```

## Success Criteria

- [ ] CNV Operator is successfully deployed and healthy
- [ ] VM on standard OVN-K UDN can communicate across the network
- [ ] VM is running on a worker node and accessible via console

## Notes

- DPU worker integration will be added as a separate task list
- SR-IOV configuration is a future enhancement
- Document any issues or variations encountered during deployment
