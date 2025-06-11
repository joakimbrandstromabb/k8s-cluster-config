## Kubernetes Cluster Setup on Proxmox using Fedora, kubeadm, CRI-O

⸻

✅ Overview

| Component       | Role                     |
|----------------|--------------------------|
| Fedora          | Base OS on Proxmox VMs   |
| kubeadm         | Bootstrap the cluster    |
| CRI-O           | Container runtime        |
| Calico          | CNI for networking       |
| CoreDNS         | Cluster DNS              |
| MetalLB         | LoadBalancer support     |
| NGINX Ingress   | Ingress Controller       |
| Longhorn        | Persistent storage       |
| Keycloak        | Auth / OIDC              |
| ArgoCD          | GitOps                   |

⸻

🖥️ 1. Create Fedora VMs on Proxmox

Create 3+ Fedora VMs:
	•	Fedora Server 39+
	•	At least 2vCPU, 4GB RAM (8+ GB for Longhorn nodes)
	•	Use static IPs (or DHCP reservation)

Set hostname:

```bash
hostnamectl set-hostname k8s-master-01
```

⸻

⚙️ 2. System preparation

This setup assumes SELinux is kept in permissive mode throughout, while swap and firewalld are disabled as required by kubeadm and Kubernetes networking components.

⸻

✅ Updated Plan: Keep SELinux permissive, disable swap + firewalld, enable IP forwarding

🛠 On all nodes, do the following in order:

1. Disable swap

```bash
swapoff -a
sed -i '/swap/d' /etc/fstab
```

⚠️ kubelet will refuse to start if swap is on.

2. Enable IP forwarding

```bash
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/k8s.conf
sysctl --system
```

3. Disable firewalld

```bash
systemctl disable --now firewalld
```

Even though firewalld can be configured, it’s simpler (and safer for MetalLB/Calico) to let Kubernetes manage iptables rules directly.

4. Set SELinux in permissive mode (effectively disabling it)

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Add the Kubernetes yum repository

⸻

🧱 3. Install CRI-O

#### Define the Kubernetes version and used CRI-O stream

```bash
KUBERNETES_VERSION=v1.33
CRIO_VERSION=v1.33
```

### Distributions using `rpm` packages

#### Add the CRI-O repository

```bash
cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/repodata/repomd.xml.key
EOF
```

#### Install package dependencies from the official repositories

```bash
dnf install -y container-selinux
```

#### Install the packages

```bash
dnf install -y cri-o kubelet kubeadm kubectl
```

#### Configure a Container Network Interface (CNI) plugin

CRI-O is capable of working with different [CNI plugins](https://github.com/containernetworking/cni),
which may require a custom configuration. The CRI-O package ships a default
[IPv4 and IPv6 (dual stack) configuration](templates/latest/cri-o/bundle/10-crio-bridge.conflist.disabled)
for the [`bridge`](https://www.cni.dev/plugins/current/main/bridge) plugin,
which is disabled by default. The configuration can be enabled by renaming the
disabled configuration file in `/etc/cni/net.d`:

```bash
mv /etc/cni/net.d/10-crio-bridge.conflist.disabled /etc/cni/net.d/10-crio-bridge.conflist
```

The bridge plugin is suitable for single-node clusters in CI and testing
environments. Different CNI plugins are recommended to use CRI-O in production.

#### Start CRI-O

```bash
systemctl start crio.service
```

⸻

🔧 4. Install kubeadm, kubelet, kubectl

```bash
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
EOF
```

```bash
dnf install -y kubelet kubeadm kubectl
systemctl enable --now kubelet
```

⸻

🚀 5. Initialize cluster (on control plane)

Before initializing the cluster, ensure IP forwarding is enabled:

```bash
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/k8s.conf
sysctl --system
```

Then run:

```bash
kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///var/run/crio/crio.sock
```

⚠️ Save the kubeadm join command output somewhere safe — you'll need it for adding worker nodes.

🔹 kubeadm init

This is the main command that:
	•	Bootstraps your Kubernetes control plane
	•	Sets up etcd, the API server, controller-manager, scheduler, and kubelet
	•	Generates certificates, config files, and the kubeconfig for your cluster

⸻

🔹 --pod-network-cidr=192.168.0.0/16

This tells Kubernetes:
	•	What IP range to allocate for pods (i.e. the internal network between containers)
	•	You must match this to your CNI plugin (like Calico or Flannel)

✅ Example:
	•	Calico uses 192.168.0.0/16 by default
	•	If you plan to use Calico, this value is correct

  •	The pod network (192.168.0.0/16) is virtual and internal to Kubernetes.
	•	It won’t conflict with your home LAN (192.168.1.0/24) as long as there’s no overlap.

⸻

🔹 --cri-socket=unix:///var/run/crio/crio.sock

This tells kubeadm:
	•	Use CRI-O as the container runtime
	•	The default socket location for CRI-O is /var/run/crio/crio.sock

Without this flag, kubeadm may default to containerd or dockershim (deprecated).

After init:

```bash
mkdir -p ~/.kube
cp /etc/kubernetes/admin.conf ~/.kube/config
```

Run the kubeadm join ... output on your worker nodes.

⸻

🚀 6. Bootstrap ArgoCD and GitOps

Instead of applying network and infrastructure YAMLs manually, we use ArgoCD to manage everything declaratively from a Git repo.

### 1. Install ArgoCD (Early)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Prepare Git Repository

Create a Git repo with this structure:

```
k8s-cluster-config/
├── base/
│   ├── calico/
│   │   └── calico.yaml
│   ├── metallb/
│   │   ├── metallb.yaml
│   │   └── ip-pool.yaml
│   ├── ingress/
│   │   └── nginx.yaml
│   ├── longhorn/
│   │   └── longhorn.yaml
│   ├── keycloak/
│   │   └── helm-values.yaml
│   └── argocd/
│       └── apps.yaml
```

Populate each directory with the respective manifests (e.g. Calico from GitHub, MetalLB config, etc.).

### 3. Create an ArgoCD App of Apps

Create `argocd/apps.yaml` with:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: infrastructure
  namespace: argocd
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/k8s-cluster-config
    targetRevision: HEAD
    path: base
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Then apply it:

```bash
kubectl apply -f argocd/apps.yaml
```

Now ArgoCD will take care of applying Calico, MetalLB, Longhorn, Ingress, Keycloak, etc.

⸻

🌐 7. Install Calico (CNI)

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

⸻

💡 8. Join worker nodes

Steps required on worker nodes:

1. Disable swap (same as control plane):

```bash
swapoff -a
sed -i '/swap/d' /etc/fstab
```

2. Enable IP forwarding:

```bash
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/k8s.conf
sysctl --system
```

3. Disable firewalld:

```bash
systemctl disable --now firewalld
```

4. Set SELinux in permissive mode:

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

5. Install Kubernetes tooling:

```bash
dnf install -y kubelet kubeadm kubectl
systemctl enable --now kubelet
```

6. Install CRI-O (same repo setup as control plane), enable and start CRI-O service:

```bash
dnf install -y cri-o
systemctl enable --now crio
```

7. Join the cluster using the token and hash from the master output:

```bash
kubeadm join ... --cri-socket=unix:///var/run/crio/crio.sock
```

8. Change the Role label

```bash
kubectl label node kube-worker-01.192.168.1.207 node-role.kubernetes.io/worker=worker
```
⸻

📦 9. Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

```bash
kubectl create namespace metallb-system
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pool1
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: adv1
  namespace: metallb-system
EOF
```

⸻

🌍 10. Install NGINX Ingress

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

MetalLB should assign a LoadBalancer IP.

⸻

💾 11. Install Longhorn

Requires open-iscsi on all nodes:

```bash
dnf install -y iscsi-initiator-utils
systemctl enable --now iscsid
```

Then:

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.1/deploy/longhorn.yaml
```

⸻

🔐 12. Install Keycloak (Helm)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install keycloak bitnami/keycloak \
  --namespace keycloak --create-namespace \
  --set auth.adminUser=admin \
  --set auth.adminPassword=adminpass \
  --set ingress.enabled=true \
  --set ingress.hostname=keycloak.skinode.org
```

Make sure DNS points to MetalLB-assigned IP.

⸻

🚀 13. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Optional ingress:

```yaml
# argo-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: argocd.skinode.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```

Example Ingress for Keycloak with TLS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: keycloak
  namespace: keycloak
  annotations:
    cert-manager.io/cluster-issuer: "internal-ca"
spec:
  rules:
  - host: keycloak.skinode.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: keycloak
            port:
              number: 80
  tls:
  - hosts:
    - keycloak.skinode.org
    secretName: keycloak-tls
```

⸻

🧠 14. CoreDNS is already running

To tweak the config:

```bash
kubectl -n kube-system edit configmap coredns
```

⸻

📋 15. Add Cert-Manager with Internal CA

Install `cert-manager` and configure a private internal CA for TLS.

```bash
kubectl create namespace cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Generate a self-signed CA:

```bash
openssl req -x509 -newkey rsa:2048 -keyout ca.key -out ca.crt -days 3650 -nodes -subj "/CN=skinode-internal-ca"
```

Create a Kubernetes secret for the CA:

```bash
kubectl -n cert-manager create secret tls skinode-ca \
  --cert=ca.crt \
  --key=ca.key
```

Define a `ClusterIssuer` that uses the CA:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: skinode-ca
```

Apply it:

```bash
kubectl apply -f internal-ca.yaml
```

Use the `internal-ca` in your Ingress resources:

```yaml
annotations:
  cert-manager.io/cluster-issuer: "internal-ca"
tls:
- hosts:
  - keycloak.skinode.org
  secretName: keycloak-tls
```

To trust the certificates, install `ca.crt` on your client devices (browsers or OS trust store).

To export the CA cert for client trust:

```bash
kubectl get secret -n cert-manager skinode-ca -o jsonpath='{.data.tls\.crt}' | base64 -d > ca.crt
```

⸻