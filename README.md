Absolutely. Here is a **GitHub-ready detailed `README.md`** for your Helm repository. It documents not just *what* you deployed, but also the **WHY behind the Kubernetes and Helm design**, which makes it useful as a DevOps portfolio project.

````markdown
# Student MERN Application on Kubernetes using Helm

A production-style MERN student management application deployed on Kubernetes using Docker, Helm, kOps, and AWS.

This project demonstrates how a containerized MERN application can be deployed on a Kubernetes cluster with:

- React frontend
- Node.js / Express backend
- MongoDB ReplicaSet
- Kubernetes StatefulSet
- Persistent Volumes
- Kubernetes Services
- Nginx reverse proxy
- AWS LoadBalancer
- Helm packaging and templating
- kOps-managed Kubernetes cluster

---

## Architecture

```text
                         Internet
                            │
                            ▼
                    AWS LoadBalancer
                            │
                            ▼
                    React / Nginx Pods
                       2 Replicas
                            │
                     /api/students
                            │
                            ▼
                    node-service
                       ClusterIP
                            │
                            ▼
                  Node.js / Express Pods
                       2 Replicas
                            │
                            ▼
                  MongoDB Headless Service
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
        mongodb-0      mongodb-1      mongodb-2
         PRIMARY       SECONDARY       SECONDARY
             └──────────────┼──────────────┘
                            │
                       ReplicaSet rs0
                            │
                            ▼
                    Persistent Storage
                         AWS EBS
````

---

# Technology Stack

## Application

* React
* Vite
* Axios
* Node.js
* Express.js
* Mongoose
* MongoDB

## Containerization

* Docker
* Docker Hub

## Kubernetes

* Kubernetes
* Deployments
* StatefulSet
* Services
* ConfigMap
* PersistentVolumeClaim
* PersistentVolume
* MongoDB ReplicaSet

## Helm

* Helm 4
* Helm Chart
* `values.yaml`
* Helm templates
* Helm release management

## Infrastructure

* AWS EC2
* AWS EBS
* AWS LoadBalancer
* kOps

---

# Project Structure

```text
student-mern-helm-kubernetes/
│
├── backend/
│   ├── package.json
│   ├── server.js
│   └── Dockerfile
│
├── frontend/
│   ├── package.json
│   ├── src/
│   │   └── App.jsx
│   ├── nginx.conf
│   ├── vite.config.js
│   └── Dockerfile
│
├── student-app/
│   ├── Chart.yaml
│   ├── values.yaml
│   │
│   └── templates/
│       ├── node-config.yaml
│       ├── node-deployment.yml
│       ├── node-service.yaml
│       ├── react-deployment.yaml
│       ├── react-service.yaml
│       ├── sts.yml
│       └── sts-svc.yml
│
├── .gitignore
└── README.md
```

---

# Application Overview

This project is a simple student management application.

The frontend displays student information retrieved from the backend.

Example student data:

```json
[
  {
    "name": "Rahul Sharma",
    "age": 21,
    "course": "Computer Science"
  },
  {
    "name": "Akashad Patel",
    "age": 22,
    "course": "Information Technology"
  },
  {
    "name": "Amit Kumar",
    "age": 20,
    "course": "Electronics"
  }
]
```

The data is stored in:

```text
MongoDB
    ↓
school database
    ↓
students collection
```

---

# Backend

The backend is built using:

* Node.js
* Express
* Mongoose

The backend exposes:

```text
GET /api/students
```

The endpoint retrieves students from MongoDB.

The backend listens on:

```text
Port 5000
```

The MongoDB connection is provided through the Kubernetes ConfigMap.

---

# Frontend

The frontend is built using:

* React
* Vite
* Axios

The React application requests:

```text
/api/students
```

The frontend does not directly connect to MongoDB or the Node.js pod.

Instead, Nginx acts as a reverse proxy.

```text
Browser
   │
   ▼
Nginx
   │
   │ /api/*
   ▼
node-service:5000
```

This allows the browser to communicate with the application using the same public hostname.

---

# Docker Architecture

Both frontend and backend are containerized.

## Backend Image

```text
abhisalunke16/student-backend:v1
```

The backend Dockerfile uses Node.js and exposes port `5000`.

```dockerfile
FROM node:20

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 5000

CMD ["node", "server.js"]
```

---

# Frontend Docker Image

```text
abhisalunke16/student-frontend:v1
```

The frontend uses a multi-stage Docker build.

First stage:

```text
Node.js
   ↓
npm install
   ↓
npm run build
   ↓
React dist/
```

Second stage:

```text
Nginx
   ↓
React static files
```

This separates the build environment from the runtime environment.

---

# Kubernetes Architecture

The application uses different Kubernetes workload types depending on the responsibility of the component.

| Component | Kubernetes Resource | Replicas |
| --------- | ------------------- | -------: |
| React     | Deployment          |        2 |
| Node.js   | Deployment          |        2 |
| MongoDB   | StatefulSet         |        3 |

---

# Why Deployments for React and Node.js?

React and Node.js application pods are largely stateless.

If a pod fails, Kubernetes can create a replacement pod.

Therefore, a Deployment is appropriate.

For example:

```yaml
replicas: 2
```

provides two application pods.

This improves:

* Availability
* Fault tolerance
* Rolling updates
* Horizontal scaling

---

# Why StatefulSet for MongoDB?

MongoDB is stateful.

Unlike frontend and backend pods, MongoDB requires:

* Stable pod identity
* Persistent storage
* Stable network identity

Therefore, MongoDB uses a StatefulSet.

The pods receive predictable names:

```text
mongodb-0
mongodb-1
mongodb-2
```

Each pod also gets its own PVC:

```text
mongo-data-mongodb-0
mongo-data-mongodb-1
mongo-data-mongodb-2
```

---

# MongoDB ReplicaSet

MongoDB is deployed as a three-member ReplicaSet.

```text
                rs0
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
 mongodb-0   mongodb-1   mongodb-2
  PRIMARY    SECONDARY   SECONDARY
```

The StatefulSet starts MongoDB using:

```text
--bind_ip_all
--replSet rs0
```

The ReplicaSet is initialized using:

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-0.mongodb:27017" },
    { _id: 1, host: "mongodb-1.mongodb:27017" },
    { _id: 2, host: "mongodb-2.mongodb:27017" }
  ]
})
```

After initialization:

```text
mongodb-0 → PRIMARY
mongodb-1 → SECONDARY
mongodb-2 → SECONDARY
```

The secondaries replicate data from the primary.

---

# MongoDB Headless Service

MongoDB uses a Headless Service:

```yaml
clusterIP: None
```

The service is named:

```text
mongodb
```

This allows StatefulSet pods to have stable DNS names.

For example:

```text
mongodb-0.mongodb:27017
mongodb-1.mongodb:27017
mongodb-2.mongodb:27017
```

These stable names are important for MongoDB ReplicaSet configuration.

---

# Persistent Storage

MongoDB uses Kubernetes PersistentVolumeClaims.

The architecture is:

```text
MongoDB Pod
     │
     ▼
PersistentVolumeClaim
     │
     ▼
PersistentVolume
     │
     ▼
AWS EBS
```

Each MongoDB replica receives its own storage.

Example:

```text
mongodb-0
   ↓
mongo-data-mongodb-0
   ↓
2Gi

mongodb-1
   ↓
mongo-data-mongodb-1
   ↓
2Gi

mongodb-2
   ↓
mongo-data-mongodb-2
   ↓
2Gi
```

This prevents MongoDB data from being tied to the temporary lifecycle of a pod.

---

# Kubernetes Services

Three important services are used.

## React Service

```text
react-service
```

Type:

```text
LoadBalancer
```

This exposes the frontend outside the Kubernetes cluster.

AWS provisions a LoadBalancer for this service.

---

## Node Service

```text
node-service
```

Type:

```text
ClusterIP
```

The backend is only accessible inside the Kubernetes cluster.

The React/Nginx pods communicate with:

```text
node-service:5000
```

There is no need to expose the Node.js application directly to the Internet.

---

## MongoDB Service

```text
mongodb
```

Type:

```text
Headless Service
```

It provides stable DNS identities for the MongoDB StatefulSet.

---

# Kubernetes ConfigMap

The Node.js application receives its MongoDB connection string through a ConfigMap.

Example:

```yaml
data:
  MONGO_URI: "mongodb://mongodb-0.mongodb:27017,mongodb-1.mongodb:27017,mongodb-2.mongodb:27017/school?replicaSet=rs0"
  PORT: "5000"
```

This keeps application configuration outside the container image.

The backend Deployment loads these values using:

```yaml
envFrom:
  - configMapRef:
      name: node-config
```

---

# Helm

The Kubernetes resources are packaged into a Helm chart.

The chart is located at:

```text
student-app/
```

The important Helm files are:

```text
student-app/
├── Chart.yaml
├── values.yaml
└── templates/
```

---

# Chart.yaml

The chart metadata is defined in `Chart.yaml`.

```yaml
apiVersion: v2
name: mern-app
description: MERN Stack Application using Helm
type: application
version: 0.1.0
appVersion: "1.0"
```

---

# values.yaml

The configurable values are centralized in:

```text
values.yaml
```

Example:

```yaml
mongodb:
  image: mongo:7.0
  replicas: 3
  storage: 2Gi
  port: 27017

node:
  image: abhisalunke16/student-backend:v1
  replicas: 2
  port: 5000

react:
  image: abhisalunke16/student-frontend:v1
  replicas: 2
  port: 80

mongoUri: "mongodb://mongodb-0.mongodb:27017,mongodb-1.mongodb:27017,mongodb-2.mongodb:27017/school?replicaSet=rs0"
```

Instead of hardcoding values throughout Kubernetes manifests, Helm templates reference these values.

For example:

```yaml
replicas: {{ .Values.node.replicas }}
```

This makes the chart easier to configure.

---

# Helm Templates

The Kubernetes manifests are converted into Helm templates.

For example, the Node.js Deployment contains:

```yaml
replicas: {{ .Values.node.replicas }}

image: {{ .Values.node.image }}

containerPort: {{ .Values.node.port }}
```

Helm replaces these expressions with values from:

```text
values.yaml
```

---

# Helm Rendering

Before deploying, the templates can be rendered locally.

```bash
helm template .
```

This allows the generated Kubernetes YAML to be inspected before applying it to the cluster.

---

# Helm Lint

The chart can be validated using:

```bash
helm lint .
```

Expected result:

```text
1 chart(s) linted, 0 chart(s) failed
```

---

# Helm Installation

Install the application using:

```bash
helm install student-app .
```

Helm creates a release named:

```text
student-app
```

Check the release:

```bash
helm list
```

Example:

```text
NAME          NAMESPACE   REVISION   STATUS
student-app   default     1          deployed
```

---

# Verify Kubernetes Resources

Check the StatefulSet:

```bash
kubectl get sts
```

Check Deployments:

```bash
kubectl get deploy
```

Check Pods:

```bash
kubectl get pods
```

Check Services:

```bash
kubectl get svc
```

Check PersistentVolumeClaims:

```bash
kubectl get pvc
```

---

# Expected Deployment

The final deployment should look approximately like:

```text
MongoDB:
3/3 pods Running

Node.js:
2/2 pods Running

React:
2/2 pods Running
```

Example:

```text
mongodb-0       1/1   Running
mongodb-1       1/1   Running
mongodb-2       1/1   Running

node-deploy-*   1/1   Running
node-deploy-*   1/1   Running

react-deploy-*  1/1   Running
react-deploy-*  1/1   Running
```

---

# Verify MongoDB ReplicaSet

Connect to MongoDB:

```bash
kubectl exec -it mongodb-0 -- mongosh
```

Check ReplicaSet status:

```javascript
rs.status()
```

Expected:

```text
mongodb-0.mongodb:27017 → PRIMARY
mongodb-1.mongodb:27017 → SECONDARY
mongodb-2.mongodb:27017 → SECONDARY
```

---

# Verify MongoDB Data

Switch to the application database:

```javascript
use school
```

Check students:

```javascript
db.students.find().pretty()
```

Expected data:

```text
Rahul Sharma
Akashad Patel
Amit Kumar
```

---

# Application Verification

The React frontend is exposed using the Kubernetes LoadBalancer service.

Check:

```bash
kubectl get svc react-service
```

The service provides an AWS LoadBalancer hostname.

Opening that hostname in a browser displays:

```text
College Students

Rahul Sharma
Age: 21
Course: Computer Science

Akashad Patel
Age: 22
Course: Information Technology

Amit Kumar
Age: 20
Course: Electronics
```

This confirms that the complete application path is functioning.

---

# End-to-End Request Flow

When a user opens the application:

```text
1. Browser
      │
      ▼
2. AWS LoadBalancer
      │
      ▼
3. React/Nginx Pod
      │
      ├── Static React files
      │
      └── /api/*
              │
              ▼
4. node-service:5000
              │
              ▼
5. Node.js Pod
              │
              ▼
6. MongoDB Headless Service
              │
              ▼
7. MongoDB ReplicaSet
              │
              ▼
8. school.students
```

The response then travels back through the same application layers to the browser.

---

# Helm Release Management

Helm tracks the Kubernetes application as a release.

Check releases:

```bash
helm list
```

Get release information:

```bash
helm status student-app
```

See release history:

```bash
helm history student-app
```

Upgrade the release after changing templates or values:

```bash
helm upgrade student-app .
```

Rollback if necessary:

```bash
helm rollback student-app 1
```

Uninstall the release:

```bash
helm uninstall student-app
```

> Note: Stateful workloads and persistent storage require additional care during uninstall/reinstall operations. Always verify PVC/PV behavior before deleting production data.

---

# Scaling

One advantage of Helm templating is that application configuration can be changed through `values.yaml`.

For example:

```yaml
node:
  replicas: 3

react:
  replicas: 3
```

Then upgrade:

```bash
helm upgrade student-app .
```

Kubernetes will adjust the Deployments accordingly.

MongoDB scaling is different because it is a stateful database and ReplicaSet membership must be considered carefully.

---

# Why Helm?

Without Helm, Kubernetes applications often require multiple YAML files:

```text
ConfigMap
Deployment
Service
StatefulSet
PVC
...
```

Helm packages these resources together.

Instead of manually maintaining separate configurations for every environment, Helm allows reusable templates.

For example:

```text
values.yaml
     │
     ▼
Helm Templates
     │
     ▼
Rendered Kubernetes YAML
     │
     ▼
Kubernetes Cluster
```

This makes deployments more repeatable and configurable.

---

# Raw Kubernetes vs Helm

This project was first deployed using standard Kubernetes YAML manifests.

The workflow was:

```text
Raw Kubernetes YAML
        ↓
Verify application
        ↓
Convert manifests into Helm templates
        ↓
Create values.yaml
        ↓
helm lint
        ↓
helm template
        ↓
helm install
        ↓
Verify application
```

This approach makes it easier to understand what Helm is doing because the Helm chart is based on already-working Kubernetes resources.

---

# Production Considerations

This project demonstrates production-style Kubernetes concepts, but it is still a learning/portfolio environment.

For a real production deployment, several areas should be improved.

## Secrets

The MongoDB connection string is currently stored in a ConfigMap.

Production systems should use Kubernetes Secrets or an external secret-management system for sensitive credentials.

---

## MongoDB Authentication

MongoDB authentication should be enabled in production.

The current learning deployment has authentication disabled.

---

## TLS

The AWS LoadBalancer currently serves the application over HTTP.

Production environments should use HTTPS with a valid TLS certificate.

---

## Resource Limits

Production workloads should define CPU and memory:

```yaml
resources:
  requests:
  limits:
```

---

## Health Checks

Deployments should use:

```text
livenessProbe
readinessProbe
startupProbe
```

to allow Kubernetes to properly determine application health.

---

## Image Optimization

The backend image can be optimized further using:

* Smaller base images
* Multi-stage builds
* `.dockerignore`
* Production-only dependencies

---

## MongoDB Backups

ReplicaSets provide high availability and replication, but they are **not a replacement for backups**.

Production MongoDB should have:

* Automated backups
* Snapshot strategy
* Disaster recovery
* Restore testing

---

## Network Policies

Kubernetes NetworkPolicies can be added to restrict traffic:

```text
React
  ↓
Node
  ↓
MongoDB
```

and prevent unnecessary communication between application tiers.

---

# Security

Do not commit the following to GitHub:

```text
.env
kubeconfig
AWS credentials
private keys
TLS private keys
passwords
MongoDB credentials
```

Use `.gitignore` appropriately.

Example:

```gitignore
.env
*.pem
*.key
.kube/
kubeconfig
```

---

# Learning Objectives

This project demonstrates practical understanding of:

### Docker

* Dockerfiles
* Image creation
* Multi-stage builds
* Docker Hub
* Container networking

### Kubernetes

* Pods
* Deployments
* StatefulSets
* Services
* ConfigMaps
* PVCs
* Persistent storage
* DNS
* ReplicaSets
* LoadBalancers

### MongoDB

* ReplicaSet architecture
* PRIMARY / SECONDARY
* Replication
* Persistent storage
* MongoDB Kubernetes deployment

### Helm

* Helm charts
* Chart.yaml
* values.yaml
* Templates
* Helm functions
* Helm linting
* Helm rendering
* Helm releases
* Helm upgrades
* Helm rollbacks

### AWS

* EC2
* EBS
* AWS LoadBalancer
* Kubernetes on AWS using kOps

---

# Final Architecture

```text
                         ┌─────────────────┐
                         │     Browser     │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │ AWS LoadBalancer│
                         └────────┬────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │ React Deployment         │
                    │ 2 Pods + Nginx           │
                    └────────────┬─────────────┘
                                 │
                              /api/*
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │ node-service             │
                    │ ClusterIP :5000          │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │ Node.js Deployment        │
                    │ 2 Pods                    │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │ MongoDB Headless Service │
                    └────────────┬─────────────┘
                                 │
                  ┌──────────────┼──────────────┐
                  ▼              ▼              ▼
             mongodb-0      mongodb-1      mongodb-2
              PRIMARY        SECONDARY       SECONDARY
                  └──────────────┼──────────────┘
                                 │
                              rs0
                                 │
                                 ▼
                        Persistent Volumes
                                 │
                                 ▼
                              AWS EBS
```

---

# Project Status

Current implementation successfully demonstrates:

* [x] MERN application
* [x] Dockerized frontend
* [x] Dockerized backend
* [x] Docker images pushed to Docker Hub
* [x] Kubernetes cluster using kOps/AWS
* [x] React Deployment
* [x] Node.js Deployment
* [x] MongoDB StatefulSet
* [x] MongoDB ReplicaSet
* [x] Persistent storage
* [x] Kubernetes Services
* [x] ConfigMap
* [x] Nginx reverse proxy
* [x] AWS LoadBalancer
* [x] Helm chart
* [x] Helm values
* [x] Helm templates
* [x] Helm lint
* [x] Helm template validation
* [x] Helm release installation
* [x] End-to-end application testing

---

# Future Improvements

Possible next steps:

* [ ] Kubernetes NetworkPolicies
* [ ] Kubernetes Secrets
* [ ] MongoDB authentication
* [ ] HTTPS / TLS
* [ ] Liveness and readiness probes
* [ ] CPU and memory limits
* [ ] Helm helper templates
* [ ] Helm environment-specific values
* [ ] Helm upgrades and rollback testing
* [ ] CI/CD with GitHub Actions
* [ ] Monitoring with Prometheus
* [ ] Grafana dashboards
* [ ] Centralized logging
* [ ] MongoDB backup and restore strategy
* [ ] Ingress controller
* [ ] GitOps using Argo CD

---
