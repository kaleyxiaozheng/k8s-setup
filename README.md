# Login project design
| Components | Tech Stack | Description |
| :--- | :--- | :--- |
| Frontend | Typescript | UI - sending log on/log in request |
| Backend |	Python | dealing with business logic, connectig database and do validation |
| Database | PostgreSQL | Storing data information (using Docker image) |

# Project Core Steps Breakdown 
<details>
<summary>1. Environment Setup</summary>

- Provision a Virtual Machine (VM) - UTM
- Install Operating System (OS) - Ubuntu 24.04
- Configure Network & SSH
- Disable Swap (`Prerequisites for running Kubernetes`)

</details>

<details><summary>2. Runtime Installation</summary>

- Install Container Runtime - containerd
- Configure Systemd Cgroup Driver - Allow Kubernetes to manage resources more effectively
- Restart & Verify Service
</details>

<details><summary>3. K8S Components Installation</summary>

- Add Kubernetnes Repository
- Install Kubeadm, Kubelet, and Kubectl
- Hold packages - Prevent automatic updates from causing cluster incompatibility 
</details>

<details><summary>4. Cluster Initialization</summary>

1. Initialize Control Plane - operate `kubeadm init`
2. Set up Kubeconfig
3. Deploy Pod Network Add-on (CNI) - Allow communications between pods.
</details>

<details><summary>5. Verfication & Deployment</summary>

- Check Node Status `kubectl get nodes`
- Deploy Login Application
- Expose Service - Allow access the service from the public
</details>

## Steps
- [1. Environment Setup](#1-environment-setup)
- [2. Runtime Installation](#2-runtime-installation)
- [3. K8S Components Installation](#3-k8s-components-installation)
---
## 1. Environment Setup
Simulate multiple Linux servers using Virtual Machines to build a real cluster.

Recommended tools: UTM + Ubuntu Server

1. Download and install [UTM](https://docs.getutm.app/installation/ios/)
2. Download Ubuntu Server for ARM os
3. Set up 2 or 3 VM, 1 for Master Node, 1 or 2 for Worker Nodes

## 2. Runtime Installation
1. Install containerd in each VM
```bash
# install containerd
sudo apt-get update
sudo apt-get install -y containerd
# config containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

## 3. K8S Components Installation
- kubelet(The "Site Manager")
>
> - role: The Node Agent
>
> - Key Action: This is the most important component running on every server (node) in the cluster. It acts like a manager on a construction site. Its job is to make sure that the containers (Pods) are running as they should be.
- kubeadm (The "Architect")
> - role: The Cluster Bootstrapper
>
> - Key Action: You use it once to set up the cluster (kubeadm init) or to add new servers to the cluster (kubeadm join).
- kubectl (The "Command Center")
>
> - role: The Command Line Interface (CLI).

| Tool | Analog |	Frequency of Use	| Where it runs |
| :--- | :--- | :--- | :--- |
| kubeadm	| The Constructor	| Only for setup/updates |	On the servers |
| kubelet	| The Foreman	| Always running (24/7)	| On every server |
| kubectl	| The Remote Control	| Every time you work	| On your Mac |
# Develop and Deploy App Core Steps Breakdown
<details><summary>1. Prepare Docker images</summary>

- ⚛️ Frontend Dockerfile - use Nginx to serve static JS and HTML files
- 🛠️ Backend Dockerfile - install python, run API server
- 🛢 No Dockerfile, config image:postgres in k8s 
</details>

<details><summary>2. Create Kubernetes deployment yaml files (or helm chart - in dockerfile)</summary>

1. deployment.yaml - create pod
2. service.yaml - allow communication internally and externally
</details>

<details><summary>3. Deploy the app to local Cluster</summary>

> **Note:** Config PersistentVolume(PV) and PersistentVolumeClaim(PVC) for database persistence

> **Tips:** 
> 1. Avoid hardcoding IP addresses when connecting to the database. Using Service name (e.g., http://db-service:3306) for database directly access. 预留问题1: 以后使用https
>
> 2. Avoid writing secrets in yaml file, using K8s secret component.

</details>

<details><summary>4. Verify App</summary>

- Confirm containers status are all running using `kubectl get pods`
- Expose the frontend using `kubectl port-forward` or `NodePort service` to access the UL.
- Input account password, or create/update/delete new account, monitor endpoint logs `kubectl logs <backend-pod-names>`
</details>

Environment Setup
## Steps
1. Download UTM (Virtual Machine) and install 

<details><summary>k8s deployment yaml example</summary>

- db.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-db
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password123"
        - name: MYSQL_DATABASE
          value: "userdb"
---
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
```
- backend.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend-container
        image: login-backend:v1
        imagePullPolicy: Never # 强制 K8s 使用本地镜像
        env:
        - name: DB_HOST
          value: "db-service" # 关键：利用 K8s 服务发现连接数据库
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 3000
      targetPort: 3000
```
- port forward
```
# 将集群内后端的 3000 端口转发到你 Mac 的 3000 端口
kubectl port-forward service/backend-service 3000:3000
```
</details>


预留问题1: 以后使用https