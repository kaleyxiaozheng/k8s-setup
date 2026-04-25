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

``` bash
# Install on all nodes
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Summary
### 1. Create Master Node and name it as `k8s-master`
> Choose `Create a New Virtual Machine`, then `Virtualize` for best performance
>
>Choose Ubuntu Server ARM64 iso file
>
> Setup hardware:
>
> - RAM: save at least 2048 MB (2GB). 
>
> **NOTE:** A minimum of 2GB of RAM is required; otherwise, the K8s control plane will not start.
>
> - CPU: 2
>
> - Storage: 20GB or more (K8s images are quite large)
>
> Share context: skip for now
>
> Tick Install OpenSSH server, then you can access the master node via Mac Terminal.
>
> `IPv4 address for enp0s1: 192.168.64.2` is somewhere telling you the IP address like family address.

$Knowlege$:   
1. Why use LVM?
> LVM (Logical Volume Management) is a highly practical technology. Simply put, it acts like a "flexible rack" for your hard drive. If you find your disk space running low due to too many Kubernetes images, LVM allows you to resize partitions easily without reinstalling the OS. This perfectly aligns with the "multi-dimensional capabilities" you aim to build.
>
> If you selected 'Use LVM' while installing Ubuntu, and later find that /var/lib/containerd (where images are stored) is full, you can 'borrow' free disk space for it with just a few simple commands, without having to start all over again.
>
2. `sudo apt install htop` to manage your Linux infrastructure.
>
> `apt` (Advanced package Tool): This is the Package Manager for Ubuntu. View it Like `App Store` for Linux command lines.
>
> `htop` software name. It is an interactive process viewer and system monitor
>
3. turn off Swap `sudo sed -i '/swap/d' /etc/fstab` 

### Verify
1. Start Master Server, input username and password, then check IP address `ip addr show enp0s1`
![image](./img/k8s_master_login.png)

2. SSH Connect from Mac terminal `ssh ubuntu@192.168.64.2`
![image](./img/SSH_k8s_master_server.png)

3. Check environment to ensure Swap is closed `free -h`
![image](./img/verify_swap_close.png)
> Run `sudo swapoff -a` if all values in line Swap are not 0

### 2. Install Container Runtime in `k8s-master` server
1. Forward IPv4 and allow iptables to see bridged traffic. This is a mandatory prerequisite for Kubernetens networking.
>
> Run following commands either via k8s-master server or via Mac terminal (need to ssh k8s-master server first)
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure the required sysctl parameters and ensure they persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters without a reboot.
sudo sysctl --system
```

2. Install containerd
```bash
sudo apt update
sudo apt install -y containerd
```

3. Generate the default configuration file (Critical Step)
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
# Modify the configuration file to use SystemdCgroup (for better performance and stability)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

4. Verify containerd status with command `sudo systemctl status containerd`
![image](./img/verify_containerd_status.png)

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

**TIP:**
1. `sudo poweroff` to turn off k8s-master node
预留问题1: 以后使用https