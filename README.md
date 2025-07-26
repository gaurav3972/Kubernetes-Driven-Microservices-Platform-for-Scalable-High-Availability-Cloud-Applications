# 🚀 Deploying and Managing Microservices in a Cloud-Native Environment

## 📌 Project Overview

- Set up a Kubernetes cluster
- Containerize and deploy microservices
- Enable service discovery, autoscaling, and persistence

---

## 🛠️ Technologies Used

- Docker
- Kubernetes (Minikube or EKS/GKE/AKS)
- kubectl
- Docker Hub (for container registry)
- Metrics Server (for autoscaling)
- Load testing tools: `hey`, `ab`, or `curl`
---
![](https://github.com/gaurav3972/Kubernetes-Driven-Microservices-Platform-for-Scalable-High-Availability-Cloud-Applications/blob/main/images/000.0..png)
## 🧩 Deployment Architecture & Setup Summary

### 🔹 **Environment Setup**

| Component          | Setup Used                             |
| ------------------ | -------------------------------------- |
| Kubernetes Cluster | Minikube (for local development)       |
| Container Registry | Docker Hub                             |
| CLI Tools          | `kubectl`, `docker`, `minikube`, `hey` |
| Number of Nodes    | 1 (Minikube single-node cluster)       |

---

### 🔹 **Microservices Deployed**

| Service Name      | Pod Replicas         | Kubernetes Service Type | Accessible At                      |
| ----------------- | -------------------- | ----------------------- | ---------------------------------- |
| `user-service`    | 2                    | NodePort (30001)        | `minikube service user-service`    |
| `product-service` | 2 (auto-scaled to 5) | NodePort (30002)        | `minikube service product-service` |
| `order-service`   | 2                    | NodePort (30003)        | `minikube service order-service`   |

* **Each service is deployed as a separate pod group (Deployment)**
* **Each is exposed using a Kubernetes `Service` (NodePort)** for external access.

---

### 🔹 **Autoscaling Setup**

| Service           | HPA Configured | Min Replicas | Max Replicas | Target CPU Utilization |
| ----------------- | -------------- | ------------ | ------------ | ---------------------- |
| `product-service` | ✅ Yes          | 2            | 5            | 50%                    |

> The autoscaler will increase the number of pods based on CPU load.
![](https://github.com/gaurav3972/Kubernetes-Driven-Microservices-Platform-for-Scalable-High-Availability-Cloud-Applications/blob/main/images/Screenshot%202025-07-06%20163437.png)
---

### 🔹 **Storage & Data Persistence**

| Service         | Persistent Volume      | Path Used in Container |
| --------------- | ---------------------- | ---------------------- |
| `order-service` | `/mnt/data` (hostPath) | `/app/data`            |

* The `order-service` is bound to a Persistent Volume and Claim.
* Data stored here survives pod restarts and reschedules.

---

## 🗂️ Deployment Flow (Step-by-Step Summary)

1. **Start Kubernetes cluster** (Minikube or cloud).
2. **Build Docker images** for each service and **push** them to Docker Hub.
3. **Create Kubernetes Deployments** for each microservice with `replicas: 2`.
4. **Create Kubernetes Services** to expose microservices.
5. **Configure and apply HPA** for `product-service` to scale between 2–5 replicas.
6. **Configure Persistent Volume (PV)** and **Persistent Volume Claim (PVC)** for `order-service` to store data.
7. **Deploy all Kubernetes manifests** using `kubectl apply -f k8s-manifests/`.
8. **Access services externally** via `minikube service <service-name>`.
9. **Test autoscaling** by generating CPU load using `hey`.
10. **Test persistence** by writing data through `order-service`, then restarting pods and verifying the data remains.
![](https://github.com/gaurav3972/Kubernetes-Driven-Microservices-Platform-for-Scalable-High-Availability-Cloud-Applications/blob/main/images/Screenshot%202025-07-07%20124208.png)
---
## 1️⃣ Kubernetes Cluster Setup

### 🔹 Option 1: Local (Minikube)
```
minikube start
kubectl get nodes
````
### 🔹 Option 2: Cloud (EKS, GKE, or AKS)

#### AWS EKS (Example)

```bash
eksctl create cluster --name microservices-cluster --region us-east-1 --nodes 2
aws eks update-kubeconfig --name microservices-cluster --region us-east-1
kubectl get nodes
```

---

## 2️⃣ Microservices & Dockerization

### 🧱 Each service contains:

* `app.js` (Node.js or Python app)
* `Dockerfile`

### 🔸 Sample `Dockerfile` (Node.js)

```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["node", "app.js"]
```

### 🔸 Build and Push Docker Images

```bash
docker build -t <dockerhub-username>/user-service ./user-service
docker push <dockerhub-username>/user-service

docker build -t <dockerhub-username>/product-service ./product-service
docker push <dockerhub-username>/product-service

docker build -t <dockerhub-username>/order-service ./order-service
docker push <dockerhub-username>/order-service
```

---

## 3️⃣ Kubernetes Deployment

### 🔸 Example: `user-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user
  template:
    metadata:
      labels:
        app: user
    spec:
      containers:
        - name: user
          image: <dockerhub-username>/user-service
          ports:
            - containerPort: 3000
```

### 🔸 Apply Deployments

```bash
kubectl apply -f k8s-manifests/user-deployment.yaml
kubectl apply -f k8s-manifests/product-deployment.yaml
kubectl apply -f k8s-manifests/order-deployment.yaml
```
![](https://github.com/gaurav3972/Kubernetes-Driven-Microservices-Platform-for-Scalable-High-Availability-Cloud-Applications/blob/main/images/Screenshot%202025-07-06%20163700.png)
---

## 4️⃣ Service Discovery and Load Balancing

### 🔸 Internal Communication: `ClusterIP`

### 🔸 External Access: `NodePort` or `LoadBalancer`

### 🔸 Example: `user-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: NodePort
  selector:
    app: user
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30001
```

### 🔸 Apply Services

```bash
kubectl apply -f k8s-manifests/user-service.yaml
kubectl apply -f k8s-manifests/product-service.yaml
kubectl apply -f k8s-manifests/order-service.yaml
```

### 🔸 Access the services

```bash
minikube service user-service
```

---

## 5️⃣ Horizontal Pod Autoscaling (HPA)

### 🔸 Prerequisite (Minikube only)

```bash
minikube addons enable metrics-server
```

### 🔸 `hpa-product.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-deployment
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

### 🔸 Apply and Monitor HPA

```bash
kubectl apply -f k8s-manifests/hpa-product.yaml
kubectl get hpa
```

### 🔸 Simulate Load

```bash
hey -z 1m -c 50 http://<NODE-IP>:<PORT>/product
```

---

## 6️⃣ Persistent Volume & Claim

### 🔸 `pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: order-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

### 🔸 `pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: order-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### 🔸 Modify `order-deployment.yaml`

```yaml
      volumes:
        - name: order-storage
          persistentVolumeClaim:
            claimName: order-pvc
      containers:
        - name: order
          image: <dockerhub-username>/order-service
          volumeMounts:
            - mountPath: "/app/data"
              name: order-storage
```

### 🔸 Apply PV and PVC

```bash
kubectl apply -f k8s-manifests/pv.yaml
kubectl apply -f k8s-manifests/pvc.yaml
```
![](https://github.com/gaurav3972/Kubernetes-Driven-Microservices-Platform-for-Scalable-High-Availability-Cloud-Applications/blob/main/images/Screenshot%202025-07-06%20163648.png)
---
![](https://github.com/gaurav3972/Kubernetes-Driven-Microservices-Platform-for-Scalable-High-Availability-Cloud-Applications/blob/main/images/Screenshot%202025-07-06%20163830.png)
![](https://github.com/gaurav3972/Kubernetes-Driven-Microservices-Platform-for-Scalable-High-Availability-Cloud-Applications/blob/main/images/Screenshot%202025-07-06%20163843.png)
![](https://github.com/gaurav3972/Kubernetes-Driven-Microservices-Platform-for-Scalable-High-Availability-Cloud-Applications/blob/main/images/Screenshot%202025-07-06%20163900.png)
![](https://github.com/gaurav3972/Kubernetes-Driven-Microservices-Platform-for-Scalable-High-Availability-Cloud-Applications/blob/main/images/Screenshot%202025-07-06%20163928.png)
![](https://github.com/gaurav3972/Kubernetes-Driven-Microservices-Platform-for-Scalable-High-Availability-Cloud-Applications/blob/main/images/Screenshot%202025-07-06%20163943.png)
Here is your **project summary in paragraph form** along with a ✅ **Validation Checklist**:

---

### ✅ **Validation Checklist**

* [x] Microservices are containerized and pushed to Docker Hub
* [x] Kubernetes cluster is up and running (Minikube or EKS)
* [x] Deployments for `user`, `product`, and `order` services are applied
* [x] NodePort services are configured for external access
* [x] Horizontal Pod Autoscaler (HPA) is set up for `product-service`
* [x] Load testing using `hey` confirms autoscaling functionality
* [x] Persistent Volume and PVC are bound to `order-service`
* [x] Data remains intact after pod restarts or rescheduling
* [x] All manifests are correctly applied using `kubectl apply -f`
* [x] Each service is accessible via `minikube service <service-name>`

---

  ### 📝 **Project Summary**
The project titled **“Kubernetes-Driven Microservices Platform for Scalable High-Availability Cloud Applications”** focuses on deploying and managing a set of containerized microservices in a cloud-native environment using Kubernetes. It includes three primary services—`user-service`, `product-service`, and `order-service`—each built using Node.js or Python and containerized using Docker. These services are deployed into a Kubernetes cluster (either locally with Minikube or in the cloud via EKS), with each service running as a separate deployment and exposed externally using NodePort services. The `product-service` includes Horizontal Pod Autoscaler (HPA) configuration to automatically scale the number of pods based on CPU utilization, allowing it to grow from 2 to 5 replicas under load. Persistent data storage is handled by binding a Persistent Volume (PV) and Persistent Volume Claim (PVC) to the `order-service`, ensuring data survives restarts and rescheduling. All services and configurations are managed through Kubernetes manifests using `kubectl`. The project ensures scalable, resilient, and persistent application delivery with proper service discovery, load balancing, autoscaling, and volume management features built-in.
