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
