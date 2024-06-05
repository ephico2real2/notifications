Sure, here is the updated draft PR including the creation of a self-signed ClusterIssuer:

---

**Title: Integrate Linkerd with Cert Manager Using Self-Signed ClusterIssuer and Helm Templates**

**Description:**

This PR implements the integration of Linkerd with Cert Manager using a self-signed ClusterIssuer. The setup has been converted into Helm templates for easier deployment and management. Additionally, we added a Trust Manager Helm chart to automate the extraction and replication of certificates across the cluster.

**High-Level Overview:**

1. **Cert Manager Integration with Linkerd:**
   - Integrated Cert Manager with Linkerd to manage certificates efficiently.

2. **Created a Self-Signed ClusterIssuer:**
   - Created a self-signed ClusterIssuer to serve as the initial certificate authority.

3. **Root Certificate Creation:**
   - Generated a root certificate (`linkerd-trust-anchor`) using the self-signed ClusterIssuer.
   - Stored the root certificate in a Kubernetes secret named `linkerd-trust-anchor`.

4. **Intermediate Issuer Configuration:**
   - Configured an intermediate ClusterIssuer using the `linkerd-trust-anchor` secret.
   - Used the intermediate ClusterIssuer to create the Linkerd identity issuer.

5. **Trust Manager Integration:**
   - Added a Trust Manager Helm chart that deploys the Trust CRDs.
   - The Trust Manager extracts the `ca.crt` certificate from the `linkerd-trust-anchor` secret.
   - This certificate is converted into a ConfigMap (`linkerd-identity-trust-roots`) with the key `ca-bundle.crt`.
   - The ConfigMap is replicated across all namespaces in the cluster, ensuring consistent trust roots.

6. **Automation and Efficiency:**
   - Previously, the ROOT CA was manually created and uploaded to Google Secret Manager.
   - The new solution automates the integration with Cert Manager to generate the root CA, eliminating the need for manual creation and fostering faster adoption.
   - The `linkerd-identity-trust-roots` ConfigMap no longer needs to be created manually, streamlining the process and reducing potential errors.

**Benefits:**
- Improved certificate management and security within the Kubernetes cluster.
- Automated certificate generation and replication, reducing manual efforts and potential mistakes.
- Enhanced scalability and adoption of the solution across different environments.

**File Structure:**
- `charts/linkerd-os-control-plane/templates/1-cert-manager-selfsigned-cluster-issuer.yaml`
- `charts/linkerd-os-control-plane/templates/2-linkerd-root-ca.yaml`
- `charts/linkerd-os-control-plane/templates/3-linkerd-intermediate-issuer.yaml`
- `charts/linkerd-os-control-plane/templates/4-linkerd-identity-issuer.yaml`
- `charts/linkerd-os-control-plane/templates/5-linkerd-identity-trust-roots-cm.yaml`

**Conclusion:**
This integration not only simplifies the certificate management process but also ensures that all namespaces within the cluster have consistent trust roots, thereby improving the overall security and efficiency of the deployment.

---
