Got it. You want to parameterize only the `duration` and `renewBefore` fields, leaving the rest of the configuration static. Here's how you can achieve that:

1. **Create the Helm Chart Directory Structure**:
   - Create a directory for your Helm chart (e.g., `linkerd-cert-manager`).
   - Inside this directory, create the following subdirectories and files:
     - `templates/`
     - `values.yaml`
     - `Chart.yaml`

2. **Define the Chart Metadata in `Chart.yaml`**:
   ```yaml
   apiVersion: v2
   name: linkerd-cert-manager
   description: A Helm chart to deploy cert-manager Issuer and Certificate for Linkerd
   version: 0.1.0
   ```

3. **Create the `values.yaml` File**:
   ```yaml
   certificate:
     duration: 48h
     renewBefore: 25h
   ```

4. **Create the Issuer Template in `templates/issuer.yaml`**:
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: linkerd-trust-anchor
     namespace: linkerd
   spec:
     ca:
       secretName: linkerd-trust-anchor
   ```

5. **Create the Certificate Template in `templates/certificate.yaml`**:
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: linkerd-identity-issuer
     namespace: linkerd
   spec:
     secretName: linkerd-identity-issuer
     duration: {{ .Values.certificate.duration }}
     renewBefore: {{ .Values.certificate.renewBefore }}
     issuerRef:
       name: linkerd-trust-anchor
       kind: Issuer
     commonName: identity.linkerd.cluster.local
     dnsNames:
       - identity.linkerd.cluster.local
     isCA: true
     privateKey:
       algorithm: ECDSA
     usages:
       - cert sign
       - crl sign
       - server auth
       - client auth
   ```

6. **Package and Deploy the Helm Chart**:
   - Navigate to the directory containing your Helm chart.
   - Package the Helm chart using:
     ```sh
     helm package .
     ```
   - Deploy the Helm chart using:
     ```sh
     helm install linkerd-cert-manager ./linkerd-cert-manager-0.1.0.tgz
     ```

This configuration ensures that only the `duration` and `renewBefore` fields are parameterized, while the rest of the YAML manifests remain static.
