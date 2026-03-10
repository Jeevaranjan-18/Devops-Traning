# Kubernetes Clean Setup on Arch Linux (Master + Worker)

## Prerequisites
- Arch Linux on all nodes
- Nodes must be able to ping each other
- Swap disabled

Example:

Master: 192.168.1.10
Worker: 192.168.1.11

---

## 1. Update System

```bash
sudo pacman -Syu
```

---

## 2. Install Required Packages

```bash
sudo pacman -S containerd kubelet kubeadm kubectl iproute2 iptables socat conntrack crictl
```

---

## 3. Enable Container Runtime

```bash
sudo systemctl enable --now containerd
```

Generate config:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

Edit:

```
/etc/containerd/config.toml
```

Change:

```
SystemdCgroup = false
```

To:

```
SystemdCgroup = true
```

Restart containerd:

```bash
sudo systemctl restart containerd
```

---

## 4. Disable Swap

```bash
sudo swapoff -a
```

Edit fstab:

```bash
sudo nano /etc/fstab
```

Comment the swap line.

---

## 5. Enable Kernel Modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Create file:

```
/etc/modules-load.d/k8s.conf
```

Content:

```
overlay
br_netfilter
```

---

## 6. Configure Networking

Create:

```
/etc/sysctl.d/k8s.conf
```

Content:

```
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

Apply:

```bash
sudo sysctl --system
```

---

## 7. Enable kubelet

```bash
sudo systemctl enable --now kubelet
```

---

## 8. Initialize Master Node

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Save the join command shown at the end.

---

## 9. Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Test:

```bash
kubectl get nodes
```

---

## 10. Install Network Plugin (Calico)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Check pods:

```bash
kubectl get pods -A
```

---

## 11. Join Worker Node

Run the join command on worker node:

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## 12. Verify Cluster

```bash
kubectl get nodes
```

Expected output:

```
NAME      STATUS   ROLES           AGE
master    Ready    control-plane
worker    Ready    <none>
```

---

## 13. Test Deployment

Create deployment:

```bash
kubectl create deployment nginx --image=nginx
```

Expose service:

```bash
kubectl expose deployment nginx --type=NodePort --port=80
```

Check pods:

```bash
kubectl get pods
```

