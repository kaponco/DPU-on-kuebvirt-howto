# VM Deployment Task List

This checklist covers all steps from an existing OpenShift cluster to running a VM on a worker node.

## Phase 1: CNV Operator Installation

- [x] Create openshift-cnv namespace
  ```bash
  oc create namespace openshift-cnv
  ```

- [x] Create OperatorGroup for openshift-cnv namespace
  ```bash
  oc apply -f custom-resources/operatorgroup-cnv.yaml
  ```

- [x] Install OpenShift Virtualization Operator via Subscription
  ```bash
  oc apply -f custom-resources/subscription-cnv.yaml
  ```

- [x] Wait for operator installation to complete
  ```bash
  oc get csv -n openshift-cnv -w
  ```
  - Wait until CSV shows "Succeeded" phase

- [x] Create HyperConverged Custom Resource (CR)
  ```bash
  oc apply -f custom-resources/hyperconverged.yaml
  ```

- [x] Wait for HyperConverged deployment to complete (may take several minutes)
  ```bash
  oc wait --for=condition=Available hyperconverged/kubevirt-hyperconverged -n openshift-cnv --timeout=10m
  ```
  - Command will block until Available condition is True or timeout expires

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

## Phase 2: DPF and SR-IOV Configuration

**Note**: This phase is required for DPU-accelerated networking. Skip this phase if you don't have DPU hardware or don't need hardware acceleration.

- [ ] Create dpf-operator-system namespace
  ```bash
  oc create namespace dpf-operator-system
  ```

- [ ] Create image pull secret for DPF images
  ```bash
  oc create secret docker-registry dpf-pull-secret \
    --docker-server=<registry-url> \
    --docker-username=<username> \
    --docker-password=<password> \
    -n dpf-operator-system
  ```
  - Replace `<registry-url>`, `<username>`, and `<password>` with your credentials
  - This is required to pull NVIDIA DPF container images

- [ ] Install DPF Operator
  **Note**: DPF installation typically requires the automated framework from [rh-ecosystem-edge/openshift-dpf](https://github.com/rh-ecosystem-edge/openshift-dpf). See `docs/openshift-dpf-readme.md` for automated installation instructions.
  
  For reference, the operator should be deployed in `dpf-operator-system` namespace.

- [ ] Create DPFOperatorConfig
  ```bash
  oc apply -f custom-resources/dpfoperatorconfig.yaml
  ```
  - Configures DPF operator settings including SR-IOV device plugin
  - Adjust `kubernetesAPIServerVIP` in the YAML to match your cluster's API endpoint

- [ ] Wait for DPF operator to be ready
  ```bash
  oc wait --for=condition=Ready dpfoperatorconfig/dpfoperatorconfig -n dpf-operator-system --timeout=10m
  ```

- [ ] Verify DPF operator components are running
  ```bash
  oc get deployment -n dpf-operator-system
  ```
  - Check that `dpf-operator-controller-manager`, `dpf-provisioning-controller-manager`, and other components are ready

- [ ] Create DPUCluster for static cluster management
  ```bash
  oc apply -f custom-resources/dpucluster.yaml
  ```
  - Defines the DPU cluster with max nodes and static configuration

- [ ] Wait for DPUCluster to be ready
  ```bash
  oc get dpucluster doca -n dpf-operator-system -o jsonpath='{.status.phase}'
  ```
  - Should show "Ready" when complete

- [ ] Configure SR-IOV Virtual Functions
  ```bash
  oc apply -f custom-resources/nodesriovdevicepluginconfig.yaml
  ```
  - Creates VF pools for management (vfs-mgmt), UDN (vfs-udn), and general use (vfs)
  - Adjust VF ranges based on your BlueField-3 configuration

- [ ] Verify DPU devices are discovered
  ```bash
  oc get dpudevices -n dpf-operator-system
  ```
  - Should show your BlueField-3 devices

- [ ] Verify DPU host workers are labeled
  ```bash
  oc get nodes -l node-role.kubernetes.io/dpu-host-worker
  ```
  - DPU host workers should have `dpu-host-worker` and `worker-dpf` labels

- [ ] Verify SR-IOV device plugin is running
  ```bash
  oc get pods -n dpf-operator-system | grep sriov
  ```

## Phase 3: User Defined Network (UDN) Configuration

- [ ] Create namespace for VM workloads
  ```bash
  oc create namespace vm-workloads
  ```

- [ ] Create UserDefinedNetwork Custom Resource
  - Define network topology (Layer2 or Layer3)
  - Specify subnet configuration
  - Apply the CR to target namespace
  ```bash
  oc apply -f custom-resources/user-defined-network.yaml -n vm-workloads
  ```

- [ ] Verify UDN is created successfully
  ```bash
  oc get userdefinednetwork -n vm-workloads
  oc describe userdefinednetwork vm-udn -n vm-workloads
  ```

- [ ] Validate OVN-Kubernetes components recognize the UDN
  ```bash
  oc get network-attachment-definition -n vm-workloads
  ```

## Phase 4: Virtual Machine Deployment

- [ ] Create Virtual Machine manifest
  - Configure VM specs (CPU, memory, storage)
  - Reference the UDN in network interface configuration
  - Includes nodeSelector for DPU host worker nodes
  - Template available at: `custom-resources/virtual-machine-dpu.yaml`

- [ ] Apply VM manifest
  ```bash
  oc apply -f custom-resources/virtual-machine-dpu.yaml -n vm-workloads
  ```

- [ ] Verify VM is created and starting
  ```bash
  oc get vm -n vm-workloads
  oc get vmi -n vm-workloads
  ```

- [ ] Wait for VM to reach Running state
  ```bash
  oc get vmi test-vm -n vm-workloads -w
  ```

- [ ] Verify VM pod is scheduled on a DPU host worker node
  ```bash
  oc get pod -n vm-workloads -o wide | grep virt-launcher
  ```
  - Confirm the NODE column shows a `dpu-host-worker` node (e.g., nvd-srv-22 or nvd-srv-23)

## Phase 5: Network Validation

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
- [ ] DPF operator is running with SR-IOV device plugin configured
- [ ] DPU devices are discovered and DPU host workers are labeled correctly
- [ ] VM is running on a DPU host worker node with hardware-accelerated networking
- [ ] VM can communicate across the UDN with DPU offload
- [ ] VM is accessible via console

## Notes

- DPF installation typically uses the automated framework from [rh-ecosystem-edge/openshift-dpf](https://github.com/rh-ecosystem-edge/openshift-dpf)
- See `docs/openshift-dpf-readme.md` for automated installation instructions
- The custom resources in this checklist represent the configuration after DPF installation
- Adjust MTU, VIP, and VF ranges in the custom resources to match your environment
- Document any issues or variations encountered during deployment
