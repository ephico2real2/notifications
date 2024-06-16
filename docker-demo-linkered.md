Docker
---

### `cleanup_dockerhub.sh`

```bash
#!/bin/bash

set -e

NAMESPACE=linkerd-poc
REGISTRY_SECRET=regcred
SERVICE_ACCOUNT_NAME=artifact-registry-ksa

# Delete the Kubernetes secret for Docker Hub
if kubectl get secret $REGISTRY_SECRET -n $NAMESPACE > /dev/null 2>&1; then
    kubectl delete secret $REGISTRY_SECRET -n $NAMESPACE
fi

# Remove the imagePullSecrets annotation from the specified service account
kubectl patch serviceaccount $SERVICE_ACCOUNT_NAME \
    --namespace $NAMESPACE \
    -p "{\"imagePullSecrets\": null}"
```

### Updated `Makefile`

Here is the updated `Makefile` including references to `cleanup_dockerhub.sh`:

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
	kubectl apply -f kubernetes/linkerd-poc-namespace.yaml

registry: check-namespace
	./setup_dockerhub.sh

docker-registry-secret: registry

helloworld1: docker-registry-secret
	./apply_deployments.sh

helloworld2: docker-registry-secret
	./apply_deployments.sh

deploy: registry check-namespace
	./apply_deployments.sh


rollout:
	@echo "Available deployments in namespace $(NAMESPACE):"
	@kubectl get deployments -n $(NAMESPACE) -o custom-columns=NAME:.metadata.name
	@read -p "Enter deployment name to rollout: " deployment; \
	kubectl rollout restart deployment/$$deployment -n $(NAMESPACE); \
	echo "Waiting for pods to be ready..."; \
	until kubectl get pods -n $(NAMESPACE) -l app=$$deployment -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}' | grep -q "True"; do \
	  echo -n "."; \
	  sleep 1; \
	done; \
	echo "Pods in deployment $$deployment after rollout:"; \
	kubectl get pods -n $(NAMESPACE) -l app=$$deployment -o wide

rolloutAll:
	@echo "Rolling out all deployments in namespace $(NAMESPACE):"
	@for deployment in $(shell kubectl get deployments -n $(NAMESPACE) -o jsonpath='{.items[*].metadata.name}'); do \
	  echo "Rolling out $$deployment..."; \
	  kubectl rollout restart deployment/$$deployment -n $(NAMESPACE); \
	  echo "Waiting for pods to be ready for $$deployment..."; \
	  until kubectl get pods -n $(NAMESPACE) -l app=$$deployment -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}' | grep -q "True"; do \
	    echo -n "."; \
	    sleep 1; \
	  done; \
	  echo "Pods in deployment $$deployment after rollout:"; \
	  kubectl get pods -n $(NAMESPACE) -l app=$$deployment -o wide; \
	done

inspect:
	kubectl get pods -n $(NAMESPACE) -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.linkerd\.io/proxy-injector\.linkerd\.io/status}{"\n"}{end}'

clean:
	kubectl delete -f kubernetes/helloworld1.yaml || true \
	&& kubectl delete -f kubernetes/helloworld2.yaml || true \
	&& kubectl delete namespace $(NAMESPACE) || true

clean-dockerhub:
	./cleanup_dockerhub.sh

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

3. **Create the namespace with Linkerd enabled:**

   ```bash
   make namespace
   ```

4. **Set up Docker Hub registry secret (you will be prompted for your Docker password):**

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

10. **Clean up the Docker Hub resources:**

    ```bash
    make clean-dockerhub
    ```

### Summary of Key Files

1. **`linkerd-poc-namespace.yaml`**

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: linkerd-poc
      annotations:
        linkerd.io/inject: enabled
    ```

2. **`helloworld1.yaml.template`**

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

3. **`helloworld2.yaml.template`**

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
          annotations:
            linkerd.io/inject: disabled
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

4. **`setup_dockerhub.sh`**

```bash
#!/bin/bash

set -e

NAMESPACE=linkerd-poc
REGISTRY_SECRET=regcred
DOCKER_USERNAME=your_dockerhub_username
DOCKER_EMAIL=your_dockerhub_email
SERVICE_ACCOUNT_NAME=artifact-registry-ksa

# Prompt for Docker password
read -sp 'Enter Docker Hub password: ' DOCKER_PASSWORD
echo

# Create Kubernetes secret for Docker Hub
if kubectl get secret $REGISTRY_SECRET -n $NAMESPACE > /dev/null 2>&1; then
    kubectl delete secret $REGISTRY_SECRET -n $NAMESPACE
fi

kubectl create secret docker-registry $REGISTRY_SECRET \
    --namespace $NAMESPACE \
    --docker-username=$DOCKER_USERNAME \
    --docker-password=$DOCKER_PASSWORD \
    --docker-email=$DOCKER_EMAIL

# Create the service account if it doesn't exist
if ! kubectl get serviceaccount $SERVICE_ACCOUNT_NAME -n $NAMESPACE > /dev/null 2>&1; then
    kubectl create serviceaccount $SERVICE_ACCOUNT_NAME --namespace $NAMESPACE
fi

# Annotate and patch the specified service account
kubectl annotate serviceaccount $SERVICE_ACCOUNT_NAME \
    --namespace $NAMESPACE \
    "kubernetes.io/service-account.name=$SERVICE_ACCOUNT_NAME" --overwrite

kubectl patch serviceaccount $SERVICE_ACCOUNT_NAME \
    --namespace $NAMESPACE \
    -p "{\"imagePullSecrets\": [{\"name\": \"$REGISTRY_SECRET\"}]}"

```

5. **`cleanup_dockerhub.sh`**

```bash
    #!/bin/bash

    set -e

    NAMESPACE=linkerd-poc
    REGISTRY_SECRET=regcred
    SERVICE_ACCOUNT_NAME=artifact-registry-ksa

    # Delete the Kubernetes secret for Docker Hub
    if kubectl get secret $REGISTRY_SECRET -n $NAMESPACE > /dev/null 2>&1; then
        kubectl delete secret $REGISTRY_SECRET -n $NAMESPACE
    fi

    # Remove the imagePullSecrets annotation from the specified service account
    kubectl patch serviceaccount $SERVICE_ACCOUNT_NAME \
        --namespace $NAMESPACE \
        -p "{\"imagePullSecrets\": null}"
```

6. **`apply_deployments.sh`**


```bash

#!/bin/bash

set -e

# Function to check if envsubst is installed
check_envsubst() {
  if ! command -v envsubst &> /dev/null; then
    echo "envsubst could not be found. Please install it using:"
    echo "  brew install gettext"
    exit 1
  fi
}

# Check if envsubst is installed
check_envsubst

# Load environment variables from .env file
if [ -f .env ]; then
  export $(grep -v '^#' .env | xargs)
else
  echo ".env file not found!"
  exit 1
fi

# Substitute variables in helloworld1.yaml.template and apply
envsubst < kubernetes/helloworld1.yaml.template | kubectl apply -f -

# Substitute variables in helloworld2.yaml.template and apply
envsubst < kubernetes/helloworld2.yaml.template | kubectl apply -f -

```


############################
```
rollout:
	@echo "Available deployments in namespace $(NAMESPACE):"
	@kubectl get deployments -n $(NAMESPACE) -o custom-columns=NAME:.metadata.name
	@read -p "Enter deployment name to rollout: " deployment; \
	kubectl rollout restart deployment/$$deployment -n $(NAMESPACE); \
	echo "Waiting for pods to be ready..."; \
	until [ $$(kubectl get pods -n $(NAMESPACE) -l app=$$deployment -o 'jsonpath={..status.conditions[?(@.type=="Ready")].status}' | grep -c True) -eq $$(kubectl get pods -n $(NAMESPACE) -l app=$$deployment --field-selector=status.phase=Running --no-headers | wc -l) ]; do \
	  echo -n "."; \
	  sleep 1; \
	done; \
	echo "Pods in deployment $$deployment after rollout:"; \
	kubectl get pods -n $(NAMESPACE) -l app=$$deployment -o wide

rolloutAll:
	@echo "Rolling out all deployments in namespace $(NAMESPACE):"
	@for deployment in $(shell kubectl get deployments -n $(NAMESPACE) -o jsonpath='{.items[*].metadata.name}'); do \
	  echo "Rolling out $$deployment..."; \
	  kubectl rollout restart deployment/$$deployment -n $(NAMESPACE); \
	  echo "Waiting for pods to be ready for $$deployment..."; \
	  until [ $$(kubectl get pods -n $(NAMESPACE) -l app=$$deployment -o 'jsonpath={..status.conditions[?(@.type=="Ready")].status}' | grep -c True) -eq $$(kubectl get pods -n $(NAMESPACE) -l app=$$deployment --field-selector=status.phase=Running --no-headers | wc -l) ]; do \
	    echo -n "."; \
	    sleep 1; \
	  done; \
	  echo "Pods in deployment $$deployment after rollout:"; \
	  kubectl get pods -n $(NAMESPACE) -l app=$$deployment -o wide; \
	done
```
---






