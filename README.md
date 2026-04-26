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

> **TIP:** Why do we go through the trouble of modifying /etc/fstab?
Because Kubernetes is designed around the principle of “full control.” If the system allows swap (virtual memory), then when memory runs low, Linux may silently move data to disk. This makes it difficult for Kubernetes to accurately measure and manage Pod performance.

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
<details><summary>Explanation:</summary>
以上设置是Kubernetes网络配置中非常低层。目的是确保容器之间的网络流量能够被Linux内核正确地拦截、转发和处理。以上代码是在Master Node上操作。

1. 开启内核模式（设置桥接流量）
Kubernetes 的 Pod 网络插件（如 Calico 或 Flannel）通常依赖 overlay 和 br_netfilter 这两个内核模块。
•	overlay: 允许容器文件系统的分层叠加。
•	br_netfilter: 使得经过 Linux 网桥的流量能够被 iptables 处理。
```bash
# 创建一个配置文件，让系统在重启后自动加载这些模块
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 立即手动加载模块
sudo modprobe overlay
sudo modprobe br_netfilter
```
2. 配置 sysctl 参数 (允许 iptables 检查流量)。加载了模块后，你还需要显式地告诉 Linux 内核，把网桥上的流量交给 iptables 规则去过滤。这是实现 K8s Service（负载均衡）和 Network Policy（安全策略）的基础。
```bash
# 创建 sysctl 配置文件
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用参数，无需重启
sudo sysctl --system
```
3. 为什么要这样做？（原理拆解）
在默认情况下，Linux 的网桥（Bridge）工作在数据链路层（L2），而 iptables 工作在网络层（L3）。
•	如果没有 br_netfilter: 容器之间的二层流量会直接通过网桥转发，绕过 iptables。
•	后果: K8s 无法通过 iptables 规则来实现 Service 的转发，也无法通过 Network Policy 来阻断不合规的流量。
4. 验证配置是否生效
配置完成后，你可以运行以下命令检查：
	1.	检查模块: lsmod | grep br_netfilter （如果有输出则正常）。
	2.	检查内核参数: sysctl net.bridge.bridge-nf-call-iptables （输出应为 1）。
</details>


2. Install containerd
```bash
sudo apt update
sudo apt install -y containerd
```

3. Generate the default configuration file (Critical Step)
```bash
# K8s requires a specific configuration for containerd, so to generate a default fire fist.
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
# Modify the configuration file to use SystemdCgroup (for better performance and stability)
# Set SystemdCgroup to true in the configuration file to ensure stable resource management
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

4. Verify containerd status with command `sudo systemctl status containerd`
```bash
# Reboot and verify
sudo systemctl restart containerd
sudo systemctl status containerd
```
![image](./img/verify_containerd_status.png)

![image](./img/verify_network_config_before_install_kubeadm.png)

5. Install kubeadm, kubelet, kubectl
```bash
# 1. Update apt and install basic dependencies
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# 2. Download k8s signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 3. Add software repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 4. Installation
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 5. Pin the version, to prevent accidental upgrades that could break the cluster
sudo apt-mark hold kubelet kubeadm kubectl
```

$Knowlege$:  
1. Why download k8s signing key?
> This is essentially the foundational step in building Software Supply Chain Security.
>
> Downloading the signing key is about verifying the authenticity and integrity of the software packages. It ensures that the tools you are downloading (like kubeadm and kubelet) are genuinely released by the official Kubernetes project and haven't been tampered with or "injected" with malicious code by a third party.
>
 > <details><summary>Here are the deeper reasons behind this</summary> 
> 1. Preventing Man-in-the-Middle (MITM) Attacks. When you download software via `apt` or `yum`, the data packets travel through numerous routers and mirror servers. Without key verification, a hacker could intercept your request and serve you a counterfeit installation package containing a Trojan.
>
> - The Role of the Key: When the official maintainers release a package, they use a private key to create a digital signature for the file.
>
> - Your Role: The "signing key" you download is actually the public key. The apt tool uses this public key to decrypt and verify the package signature. If they don't match, the system will reject the installation and throw an error.  
>
> 2.Establishing a Chain of Trust
Linux package managers (like apt) do not trust third-party repositories by default.
>
> •	When you add the Kubernetes mirror address to your sources.list, if you don't provide the corresponding GPG key, the system will prompt: "The following signatures couldn't be verified" during an apt update.
>
> •	This happens because the system cannot confirm the identity of this new repository. Importing the key is your way of telling the operating system: "I trust this repository; allow it to proceed."
>
> 3.Evolution of Key Formats (The Fine Print)
You may notice in newer installation guides that keys are usually placed in the /etc/apt/keyrings/ directory and processed using gpg --dearmor.
•	In the past: Keys were globally stored in /etc/apt/trusted.gpg. This posed a security risk because if a single key was compromised, every repository in the system was potentially exposed.
•	Current (Best Practice): Each software source's key is stored independently and bound to its specific .list file. This follows the Principle of Least Privilege and is the more secure method currently recommended by Kubernetes.</details>


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