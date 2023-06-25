# Steps to configure Kubeadm

## Vagrant
First I installed vagrant and created 3 virtual images. One for control plane "kubemaster" and 2 worker notes "kubenode01" and "kubenode02"

## Container Runtime
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
Verify the followig
```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

Verify the following, It should return 1 for each
```bash
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### Installing containerd

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install containerd.io
systemctl status containerd
```

### Configure systemd cgroup driver

Remove all the contents from file `/etc/containerd/config.toml`

Paste the following lines in the same file and then restart the containerd service by runnin `sudo systemctl restart containerd`
```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

## Cluster Setup

### Install Kubeadm, Kubectl, Kubelet
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Initialize Kubeadm (This one should be run on only controlplane node)
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --api-server-advertise-address=192.168.56.2
```
Once this command finishes, it gives you the following code to copy paste in the terminal

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

After this step you can see the master node is visible to to the cluster
```bash
kubectl get nodes
```

I have got the following token `0pvbvp.z1hcprdc7ah7lg3d`. It should be used to join worker nodes.
I also retrieved the --discovery-token-ca-hash and it is `9e7411fe2071b322c117233c174fedf1fa289407e8fbf3135bb8ffe29886b9fc`

```bash
kubeadm join 192.168.56.2:6443 --token 0pvbvp.z1hcprdc7ah7lg3d \ 
--discovery-token-ca-cert-hash sha256:9e7411fe2071b322c117233c174fedf1fa289407e8fbf3135bb8ffe29886b9fc
```

Before joining nodes, I am now installing weavenet Pod networking plugin as suggested by the kubeadm init command output

```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

The weavenet document suggessted that we should edit the weavenet daemonset and add a new env variable for the weave container
with variable name "IPALLOC_RANGE" and the value should be the same as given to --pod-network-cidr flag during `kubeadm init` in my case it is `10.244.0.0/16
