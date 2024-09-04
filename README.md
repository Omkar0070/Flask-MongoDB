# Flask-MongoDB
Flask-MongoDB Application on Kubernetes


```markdown
# Flask-MongoDB Application on Kubernetes

## Project Overview

This project demonstrates the deployment of a Python Flask application connected to MongoDB on a Kubernetes cluster. The application features autoscaling, persistent storage, and secured database access. It includes a Dockerized Flask application and a MongoDB StatefulSet, both managed through Kubernetes.

## Prerequisites

- [Docker](https://www.docker.com/) - For containerizing the Flask application
- [Minikube](https://minikube.sigs.k8s.io/docs/) - For running a local Kubernetes cluster
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) - Kubernetes command-line tool
- [Python 3.x](https://www.python.org/downloads/) - For local development
- [Git](https://git-scm.com/) - For version control

### Local Setup

### 1. Clone the Repository

```bash
git clone https://github.com/Omkar0070/Flask-MongoDB.git
cd flask-mongodb-app
```

### 2. Set Up the Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate  # On Windows use: venv\Scripts\activate
```

### 3. Install Python Dependencies

```bash
pip install -r requirements.txt
```

### 4. Set Up MongoDB Using Docker

```bash
docker pull mongo:latest
docker run -d -p 27017:27017 --name mongodb mongo:latest
```

### 5. Set Up Environment Variables

Create a `.env` file in the project directory with the following content:

```plaintext
MONGODB_URI=mongodb://localhost:27017/
```

Load the environment variables:

```bash
export $(cat .env | xargs)
```

### 6. Run the Flask Application

```bash
export FLASK_APP=app.py
export FLASK_ENV=development
flask run
```

### 7. Access the Application

Open your web browser and navigate to `http://localhost:5000` to see the welcome message.

## Kubernetes Deployment

### 1. Dockerize the Flask Application

Create a `Dockerfile` in your project directory:

```Dockerfile
# Use an official Python runtime as a parent image
FROM python:3.8-slim

# Set the working directory in the container
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Make port 5000 available to the world outside this container
EXPOSE 5000

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

Build and push the Docker image:

```bash
docker build -t flask-mongodb-app .
docker tag flask-mongodb-app your-dockerhub-username/flask-mongodb-app
docker push your-dockerhub-username/flask-mongodb-app
```

### 2. Create Kubernetes YAML Files

**Deployment for Flask Application (`flask-deployment.yaml`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
      - name: flask-app
        image: your-dockerhub-username/flask-mongodb-app:latest
        ports:
        - containerPort: 5000
        env:
        - name: MONGODB_URI
          value: "mongodb://mongo:27017/"
        resources:
          requests:
            memory: "250Mi"
            cpu: "200m"
          limits:
            memory: "500Mi"
            cpu: "500m"
```

**StatefulSet for MongoDB with Persistent Volume (`mongo-statefulset.yaml`)**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo
        image: mongo:latest
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-storage
          mountPath: /data/db
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: "password"
  volumeClaimTemplates:
  - metadata:
      name: mongo-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

**Services for Flask and MongoDB (`services.yaml`)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
  type: LoadBalancer
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  selector:
    app: mongo
  ports:
  - protocol: TCP
    port: 27017
    targetPort: 27017
  clusterIP: None # Headless service for StatefulSet
```

**Horizontal Pod Autoscaler (HPA) for Flask Application (`hpa.yaml`)**

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: flask-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flask-deployment
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

### 3. Deploy to Kubernetes

1. **Start Minikube:**

   ```bash
   minikube start
   ```

2. **Apply the YAML files to create resources:**

   ```bash
   kubectl apply -f flask-deployment.yaml
   kubectl apply -f mongo-statefulset.yaml
   kubectl apply -f services.yaml
   kubectl apply -f hpa.yaml
   ```

3. **Verify that the pods are running:**

   ```bash
   kubectl get pods
   ```

4. **Get the Minikube IP to access the Flask application:**

   ```bash
   minikube service flask-service --url
   ```

## Testing and Verification

1. **Access the Flask application through the Minikube IP.**
   - Use `curl` or a browser to verify that the application is functioning.

2. **Simulate high traffic to test autoscaling:**
   - Use a tool like `hey` or Apache Benchmark (`ab`) to simulate traffic and trigger the Horizontal Pod Autoscaler (HPA).

3. **Verify autoscaling:**

   ```bash
   kubectl get hpa
   ```

## Design Choices

- **Dockerization:** Containerized the Flask application to ensure consistency across different environments and simplify deployment.
- **StatefulSet for MongoDB:** Used StatefulSet to manage MongoDB's stateful nature and ensure data persistence.
- **Horizontal Pod Autoscaler (HPA):** Configured HPA to automatically scale the number of Flask application replicas based on CPU utilization.

## DNS Resolution

In Kubernetes, DNS resolution within the cluster is handled by the CoreDNS service. Services are assigned DNS names, and other services and pods can resolve these names to access them. For example, the Flask application can access MongoDB using the service name `mongo` due to the DNS resolution provided by Kubernetes.

## Resource Requests and Limits

- **Requests:** Specify the minimum amount of CPU and memory resources required by a container. Helps ensure that containers are scheduled on nodes that have sufficient resources.
- **Limits:** Specify the maximum amount of CPU and memory a container can use. Helps prevent any single container from consuming excessive resources and affecting other containers.
