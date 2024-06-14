### Jira Story: Implement Static Egress IP for a Namespace in OpenShift

**Summary:**
Implement static egress IP assignment for a specific namespace in OpenShift, using three distinct IPs. Apply necessary labels to infra nodes to ensure they are used for egress traffic. Verify available IP addresses from the internal IPAM tool (NAMS) to ensure they match the public IP address range CIDR assigned to the clusters.

**Description:**
As part of network management and compliance, we need to assign three distinct egress IPs to a specific namespace in OpenShift. The infra nodes should be labeled appropriately to ensure they are used for egress traffic. The implementation includes verifying available IP addresses from the internal IPAM tool (NAMS) and ensuring they are within the assigned public IP address range CIDR. The application owner or requestor will then need to coordinate with the network team to use these newly assigned IPs for firewall considerations.

**Acceptance Criteria:**
1. **IP Allocation Verification**:
   - Check available IP address allocations from the internal IPAM tool (NAMS) to ensure they are within the public IP address range CIDR assigned to the clusters.
   - Identify and document three available IP addresses for use.

2. **Node Labeling**:
   - Apply the `k8s.ovn.org/egress-assignable=true` label to all infra nodes to ensure they are used for egress traffic.

3. **Egress IP YAML Configuration**:
   - Create an EgressIP resource YAML with the three distinct IPs assigned to the specified namespace.
   - Use `matchExpressions` for the namespace selector.

4. **Documentation and Handover**:
   - Document the assigned IPs and provide instructions to the application owner or requestor for coordinating with the network team for firewall considerations.

**Tasks:**
1. **Verify Available IPs from NAMS**:
   - Access the internal IPAM tool (NAMS) to check available IP addresses.
   - Ensure the IPs match the public IP address range CIDR assigned to the clusters.
   - Document the three IP addresses to be used.

2. **Apply Node Labels**:
   - Identify the infra nodes using `oc get nodes`.
   - Apply the label to the infra nodes:
     ```sh
     oc label node <node1> k8s.ovn.org/egress-assignable=true
     oc label node <node2> k8s.ovn.org/egress-assignable=true
     # Repeat for all infra nodes
     ```

3. **Create EgressIP YAML File**:
   - Create a YAML file named `egress-ip-config.yaml` with the following content:
     ```yaml
     apiVersion: k8s.ovn.org/v1
     kind: EgressIP
     metadata:
       name: egress-ip-namespace
     spec:
       egressIPs:
         - <IP1>
         - <IP2>
         - <IP3>
       namespaceSelector:
         matchExpressions:
           - key: kubernetes.io/metadata.name
             operator: In
             values:
               - <namespace-name>
     ```

4. **Deploy EgressIP Configuration**:
   - Apply the EgressIP configuration to the cluster:
     ```sh
     oc apply -f egress-ip-config.yaml
     ```

5. **Verification**:
   - Verify the egress IP assignment using:
     ```sh
     oc get egressips
     ```
   - Ensure the egress IPs are assigned to the infra nodes.

6. **Documentation and Handover**:
   - Document the assigned IPs and provide instructions for coordinating with the network team for firewall considerations.
   - Ensure the application owner or requestor has all the necessary information.

**Assumptions:**
- Infra nodes are permanently part of the cluster and will not be scaled down.
- The internal IPAM tool (NAMS) is accessible and up-to-date with available IP addresses.

**Dependencies:**
- Access to the internal IPAM tool (NAMS).

**Comments:**
- Ensure the application owner or requestor understands the next steps for coordinating with the network team.
- Document any changes or issues encountered during the implementation.

