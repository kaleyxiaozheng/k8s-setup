# Login project design
| Components | Tech Stack | Description |
| :--- | :--- | :--- |
| Frontend | Typescript | UI - sending log on/log in request |
| Backend |	Python | dealing with business logic, connectig database and do validation |
| Database | PostgreSQL | Storing data information (using Docker image) |

# Core Steps Breakdown
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
1. Download UTM (Virtual Machine) and install 