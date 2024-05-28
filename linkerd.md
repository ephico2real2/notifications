To parameterize the provided YAML manifests as a Helm chart, you can follow these steps:

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
   issuer:
     name: linkerd-trust-anchor
     namespace: linkerd
     secretName: linkerd-trust-anchor

   certificate:
     name: linkerd-identity-issuer
     namespace: linkerd
     secretName: linkerd-identity-issuer
     duration: 48h
     renewBefore: 25h
     commonName: identity.linkerd.cluster.local
     dnsNames:
       - identity.linkerd.cluster.local
     isCA: true
     privateKeyAlgorithm: ECDSA
     usages:
       - cert sign
       - crl sign
       - server auth
       - client auth
   ```

4. **Create the Issuer Template in `templates/issuer.yaml`**:
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: {{ .Values.issuer.name }}
     namespace: {{ .Values.issuer.namespace }}
   spec:
     ca:
       secretName: {{ .Values.issuer.secretName }}
   ```

5. **Create the Certificate Template in `templates/certificate.yaml`**:
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: {{ .Values.certificate.name }}
     namespace: {{ .Values.certificate.namespace }}
   spec:
     secretName: {{ .Values.certificate.secretName }}
     duration: {{ .Values.certificate.duration }}
     renewBefore: {{ .Values.certificate.renewBefore }}
     issuerRef:
       name: {{ .Values.issuer.name }}
       kind: Issuer
     commonName: {{ .Values.certificate.commonName }}
     dnsNames:
     {{- range .Values.certificate.dnsNames }}
       - {{ . }}
     {{- end }}
     isCA: {{ .Values.certificate.isCA }}
     privateKey:
       algorithm: {{ .Values.certificate.privateKeyAlgorithm }}
     usages:
     {{- range .Values.certificate.usages }}
       - {{ . }}
     {{- end }}
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

With these steps, you have parameterized the provided YAML manifests into a Helm chart, allowing for customization via the `values.yaml` file. You can modify the values as needed for different environments.
