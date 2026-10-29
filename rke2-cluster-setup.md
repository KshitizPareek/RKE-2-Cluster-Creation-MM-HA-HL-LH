# Deploying a Kubernetes Cluster with RKE2 on Virtual Machines

A step-by-step guide to deploying a Kubernetes cluster with [RKE2](https://docs.rke2.io/) (Rancher Kubernetes Engine 2) on virtual machines: one master (server) node and two worker (agent) nodes, plus Helm and Longhorn as an optional storage class.

RKE2 is a lightweight, secure Kubernetes distribution, well suited to development and test clusters.

> Based on: [Deploying a Kubernetes Cluster with RKE2 on Virtual Machines](https://medium.com/@jdegbun/deploying-a-kubernetes-cluster-with-rke2-on-virtual-machines-f152ecb51c9f) by Jdegbun.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Configure the Master Node](#step-1-configure-the-master-node)
- [Step 2: Configure Worker 1](#step-2-configure-worker-1)
- [Step 3: Configure Worker 2](#step-3-configure-worker-2)
- [Step 4: Verify the Cluster](#step-4-verify-the-cluster)
- [Optional: Multi-Master (HA) Cluster](#optional-multi-master-ha-cluster)
- [Bonus: Install Helm and Longhorn](#bonus-install-helm-and-longhorn)

---

## Prerequisites

- 3 virtual machines (any platform) with network connectivity between them
- SSH access and `sudo` privileges on each VM
- Internet access on each VM (the installer is downloaded from `get.rke2.io`)

Example cluster used in this guide (per machine): 8 CPU cores, 18 GB RAM, 240 GB SSD.

| Role     | Hostname (example) |
|----------|--------------------|
| Master   | `master`           |
| Worker 1 | `worker1`          |
| Worker 2 | `worker2`          |

**Network ports:** the master must be reachable from the workers on TCP `9345` (RKE2 supervisor API) and `6443` (Kubernetes API). See the [RKE2 networking requirements](https://docs.rke2.io/install/requirements#networking) for the full list.

---

## Step 1: Configure the Master Node

### 1.1 Access the master node

```bash
ssh user@<master-node-ip>
```

Switch to the root user:

```bash
sudo -i
```

### 1.2 Install and start RKE2 server

```bash
curl -sfL https://get.rke2.io | sh -
mkdir -p /etc/rancher/rke2 && echo "write-kubeconfig-mode: '0644'" > /etc/rancher/rke2/config.yaml
systemctl enable --now rke2-server.service
```

> The first start can take a few minutes. Follow progress with `journalctl -u rke2-server -f`.

### 1.3 Configure kubectl on the master node

```bash
ln -s /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl
mkdir -p ~/.kube
ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
```

### 1.4 Add RKE2 binaries to the system path

```bash
export PATH=$PATH:/var/lib/rancher/rke2/bin/
```

To make this permanent:

```bash
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin/' >> ~/.bashrc
```

### 1.5 Retrieve the node token

The workers need this token to join the cluster:

```bash
cat /var/lib/rancher/rke2/server/node-token
```

Save the token somewhere secure. You will need it in the next steps, along with the master node's IP address.

---

## Step 2: Configure Worker 1

### 2.1 Access worker 1

```bash
ssh user@<worker1-node-ip>
```

```bash
sudo -i
```

### 2.2 Install and enable the RKE2 agent

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
systemctl enable rke2-agent.service
```

### 2.3 Create the agent configuration

Replace `<MASTER_NODE_IP>` and `<YOUR_NODE_TOKEN>` with your values, then run:

```bash
mkdir -p /etc/rancher/rke2/
cat > /etc/rancher/rke2/config.yaml <<EOF
server: https://<MASTER_NODE_IP>:9345
token: <YOUR_NODE_TOKEN>
EOF
```

### 2.4 Start the agent

```bash
systemctl start rke2-agent.service
```

---

## Step 3: Configure Worker 2

### 3.1 Access worker 2

```bash
ssh user@<worker2-node-ip>
```

```bash
sudo -i
```

### 3.2 Install and enable the RKE2 agent

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
systemctl enable rke2-agent.service
```

### 3.3 Create the agent configuration

Replace `<MASTER_NODE_IP>` and `<YOUR_NODE_TOKEN>` with your values, then run:

```bash
mkdir -p /etc/rancher/rke2/
cat > /etc/rancher/rke2/config.yaml <<EOF
server: https://<MASTER_NODE_IP>:9345
token: <YOUR_NODE_TOKEN>
EOF
```

### 3.4 Start the agent

```bash
systemctl start rke2-agent.service
```

---

## Step 4: Verify the Cluster

Go back to the **master node** and list the nodes:

```bash
kubectl get nodes -o wide
```

All three nodes should appear and reach the `Ready` state (this can take a minute or two after the agents start).

---

## Optional: Multi-Master (HA) Cluster

A single master is a single point of failure. To make the control plane highly available, add more **server** nodes to the cluster. RKE2 uses embedded etcd by default, so extra servers simply join the existing one.

**Things to know first:**

- Use an **odd number** of servers (3 or 5) so etcd can keep quorum. With 3 masters, the cluster tolerates the loss of 1.
- Extra masters need the same prerequisites as the first (SSH, `sudo`, internet, ports `9345` and `6443` reachable between nodes). Also open TCP `2379-2380` between the masters for etcd.
- Join the new masters **one at a time**. Wait until each shows `Ready` before adding the next.
- For real HA, use a **fixed registration address** (a load balancer, a virtual IP such as kube-vip, or a DNS name that points at all masters) instead of one master's IP. If you only use the first master's IP, the workers and API clients lose access when that node goes down.

In the commands below, `<FIXED_REGISTRATION_ADDRESS>` is that load balancer / VIP / DNS name. If you don't have one yet, use `<MASTER1_IP>`, and you can skip the `tls-san` parts.

### HA.1 Add the fixed address to the first master (skip if not using one)

On the **first master**, add the address to the TLS certificate and restart RKE2:

```bash
sudo -i
cat >> /etc/rancher/rke2/config.yaml <<EOF
tls-san:
  - <FIXED_REGISTRATION_ADDRESS>
EOF
systemctl restart rke2-server.service
```

### HA.2 Get the server token

On the **first master**:

```bash
cat /var/lib/rancher/rke2/server/token
```

By default this is the same value as `node-token` from Step 1.5, so you can reuse it.

### HA.3 Join master 2 (and then master 3)

Run on **each additional master**, one at a time. Replace the placeholders first.

```bash
ssh user@<master2-node-ip>
```

```bash
sudo -i
```

Install RKE2 as a **server** (this is the default install type, so no `INSTALL_RKE2_TYPE` is needed):

```bash
curl -sfL https://get.rke2.io | sh -
```

Create the configuration:

```bash
mkdir -p /etc/rancher/rke2/
cat > /etc/rancher/rke2/config.yaml <<EOF
server: https://<FIXED_REGISTRATION_ADDRESS>:9345
token: <YOUR_NODE_TOKEN>
write-kubeconfig-mode: "0644"
tls-san:
  - <FIXED_REGISTRATION_ADDRESS>
EOF
```

Start the server:

```bash
systemctl enable --now rke2-server.service
```

Follow the logs if you want to watch it join:

```bash
journalctl -u rke2-server -f
```

Once master 2 is `Ready` (see HA.5), repeat HA.3 on `<master3-node-ip>`.

### HA.4 Configure kubectl on the new masters (optional)

Same as Step 1.3, on each new master:

```bash
ln -s /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl
mkdir -p ~/.kube
ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
export PATH=$PATH:/var/lib/rancher/rke2/bin/
```

### HA.5 Verify

From any master:

```bash
kubectl get nodes -o wide
```

The new nodes should show the roles `control-plane,etcd,master`. You can also check etcd health:

```bash
kubectl get pods -n kube-system | grep etcd
```

### HA.6 Point the workers at the fixed address (recommended)

If your workers were configured with a single master's IP, change `server:` in `/etc/rancher/rke2/config.yaml` on each worker to the fixed address, then restart the agent:

```bash
sed -i 's|^server:.*|server: https://<FIXED_REGISTRATION_ADDRESS>:9345|' /etc/rancher/rke2/config.yaml
systemctl restart rke2-agent.service
```

> **Building a new HA cluster from scratch?** Set `token: <A_SECRET_OF_YOUR_CHOOSING>` and `tls-san` in the first master's `config.yaml` **before** its first start. That way you already know the token and every master and worker can use the same `config.yaml` values from the beginning.

---

## Bonus: Install Helm and Longhorn

Run these on the **master node**.

### Install Helm

```bash
curl -#L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Install Longhorn (StorageClass)

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm upgrade -i longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
```

> **Note:** Longhorn requires `open-iscsi` (and `nfs-common` / `nfs-utils` for RWX volumes) on every node. Install them on the master and both workers before deploying Longhorn if they are not already present. See the [Longhorn requirements](https://longhorn.io/docs/latest/deploy/install/#installation-requirements).

Check that Longhorn is running and that the StorageClass exists:

```bash
kubectl get pods -n longhorn-system
kubectl get storageclass
```

---

## Done

You now have a working RKE2 Kubernetes cluster with Longhorn as a storage class.
