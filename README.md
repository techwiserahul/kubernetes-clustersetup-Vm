*#Kubernetes Cluster Setup using kubeadm**
This guide will walk you through the steps to set up a Kubernetes cluster using kubeadm on Linux VMs. This setup includes one master node and multiple worker nodes.
**
**Prerequisites****
A Linux-based VM (Ubuntu 20.04 LTS is used in this guide).
At least 2 GB of RAM and 2 CPUs on each VM.
Root or sudo access to the VMs.
Networking setup between the master and worker nodes.
Internet access for downloading software.
Step 1: Prepare the Nodes
Update the System
Update the package index and install required packages:


#sudo apt-get update && sudo apt-get upgrade -y
#sudo apt-get install -y apt-transport-https curl
#**Disable Swap
Disable swap to ensure kubelet functions properly**:

sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
**Load Kernel Modules
Load necessary kernel modules and set system configurations:**

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

**To Install Cri-dockerD  ******
git clone https://github.com/Mirantis/cri-dockerd.git
 cd cri-dockerd
mkdir bin
go build -o bin/cri-dockerd
mkdir -p /usr/local/bin
sudo install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
sudo install packaging/systemd/* /etc/systemd/system
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable --now cri-docker.socket
**Step 3: Install kubeadm, kubelet, and kubectl
Add the Kubernetes APT repository and install the necessary packages:******

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
**Step 4: Initialize the Master Node
On the master node, initialize the Kubernetes cluster:**

sudo kubeadm init --pod-network-cidr=192.168.0.0/16
**After initialization, set up the kubeconfig file for the root user:******

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
**Alternatively, if you are using a different user:******
mkdir -p /home/youruser/.kube
sudo cp -i /etc/kubernetes/admin.conf /home/youruser/.kube/config
sudo chown youruser:youruser /home/youruser/.kube/config
**Step 5: Install a Pod Network Addon
Install a network addon like Calico for pod networking:******

**for setup calico & restart kubelet******
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
sudo mkdir -p /opt/cni/bin
sudo tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-v1.1.1.tgz

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

ls /etc/cni/net.d/


sudo systemctl daemon-reload
sudo systemctl restart kubelet

**For logs**
journalctl -u kubelet -b
**Step 6: Join Worker Nodes to the Cluster
On each worker node, use the join command provided by kubeadm init on the master node. If you don't have the command, generate a new token on the master node:******

kubeadm token create --print-join-command
**Run the kubeadm join command on each worker node:******


sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

**For worker node****
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
**Step 7: Verify the Cluster
On the master node, verify that all nodes have joined the cluster and are in the Ready state:******
kubectl get nodes
You should see both the master and worker nodes listed with their statuses.

**Step 8: Deploy a Test Application
Deploy a simple Nginx application to verify the cluster is working:******

kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
**Get the NodePort assigned to the Nginx service:******

kubectl get svc
**Access Nginx using the NodePort and the worker node's IP address in a web browser to confirm the setup.**

**Troubleshooting**
Check Node Status: If nodes are not in the Ready state, check the kubelet logs.
Network Issues: Ensure all required ports are open and that nodes can communicate over the network.
Swap Issues: Ensure swap is disabled on all nodes.
Certificates: Ensure the date and time are synchronized across all nodes to avoid certificate issues.
Conclusion
You have now successfully set up a Kubernetes cluster using kubeadm on Linux VMs. You can now deploy and manage containerized applications on your cluster.
