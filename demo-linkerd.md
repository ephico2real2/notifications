I apologize for the mistake. You are correct that the `gcloud iam service-accounts add-iam-policy-binding` command should be used to bind the IAM role to the service account. Let's correct this in the `setup_workload_identity.sh` script.

### Corrected `setup_workload_identity.sh`

```bash
#!/bin/bash

set -e

PROJECT_ID=your-project-id
SERVICE_ACCOUNT_NAME=artifact-registry-sa
NAMESPACE=linkerd-poc
KSA_NAME=artifact-registry-ksa
REGION=us-central
REPO_NAME=your-repo-name
SERVICE_ACCOUNT_EMAIL=$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com

# Enable Workload Identity on the GKE cluster
gcloud container clusters update <cluster-name> \
    --workload-pool=$PROJECT_ID.svc.id.goog

# Create Google Cloud IAM service account
if ! gcloud iam service-accounts list --filter="email:$SERVICE_ACCOUNT_EMAIL" --format="value(email)"; then
    gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME \
        --display-name "Service Account for Artifact Registry"
else
    echo "Service account $SERVICE_ACCOUNT_NAME already exists."
fi

# Grant permissions to the IAM service account
if ! gcloud projects get-iam-policy $PROJECT_ID --flatten="bindings[].members" --format="table(bindings.members)" | grep $SERVICE_ACCOUNT_EMAIL; then
    gcloud iam service-accounts add-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
        --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]" \
        --role "roles/artifactregistry.reader"
else
    echo "Service account $SERVICE_ACCOUNT_EMAIL already has the role roles/artifactregistry.reader."
fi

# Create Kubernetes service account
kubectl create serviceaccount $KSA_NAME --namespace $NAMESPACE || echo "Kubernetes service account $KSA_NAME already exists."

# Bind the Kubernetes service account to the Google IAM service account
gcloud iam service-accounts add-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
    --role "roles/iam.workloadIdentityUser" \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]"

# Annotate the Kubernetes service account
kubectl annotate serviceaccount $KSA_NAME \
    --namespace $NAMESPACE \
    iam.gke.io/gcp-service-account=$SERVICE_ACCOUNT_EMAIL --overwrite
```

### Complete Scripts and Makefile

### `setup_workload_identity.sh`

```bash
#!/bin/bash

set -e

PROJECT_ID=your-project-id
SERVICE_ACCOUNT_NAME=artifact-registry-sa
NAMESPACE=linkerd-poc
KSA_NAME=artifact-registry-ksa
REGION=us-central
REPO_NAME=your-repo-name
SERVICE_ACCOUNT_EMAIL=$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com

# Enable Workload Identity on the GKE cluster
gcloud container clusters update <cluster-name> \
    --workload-pool=$PROJECT_ID.svc.id.goog

# Create Google Cloud IAM service account
if ! gcloud iam service-accounts list --filter="email:$SERVICE_ACCOUNT_EMAIL" --format="value(email)"; then
    gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME \
        --display-name "Service Account for Artifact Registry"
else
    echo "Service account $SERVICE_ACCOUNT_NAME already exists."
fi

# Grant permissions to the IAM service account
if ! gcloud projects get-iam-policy $PROJECT_ID --flatten="bindings[].members" --format="table(bindings.members)" | grep $SERVICE_ACCOUNT_EMAIL; then
    gcloud iam service-accounts add-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
        --role "roles/artifactregistry.reader" \
        --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]"
else
    echo "Service account $SERVICE_ACCOUNT_EMAIL already has the role roles/artifactregistry.reader."
fi

# Create Kubernetes service account
kubectl create serviceaccount $KSA_NAME --namespace $NAMESPACE || echo "Kubernetes service account $KSA_NAME already exists."

# Bind the Kubernetes service account to the Google IAM service account
gcloud iam service-accounts add-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
    --role "roles/iam.workloadIdentityUser" \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]"

# Annotate the Kubernetes service account
kubectl annotate serviceaccount $KSA_NAME \
    --namespace $NAMESPACE \
    iam.gke.io/gcp-service-account=$SERVICE_ACCOUNT_EMAIL --overwrite
```

### `cleanup_registry.sh`

```bash
#!/bin/bash

set -e

PROJECT_ID=your-project-id
SERVICE_ACCOUNT_NAME=artifact-registry-sa
SERVICE_ACCOUNT_EMAIL=$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com
NAMESPACE=linkerd-poc
KSA_NAME=artifact-registry-ksa

# Revoke the IAM policy binding
gcloud iam service-accounts remove-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]" \
    --role "roles/artifactregistry.reader"

gcloud iam service-accounts remove-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]" \
    --role "roles/iam.workloadIdentityUser"

# Delete the IAM service account
gcloud iam service-accounts delete $SERVICE_ACCOUNT_EMAIL --quiet

# Delete the Kubernetes service account
kubectl delete serviceaccount $KSA_NAME --namespace $NAMESPACE
```

### Updated `Makefile`

```makefile
DOCKER_REG=your_dockerhub_username
IMAGE_NAME=helloworld
IMAGE_TAG=1.0
NAMESPACE=linkerd-poc
DEPLOYMENTS=helloworld1 helloworld2
REGISTRY_SECRET=regcred

all: build push deploy-all

build:
	docker build -t $(DOCKER_REG)/$(IMAGE_NAME):$(IMAGE_TAG) .

push:
	docker push $(DOCKER_REG)/$(IMAGE_NAME):$(IMAGE_TAG)

deploy-all: namespace registry $(DEPLOYMENTS)

namespace:
	kubectl apply -f kubernetes/namespace.yaml

registry: check-namespace
	./setup_workload_identity.sh

docker-registry-secret: registry

helloworld1: docker-registry-secret
	./apply_deployments.sh

helloworld2: docker-registry-secret
	./apply_deployments.sh

deploy: registry check-namespace
	./apply_deployments.sh

rollout:
	kubectl rollout restart deployment/$(deployment) -n $(NAMESPACE)

inspect:
	kubectl get pods -n $(NAMESPACE) -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.linkerd\\.io/proxy-injector\\.linkerd\\.io/status}{"\n"}{end}'

clean:
	kubectl delete -f kubernetes/helloworld1.yaml || true \
	&& kubectl delete -f kubernetes/helloworld2.yaml || true \
	&& kubectl delete namespace $(NAMESPACE) || true

clean-gcloud:
	./cleanup_registry.sh

check-namespace:
	@if ! kubectl get namespace $(NAMESPACE) > /dev/null 2>&1; then \
		echo "Namespace $(NAMESPACE) does not exist. Please run 'make namespace' first."; \
		exit 1; \
	fi

# Prevent make from treating command line arguments as file names
%:
	@:
```

### YAML Templates and `apply_deployments.sh`

#### `helloworld1.yaml.template`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld1
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld1
  template:
    metadata:
      labels:
        app: helloworld1
    spec:
      serviceAccountName: ${SERVICE_ACCOUNT_NAME}
      containers:
      - name: helloworld1
        image: ${DOCKER_REG}/${IMAGE_NAME}:${IMAGE_TAG}
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld1
  namespace: ${NAMESPACE}
spec:
  selector:
    app: helloworld1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

#### `helloworld2.yaml.template`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld2
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld2
  template:
    metadata:
      labels:
        app: helloworld2
    spec:
      serviceAccountName: ${SERVICE_ACCOUNT_NAME}
      containers:
      - name: helloworld2
        image: ${DOCKER_REG}/${IMAGE_NAME}:${IMAGE_TAG}
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld2
  namespace: ${NAMESPACE}
spec:
  selector:
    app: helloworld2
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### `apply_deployments.sh`

```bash
#!/bin/bash

set -e

# Load environment variables from .env file
source .env

# Substitute variables in helloworld1.yaml.template and apply
envsubst < kubernetes/helloworld1.yaml.template | kubectl apply -f -

# Substitute variables in helloworld2.yaml.template and apply
envsubst < kubernetes/helloworld2.yaml.template | kubectl apply -f -
```

### `.env` File

Create a `.env` file to hold the environment variables:

```dotenv
DOCKER_REG=your_dockerhub_username
IMAGE_NAME=helloworld
IMAGE_TAG=1.0
NAMESPACE=linkerd-poc
SERVICE_ACCOUNT_NAME=artifact-registry-ksa
```

### Usage Instructions

1. **Create the `.env` file:**

   ```bash
   echo "DOCKER_REG=your_dockerhub_username" > .env
   echo "IMAGE_NAME=helloworld" >> .env
   echo "IMAGE_TAG=1.0" >> .env
   echo "NAMESPACE=linkerd-poc" >> .env
   echo "SERVICE_ACCOUNT_NAME=artifact-registry-ksa" >> .env
   ```

2. **Build and push the Docker image:**

   ```bash
   make build
   make push
   ```

3. **Create the namespace:**

   ```bash
   make namespace
   ```

4. **Set up Workload Identity:**

   ```bash
   make registry
   ```

5. **Deploy all Kubernetes resources:**

   ```bash
   make deploy-all
   ```

6. **Deploy a specific deployment:**

   ```bash
   make deploy helloworld1
   ```

7. **Rollout a specific deployment:**

   ```bash
   make rollout deployment=helloworld1
   ```

8. **Inspect deployments to confirm if Linkerd sidecar proxy was injected:**

   ```bash
   make inspect
   ```

9. **Clean up the Kubernetes resources:**

   ```bash
   make clean
   ```

10. **Clean up the Google Cloud resources:**

    ```bash
    make clean-gcloud
    ```
################
The error message indicates that the service account `artifact-registry-sa` does not exist in the project `platform-cloud-engineering`. This is why the policy binding operation fails. Let's troubleshoot this step-by-step:

1. **Verify the Service Account Creation:**

   Ensure the service account was created correctly in the specified project. You can list the service accounts in your project to verify:

   ```bash
   gcloud iam service-accounts list --project platform-cloud-engineering
   ```

   If the service account `artifact-registry-sa` is not listed, create it:

   ```bash
   gcloud iam service-accounts create artifact-registry-sa \
       --project platform-cloud-engineering \
       --display-name "Service Account for Artifact Registry"
   ```

2. **Correct the IAM Policy Binding Command:**

   Ensure you are using the correct command and service account email format:

   ```bash
   SERVICE_ACCOUNT_EMAIL=artifact-registry-sa@platform-cloud-engineering.iam.gserviceaccount.com

   gcloud iam service-accounts add-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
       --project platform-cloud-engineering \
       --member "serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
       --role "roles/artifactregistry.reader"
   ```

### Updated `setup_workload_identity.sh` Script

Ensure the service account creation and IAM policy binding are correctly handled in the script:

```bash
#!/bin/bash

set -e

PROJECT_ID=platform-cloud-engineering
SERVICE_ACCOUNT_NAME=artifact-registry-sa
NAMESPACE=linkerd-poc
KSA_NAME=artifact-registry-ksa
REGION=us-central
REPO_NAME=cloud-eng-repo
SERVICE_ACCOUNT_EMAIL=$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com

# Enable Workload Identity on the GKE cluster
gcloud container clusters update <cluster-name> \
    --project $PROJECT_ID \
    --workload-pool=$PROJECT_ID.svc.id.goog

# Create Google Cloud IAM service account if it doesn't exist
if ! gcloud iam service-accounts list --filter="email:$SERVICE_ACCOUNT_EMAIL" --project $PROJECT_ID --format="value(email)" | grep $SERVICE_ACCOUNT_EMAIL; then
    gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME \
        --project $PROJECT_ID \
        --display-name "Service Account for Artifact Registry"
else
    echo "Service account $SERVICE_ACCOUNT_NAME already exists."
fi

# Grant permissions to the IAM service account
if ! gcloud projects get-iam-policy $PROJECT_ID --flatten="bindings[].members" --format="table(bindings.members)" | grep $SERVICE_ACCOUNT_EMAIL; then
    gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member "serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
        --role "roles/artifactregistry.reader"
else
    echo "Service account $SERVICE_ACCOUNT_EMAIL already has the role roles/artifactregistry.reader."
fi

# Create Kubernetes service account if it doesn't exist
kubectl create serviceaccount $KSA_NAME --namespace $NAMESPACE || echo "Kubernetes service account $KSA_NAME already exists."

# Bind the Kubernetes service account to the Google IAM service account
gcloud iam service-accounts add-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
    --role "roles/iam.workloadIdentityUser" \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]"

# Annotate the Kubernetes service account
kubectl annotate serviceaccount $KSA_NAME \
    --namespace $NAMESPACE \
    iam.gke.io/gcp-service-account=$SERVICE_ACCOUNT_EMAIL --overwrite
```

### Usage Instructions

1. **Create or verify the service account:**

   ```bash
   gcloud iam service-accounts list --project platform-cloud-engineering
   ```

   If the service account does not exist, create it:

   ```bash
   gcloud iam service-accounts create artifact-registry-sa \
       --project platform-cloud-engineering \
       --display-name "Service Account for Artifact Registry"
   ```

2. **Bind IAM policy:**

   ```bash
   SERVICE_ACCOUNT_EMAIL=artifact-registry-sa@platform-cloud-engineering.iam.gserviceaccount.com

   gcloud iam service-accounts add-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
       --project platform-cloud-engineering \
       --member "serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
       --role "roles/artifactregistry.reader"
   ```

3. **Run the setup script:**

   ```bash
   ./setup_workload_identity.sh
   ```

This should resolve the issue with the service account not being found and allow you to continue with the setup process. If you encounter any further issues, please provide the exact error messages for additional troubleshooting.
