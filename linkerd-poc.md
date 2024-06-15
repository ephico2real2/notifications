## POC to Demo strategy for Linkerd Implementation

---
### Directory Structure

```
tree hello-world-poc

├── Makefile
├── Dockerfile
├── app.js
├── package.json
├── kubernetes
│   ├── namespace.yaml
│   ├── helloworld1.yaml
│   └── helloworld2.yaml

```
### Dockerfile

```dockerfile
FROM node:21

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["npm", "start"]
```

### app.js

```javascript
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.send('Hello World!');
});

app.listen(port, () => {
  console.log(`App running on http://localhost:${port}`);
});
```

### package.json

```json
{
  "name": "helloworld",
  "version": "1.0.0",
  "description": "Hello World Node.js app",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.19.2"
  }
}
```

### Kubernetes Manifests

#### namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: linkerd-poc
  annotations:
    linkerd.io/inject: enabled
```

#### helloworld1.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld1
  namespace: linkerd-poc
spec:
  replicas: 2
  selector:
    matchLabels:
      app: helloworld1
  template:
    metadata:
      labels:
        app: helloworld1
    spec:
      containers:
      - name: helloworld
        image: your_dockerhub_username/helloworld:1.0
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld1
  namespace: linkerd-poc
spec:
  selector:
    app: helloworld1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```

#### helloworld2.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld2
  namespace: linkerd-poc
spec:
  replicas: 2
  selector:
    matchLabels:
      app: helloworld2
  template:
    metadata:
      annotations:
        linkerd.io/inject: disabled
      labels:
        app: helloworld2
    spec:
      containers:
      - name: helloworld
        image: your_dockerhub_username/helloworld:1.0
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld2
  namespace: linkerd-poc
spec:
  selector:
    app: helloworld2
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```

### Updated Makefile

```makefile
DOCKER_USER=your_dockerhub_username
IMAGE_NAME=helloworld
IMAGE_TAG=1.0
NAMESPACE=linkerd-poc
DEPLOYMENTS=helloworld1 helloworld2

all: build push deploy-all

build:
	docker build -t $(DOCKER_USER)/$(IMAGE_NAME):$(IMAGE_TAG) .

push:
	docker push $(DOCKER_USER)/$(IMAGE_NAME):$(IMAGE_TAG)

deploy-all: namespace $(DEPLOYMENTS)

namespace:
	kubectl apply -f kubernetes/namespace.yaml

helloworld1:
	kubectl apply -f kubernetes/helloworld1.yaml

helloworld2:
	kubectl apply -f kubernetes/helloworld2.yaml

deploy:
	$(foreach deployment, $(DEPLOYMENTS), kubectl apply -f kubernetes/$(deployment).yaml;)

rollout:
	kubectl rollout restart deployment/$(deployment) -n $(NAMESPACE)

inspect:
	kubectl get pods -n $(NAMESPACE) -o yaml | grep -A5 "linkerd.io/proxy"

clean:
	kubectl delete -f kubernetes/helloworld1.yaml || true
	kubectl delete -f kubernetes/helloworld2.yaml || true
	kubectl delete namespace $(NAMESPACE) || true
```

### Usage Instructions

1. **Build and push the Docker image:**

   ```bash
   make build
   make push
   ```

2. **Deploy the Kubernetes resources (both deployments):**

   ```bash
   make deploy-all
   ```

3. **Deploy a specific deployment:**

   ```bash
   make deploy deployment=helloworld1
   ```

4. **Rollout a specific deployment (to restart the pods and apply new annotations):**

   ```bash
   make rollout deployment=helloworld1
   ```

5. **Inspect deployments to confirm if Linkerd sidecar proxy was injected:**

   ```bash
   make inspect
   ```

6. **Clean up the resources:**

   ```bash
   make clean
   ```

### Explanation

- **deploy-all:** Deploys both `helloworld1` and `helloworld2`.
- **deploy:** Deploys a specified deployment. You can specify which deployment by using `deployment=<deployment-name>`.
- **rollout:** Restarts a specified deployment to apply changes (such as new annotations). You can specify which deployment by using `deployment=<deployment-name>`.
- **inspect:** Inspects the pods in the namespace to check if the Linkerd sidecar proxy was injected. It looks for the `linkerd.io/proxy` annotation in the pods.
- **clean:** Deletes the deployments and the namespace to clean up the resources.

---





Understood. I will parameterize the Docker server, carefully review the `Makefile`, and verify its validity. Here's a step-by-step approach to ensure everything is correct.

### Makefile

1. **Parameterize the Docker server.**
2. **Ensure all targets and dependencies are correctly set up.**
3. **Validate the correctness of the `Makefile` syntax and logic.**

### Updated Makefile

```makefile
DOCKER_REG=your_dockerhub_username
IMAGE_NAME=helloworld
IMAGE_TAG=1.0
NAMESPACE=linkerd-poc
DEPLOYMENTS=helloworld1 helloworld2
REGISTRY_SECRET=gcp-artifactor-registry
DOCKER_SERVER=us-central1-docker.pkg.dev
DOCKER_EMAIL=myname.name@company.com

all: build push deploy-all

build:
	docker build -t $(DOCKER_REG)/$(IMAGE_NAME):$(IMAGE_TAG) .

push:
	docker push $(DOCKER_REG)/$(IMAGE_NAME):$(IMAGE_TAG)

deploy-all: namespace docker-registry-secret $(DEPLOYMENTS)

namespace:
	kubectl apply -f kubernetes/namespace.yaml

docker-registry-secret: namespace
	gcloud auth configure-docker $(DOCKER_SERVER) \
	&& if kubectl get secret $(REGISTRY_SECRET) -n $(NAMESPACE); then \
		kubectl delete secret $(REGISTRY_SECRET) -n $(NAMESPACE); \
	fi \
	&& kubectl -n $(NAMESPACE) create secret docker-registry $(REGISTRY_SECRET) \
	--docker-server=$(DOCKER_SERVER) \
	--docker-username=_json_key \
	--docker-email=$(DOCKER_EMAIL) \
	--docker-password="$$(cat ~/.docker/config.json)"

helloworld1: docker-registry-secret
	kubectl apply -f kubernetes/helloworld1.yaml

helloworld2: docker-registry-secret
	kubectl apply -f kubernetes/helloworld2.yaml

deploy: docker-registry-secret
	@$(foreach deployment, $(filter-out $@,$(MAKECMDGOALS)), \
		kubectl apply -f kubernetes/$(deployment).yaml;)

rollout:
	kubectl rollout restart deployment/$(deployment) -n $(NAMESPACE)

inspect:
	kubectl get pods -n $(NAMESPACE) -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.linkerd\\.io/proxy-injector\\.linkerd\\.io/status}{"\n"}{end}'

clean:
	kubectl delete -f kubernetes/helloworld1.yaml || true \
	&& kubectl delete -f kubernetes/helloworld2.yaml || true \
	&& kubectl delete namespace $(NAMESPACE) || true

# Prevent make from treating command line arguments as file names
%:
	@:
```

### Explanation

1. **Parameterization:**
   - **DOCKER_SERVER:** Parameterized the Docker server URL.
   - **DOCKER_EMAIL:** Parameterized the Docker email.
2. **Targets:**
   - **docker-registry-secret:** Ensures the namespace exists before creating the secret. It uses `gcloud` to configure Docker and handles existing secrets by deleting them first if they exist.
   - **deploy:** Uses `$(filter-out $@,$(MAKECMDGOALS))` to get the command line arguments for specific deployment names.
   - **%:** Ensures Make doesn’t treat command line arguments as file names.

### Usage Instructions

1. **Build and push the Docker image:**

   ```bash
   make build
   make push
   ```

2. **Deploy all Kubernetes resources:**

   ```bash
   make deploy-all
   ```

3. **Deploy a specific deployment:**

   ```bash
   make deploy helloworld1
   ```

4. **Rollout a specific deployment:**

   ```bash
   make rollout deployment=helloworld1
   ```

5. **Inspect deployments to confirm if Linkerd sidecar proxy was injected:**

   ```bash
   make inspect
   ```

6. **Create Docker registry secret:**

   ```bash
   make docker-registry-secret
   ```

7. **Clean up the resources:**

   ```bash
   make clean
   ```

### Validation

1. **Syntax Check:**
   - Ensure all `Makefile` syntax is correct.
   - Use `make -n <target>` to dry-run the targets and check for errors.

2. **Functional Check:**
   - Verify that each command works as expected by running it in the correct order.
   - Ensure that parameterized variables (`DOCKER_REG`, `DOCKER_SERVER`, `DOCKER_EMAIL`, etc.) are correctly referenced in the commands.



Sure, I'll adjust the `Makefile` to ensure that the namespace exists before running any deployment-related targets (`helloworld1`, `helloworld2`, and `deploy`). Here's the updated `Makefile`:

### Updated Makefile

```makefile
DOCKER_REG=your_dockerhub_username
IMAGE_NAME=helloworld
IMAGE_TAG=1.0
NAMESPACE=linkerd-poc
DEPLOYMENTS=helloworld1 helloworld2
REGISTRY_SECRET=gcp-artifactor-registry
DOCKER_SERVER=us-central1-docker.pkg.dev
DOCKER_EMAIL=xxxx

all: build push deploy-all

build:
	docker build -t $(DOCKER_REG)/$(IMAGE_NAME):$(IMAGE_TAG) .

push:
	docker push $(DOCKER_REG)/$(IMAGE_NAME):$(IMAGE_TAG)

deploy-all: namespace registry $(DEPLOYMENTS)

namespace:
	kubectl apply -f kubernetes/namespace.yaml

registry: check-namespace
	$(MAKE) docker-registry-secret

docker-registry-secret:
	gcloud auth configure-docker $(DOCKER_SERVER) \
	&& if kubectl get secret $(REGISTRY_SECRET) -n $(NAMESPACE) > /dev/null 2>&1; then \
		kubectl delete secret $(REGISTRY_SECRET) -n $(NAMESPACE); \
	fi \
	&& kubectl -n $(NAMESPACE) create secret docker-registry $(REGISTRY_SECRET) \
	--docker-server=$(DOCKER_SERVER) \
	--docker-username=_json_key \
	--docker-email=$(DOCKER_EMAIL) \
	--docker-password="$$(cat ~/.docker/config.json)"

helloworld1: registry check-namespace
	kubectl apply -f kubernetes/helloworld1.yaml

helloworld2: registry check-namespace
	kubectl apply -f kubernetes/helloworld2.yaml

deploy: registry check-namespace
	@$(foreach deployment, $(filter-out $@,$(MAKECMDGOALS)), \
		kubectl apply -f kubernetes/$(deployment).yaml;)

rollout:
	kubectl rollout restart deployment/$(deployment) -n $(NAMESPACE)

inspect:
	kubectl get pods -n $(NAMESPACE) -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.linkerd\\.io/proxy-injector\\.linkerd\\.io/status}{"\n"}{end}'

clean:
	kubectl delete -f kubernetes/helloworld1.yaml || true \
	&& kubectl delete -f kubernetes/helloworld2.yaml || true \
	&& kubectl delete namespace $(NAMESPACE) || true

check-namespace:
	@if ! kubectl get namespace $(NAMESPACE) > /dev/null 2>&1; then \
		echo "Namespace $(NAMESPACE) does not exist. Please run 'make namespace' first."; \
		exit 1; \
	fi

# Prevent make from treating command line arguments as file names
%:
	@:
```

### Explanation

- **check-namespace:** A new target that checks if the namespace exists. If not, it prompts the user to run `make namespace` and exits with an error.
- **registry:** Depends on `check-namespace` to ensure the namespace exists before creating the Docker registry secret.
- **helloworld1, helloworld2, deploy:** All depend on `registry` and `check-namespace` to ensure the namespace exists and the registry secret is created before applying the deployment manifests.

### Usage Instructions

1. **Build and push the Docker image:**

   ```bash
   make build
   make push
   ```

2. **Create the namespace:**

   ```bash
   make namespace
   ```

3. **Create the Docker registry secret:**

   ```bash
   make registry
   ```

4. **Deploy all Kubernetes resources:**

   ```bash
   make deploy-all
   ```

5. **Deploy a specific deployment:**

   ```bash
   make deploy helloworld1
   ```

6. **Rollout a specific deployment:**

   ```bash
   make rollout deployment=helloworld1
   ```

7. **Inspect deployments to confirm if Linkerd sidecar proxy was injected:**

   ```bash
   make inspect
   ```

8. **Clean up the resources:**

   ```bash
   make clean
   ```

################################################
Thank you for the clarification. I'll update the `DOCKER_SERVER` to include the project ID and repository name, and preserve the region. Additionally, I'll ensure the `setup_registry.sh` script and `Makefile` correctly reflect these changes.

### Step-by-Step Guide

1. **Update the Shell Script for Registry Setup:**

   Create a file named `setup_registry.sh` with the following content:

   ```bash
   #!/bin/bash

   set -e

   PROJECT_ID=your-project-id
   REPO_NAME=your-repo-name
   REGION=us-central
   SERVICE_ACCOUNT_NAME=artifact-registry-sa
   NAMESPACE=linkerd-poc
   REGISTRY_SECRET=artifact-registry-secret
   DOCKER_SERVER=$REGION-docker.pkg.dev/$PROJECT_ID/$REPO_NAME
   SERVICE_ACCOUNT_EMAIL=$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com

   # Create service account
   gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME \
       --display-name "Service Account for Artifact Registry"

   # Grant permissions
   gcloud projects add-iam-policy-binding $PROJECT_ID \
       --member serviceAccount:$SERVICE_ACCOUNT_EMAIL \
       --role roles/artifactregistry.reader

   # Create and download the key
   gcloud iam service-accounts keys create ~/key.json \
       --iam-account $SERVICE_ACCOUNT_EMAIL

   # Create Kubernetes secret
   kubectl create secret docker-registry $REGISTRY_SECRET \
       --namespace $NAMESPACE \
       --docker-server=$DOCKER_SERVER \
       --docker-username=_json_key \
       --docker-password="$(cat ~/key.json)" \
       --docker-email=$SERVICE_ACCOUNT_EMAIL

   # Annotate and patch the default service account
   kubectl annotate serviceaccount default \
       --namespace $NAMESPACE \
       "kubernetes.io/service-account.name=$SERVICE_ACCOUNT_NAME" --overwrite

   kubectl patch serviceaccount default \
       --namespace $NAMESPACE \
       -p "{\"imagePullSecrets\": [{\"name\": \"$REGISTRY_SECRET\"}]}"
   ```

2. **Make the Shell Script Executable:**

   ```bash
   chmod +x setup_registry.sh
   ```

3. **Update the `Makefile` to Call the Shell Script:**

   Here’s the updated `Makefile`:

   ### Updated Makefile

   ```makefile
   DOCKER_REG=your_dockerhub_username
   IMAGE_NAME=helloworld
   IMAGE_TAG=1.0
   NAMESPACE=linkerd-poc
   DEPLOYMENTS=helloworld1 helloworld2
   REGISTRY_SECRET=artifact-registry-secret
   REGION=us-central
   PROJECT_ID=your-project-id
   REPO_NAME=your-repo-name
   SERVICE_ACCOUNT_NAME=artifact-registry-sa
   DOCKER_SERVER=$(REGION)-docker.pkg.dev/$(PROJECT_ID)/$(REPO_NAME)

   all: build push deploy-all

   build:
   	docker build -t $(DOCKER_REG)/$(IMAGE_NAME):$(IMAGE_TAG) .

   push:
   	docker push $(DOCKER_REG)/$(IMAGE_NAME):$(IMAGE_TAG)

   deploy-all: namespace registry $(DEPLOYMENTS)

   namespace:
   	kubectl apply -f kubernetes/namespace.yaml

   registry: check-namespace
   	./setup_registry.sh

   docker-registry-secret: registry

   helloworld1: docker-registry-secret
   	kubectl apply -f kubernetes/helloworld1.yaml

   helloworld2: docker-registry-secret
   	kubectl apply -f kubernetes/helloworld2.yaml

   deploy: registry check-namespace
   	@$(foreach deployment, $(filter-out $@,$(MAKECMDGOALS)), \
   		kubectl apply -f kubernetes/$(deployment).yaml;)

   rollout:
   	kubectl rollout restart deployment/$(deployment) -n $(NAMESPACE)

   inspect:
   	kubectl get pods -n $(NAMESPACE) -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.linkerd\\.io/proxy-injector\\.linkerd\\.io/status}{"\n"}{end}'

   clean:
   	kubectl delete -f kubernetes/helloworld1.yaml || true \
   	&& kubectl delete -f kubernetes/helloworld2.yaml || true \
   	&& kubectl delete namespace $(NAMESPACE) || true

   check-namespace:
   	@if ! kubectl get namespace $(NAMESPACE) > /dev/null 2>&1; then \
   		echo "Namespace $(NAMESPACE) does not exist. Please run 'make namespace' first."; \
   		exit 1; \
   	fi

   # Prevent make from treating command line arguments as file names
   %:
   	@:
   ```

### Explanation

1. **Shell Script (`setup_registry.sh`):**
   - Uses the `PROJECT_ID`, `REPO_NAME`, and `REGION` variables to construct the `DOCKER_SERVER` URL.
   - Handles the creation of the service account, granting permissions, creating and downloading the key, and setting up the Kubernetes secret and service account.

2. **Updated `Makefile`:**
   - The `DOCKER_SERVER` variable now correctly includes the `PROJECT_ID` and `REPO_NAME` along with the `REGION`.
   - The `registry` target calls the `setup_registry.sh` script.
   - The `docker-registry-secret` target depends on the `registry` target to ensure the setup is complete before proceeding.

### Usage Instructions

1. **Build and push the Docker image:**

   ```bash
   make build
   make push
   ```

2. **Create the namespace:**

   ```bash
   make namespace
   ```

3. **Create the Docker registry secret:**

   ```bash
   make registry
   ```

4. **Deploy all Kubernetes resources:**

   ```bash
   make deploy-all
   ```

5. **Deploy a specific deployment:**

   ```bash
   make deploy helloworld1
   ```

6. **Rollout a specific deployment:**

   ```bash
   make rollout deployment=helloworld1
   ```

7. **Inspect deployments to confirm if Linkerd sidecar proxy was injected:**

   ```bash
   make inspect
   ```

8. **Clean up the resources:**

   ```bash
   make clean
   ```

This setup ensures that the `DOCKER_SERVER` includes the necessary `PROJECT_ID`, `REPO_NAME`, and `REGION`, and the registry-related tasks are decoupled from the `Makefile` and handled by the `setup_registry.sh` script. This makes the `Makefile` cleaner and more maintainable.




############################


```
#!/bin/bash

set -e

PROJECT_ID=your-project-id
REPO_NAME=your-repo-name
REGION=us-central
SERVICE_ACCOUNT_NAME=artifact-registry-sa
NAMESPACE=linkerd-poc
REGISTRY_SECRET=artifact-registry-secret
DOCKER_SERVER=$REGION-docker.pkg.dev/$PROJECT_ID/$REPO_NAME
SERVICE_ACCOUNT_EMAIL=$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com

# Authenticate with Google Cloud
gcloud auth login
gcloud config set project $PROJECT_ID

# Create service account if it does not exist
if ! gcloud iam service-accounts list --filter="email:$SERVICE_ACCOUNT_EMAIL" --format="value(email)"; then
    gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME \
        --display-name "Service Account for Artifact Registry"
else
    echo "Service account $SERVICE_ACCOUNT_NAME already exists."
fi

# Grant permissions if not already granted
if ! gcloud projects get-iam-policy $PROJECT_ID --flatten="bindings[].members" --format="table(bindings.members)" | grep $SERVICE_ACCOUNT_EMAIL; then
    gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member serviceAccount:$SERVICE_ACCOUNT_EMAIL \
        --role roles/artifactregistry.reader
else
    echo "Service account $SERVICE_ACCOUNT_EMAIL already has the role roles/artifactregistry.reader."
fi

# Create and download the key
gcloud iam service-accounts keys create ~/key.json \
    --iam-account $SERVICE_ACCOUNT_EMAIL

# Create Kubernetes secret
if kubectl get secret $REGISTRY_SECRET -n $NAMESPACE > /dev/null 2>&1; then
    kubectl delete secret $REGISTRY_SECRET -n $NAMESPACE
fi

kubectl create secret docker-registry $REGISTRY_SECRET \
    --namespace $NAMESPACE \
    --docker-server=$DOCKER_SERVER \
    --docker-username=_json_key \
    --docker-password="$(cat ~/key.json)" \
    --docker-email=$SERVICE_ACCOUNT_EMAIL

# Annotate and patch the default service account
kubectl annotate serviceaccount default \
    --namespace $NAMESPACE \
    "kubernetes.io/service-account.name=$SERVICE_ACCOUNT_NAME" --overwrite

kubectl patch serviceaccount default \
    --namespace $NAMESPACE \
    -p "{\"imagePullSecrets\": [{\"name\": \"$REGISTRY_SECRET\"}]}"
```


##################

Here are the two scripts and the `Makefile` with all the necessary details:

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
    gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member "serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
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
gcloud projects remove-iam-policy-binding $PROJECT_ID \
    --member "serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
    --role "roles/artifactregistry.reader"

gcloud iam service-accounts remove-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]" \
    --role "roles/iam.workloadIdentityUser"

# Delete the IAM service account
gcloud iam service-accounts delete $SERVICE_ACCOUNT_EMAIL --quiet

# Delete the Kubernetes service account
kubectl delete serviceaccount $KSA_NAME --namespace $NAMESPACE
```

### `Makefile`

```makefile
DOCKER_REG=your_dockerhub_username
IMAGE_NAME=helloworld
IMAGE_TAG=1.0
NAMESPACE=linkerd-poc
DEPLOYMENTS=helloworld1 helloworld2
REGION=us-central
PROJECT_ID=your-project-id
REPO_NAME=your-repo-name
SERVICE_ACCOUNT_NAME=artifact-registry-sa
KSA_NAME=artifact-registry-ksa
DOCKER_SERVER=$(REGION)-docker.pkg.dev/$(PROJECT_ID)/$(REPO_NAME)

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
	kubectl apply -f kubernetes/helloworld1.yaml

helloworld2: docker-registry-secret
	kubectl apply -f kubernetes/helloworld2.yaml

deploy: registry check-namespace
	@$(foreach deployment, $(filter-out $@,$(MAKECMDGOALS)), \
		kubectl apply -f kubernetes/$(deployment).yaml;)

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

### Usage Instructions

1. **Authenticate with Google Cloud:**

   ```bash
   gcloud auth login
   gcloud config set project your-project-id
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

This setup includes the setup and cleanup scripts, as well as the `Makefile` to automate the deployment and cleanup of the necessary resources.
