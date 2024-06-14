## POC to Demo strategy for Linkerd Implementation

---
### Directory Structure

```
.
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
    "express": "^4.17.1"
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
