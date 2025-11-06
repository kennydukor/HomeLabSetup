# Home Kubernetes Lab Deployment on Proxmox (Supermicro X10SLL-S Platform)

**Author:** Kenechi Dukor 

**Platform:** Supermicro X10SLL-S (Intel Xeon E3-1225 v3, 16 GB RAM, 2 TB SSD, 160 GB HDD)  
**Host OS:** Proxmox VE 9.0.3  
**Network:** 192.168.0.0/24 LAN

---

## 1. System Overview

This home-lab environment uses Proxmox VE as the base hypervisor to host a Kubernetes cluster composed of three Ubuntu 24.04 VMs.
The setup allows containerized services (APIs, dashboards, Nextcloud, etc.) to be deployed in a production-like, reproducible environment.

| Node | Role | IP Address | vCPU | Memory |
|------|------|------------|------|--------|
| k8s-master | Control Plane | 192.168.0.32 | 2 | 3 GB |
| k8s-node1 | Worker Node | 192.168.0.39 | 2 | 4 GB |
| k8s-node2 | Worker Node | 192.168.0.43 | 2 | 4 GB |

Additional 160 GB HDD is mounted at `/mnt/pve/repo160` for repository and persistent storage.

---

## 2. Proxmox Host Configuration

Install Proxmox VE 9.x on the 2 TB SSD.

Create three Ubuntu 24.04 LTS VMs:

- Boot from `ubuntu-24.04.3-live-server-amd64.iso`.
- Disk size = 30–40 GB each.
- Enable EFI Disk and Cloud-Init if available.
- Assign static IPs through `/etc/netplan/` or router DHCP reservation.
- Enable bridge networking on Proxmox (usually `vmbr0`) to give each VM LAN access.

---

## 3. Kubernetes Cluster Setup

### Install Base Components on All Nodes

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg \
  https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl containerd
sudo systemctl enable kubelet containerd
```

### Configure Containerd

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

### Initialize the Control Plane

On `k8s-master`:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Configure kubectl for the current user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

The output provides a join command for workers:

```bash
kubeadm join 192.168.0.32:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Join Worker Nodes

Run that join command (with `sudo`) on both `k8s-node1` and `k8s-node2`.

Check cluster state:

```bash
kubectl get nodes
```

---

## 4. Network & Load Balancing Components

### Flannel (CNI)

**Purpose:** Provides pod-to-pod networking overlay (10.244.0.0/16).

Install:

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Verify:

```bash
kubectl get pods -n kube-flannel -o wide
```

Flannel creates a simple VXLAN-based overlay, allowing pods on different nodes to communicate seamlessly.

### MetalLB (Bare-Metal LoadBalancer)

**Purpose:** Enables LoadBalancer services on non-cloud clusters by assigning IPs from your LAN.

Install:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

Create an IP address pool (example range for LAN):

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  namespace: metallb-system
  name: home-lan-pool
spec:
  addresses:
  - 192.168.0.200-192.168.0.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  namespace: metallb-system
  name: home-lan-advert
spec:
  ipAddressPools:
  - home-lan-pool
EOF
```

Now, any service of type LoadBalancer (like Portainer or Nextcloud) automatically gets a local IP (e.g. 192.168.0.201).

### Ingress NGINX

**Purpose:** Routes external HTTP/HTTPS traffic to internal services using domain names.

Install:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
```

Verify:

```bash
kubectl get pods -n ingress-nginx -o wide
```

### Cert-Manager

**Purpose:** Automates SSL/TLS certificate management.

Install:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.0/cert-manager.yaml
```

Verify:

```bash
kubectl get pods -n cert-manager
```

---

## 5. Management & Monitoring Tools

### Portainer (Kubernetes GUI)

**Purpose:** Visual management of containers, pods, and volumes.

Install:

```bash
kubectl create namespace portainer
kubectl apply -n portainer -f https://downloads.portainer.io/ce2-20/portainer-k8s-nodeport.yaml
```

Access:

- Local: `https://<MetalLB-IP>:9443`
- Default user: create during first login.

### NFS or Local Persistent Storage

Mount the 160 GB HDD and make it available as NFS or hostPath storage.

Example mount:

```bash
sudo mkdir -p /mnt/repo160
sudo mount /dev/sdb1 /mnt/repo160
```

Example PV/PVC:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: repo160-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /mnt/repo160
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: repo160-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
```

Apply:

```bash
kubectl apply -f repo160-storage.yaml
```

---

## 6. Deploying Applications

### Example: NGINX Test Deployment

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer
kubectl get svc nginx
```

If MetalLB is active, you'll get an IP like 192.168.0.201, accessible via browser:

```
http://192.168.0.201
```

### Example: Nextcloud Deployment (Optional)

Persistent cloud storage using the repo160 volume:

```bash
kubectl apply -f https://raw.githubusercontent.com/nextcloud/docker/master/.examples/kubernetes/nextcloud-deployment.yaml
```

---

## 7. Handy Operational Commands

| Purpose | Command |
|---------|---------|
| List all pods in all namespaces | `kubectl get pods -A -o wide` |
| Restart a pod | `kubectl delete pod <pod-name> -n <namespace>` |
| View logs | `kubectl logs <pod-name> -n <namespace>` |
| Describe object (debugging) | `kubectl describe pod <pod-name> -n <namespace>` |
| Check nodes | `kubectl get nodes -o wide` |
| Apply manifest | `kubectl apply -f <file>.yaml` |
| Delete manifest | `kubectl delete -f <file>.yaml` |
| Edit live configuration | `kubectl edit deploy/<name>` |
| SSH into running pod | `kubectl exec -it <pod> -- /bin/bash` |

---

## 8. External Access Options

| Method | Description | Cost |
|--------|-------------|------|
| Port Forwarding | Forward router port 80/443 to MetalLB IP (manual) | Free |
| Cloudflare Tunnel | Secure HTTPS tunnel without port forwarding | Free |
| Ngrok Tunnel | Temporary public URL for testing | Free |
| Static Public IP / VPS | Permanent fixed external address | $5–10 / month |

---

## 9. Expected Behaviour

- Pods schedule dynamically across worker nodes.
- MetalLB assigns real LAN IPs to services.
- Flannel ensures seamless inter-pod communication.
- Ingress NGINX routes multiple domains via a single IP.
- Cert-Manager auto-issues and renews TLS certificates.
- Portainer visualizes workloads and persistent volumes.
- The 160 GB repo acts as shared persistent storage.

---

## 10. Maintenance & Expansion

**Backups:** Use Proxmox snapshot or velero for cluster backup.

**Scaling:** Add more worker VMs; join them using the same `kubeadm join` command.

**Updates:**

```bash
sudo apt update && sudo apt upgrade -y
kubectl version --short
kubeadm upgrade plan
```

**Monitoring (optional):** Deploy kube-prometheus-stack or Grafana for metrics.

---

## 11. Troubleshooting Cheatsheet

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| ErrImagePull | Image not accessible | Use public image or fix registry |
| CrashLoopBackOff | App crash or missing config | Check logs: `kubectl logs <pod>` |
| Node NotReady | Network (Flannel) down | `kubectl delete pod -n kube-flannel -l app=flannel` |
| Service has no external IP | MetalLB misconfigured | Reapply IP pool manifest |
| TLS/HTTPS not working | Cert-Manager not applied | Check `kubectl get certificaterequests` |

---

## 12. Summary

This setup provides a self-contained, production-style Kubernetes cluster running entirely on commodity hardware via Proxmox.

It supports:

- Multi-node orchestration
- Real LAN IP load balancing
- Persistent storage
- HTTPS ingress
- GUI management via Portainer

You can deploy and scale APIs, web services, and micro-apps exactly as in cloud environments — fully under your control.

---

**End of Document**  
*Home Kubernetes Lab Deployment on Proxmox (Supermicro X10SLL-S Platform)*  
© 2025 – Technical Architecture Guide
