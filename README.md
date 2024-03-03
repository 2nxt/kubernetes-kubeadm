# Setup Kubernetes Cluster

These steps are necessary to set up a Kubernetes cluster on both nodes. 


### Step 1: Enable iptables for bridge networking and swap off

Run the following commands on both nodes:

```bash
# Add necessary kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```
Run the following commands on both nodes

```bash
#remove or hash the swap line
{
sed -i 's/\/swapfile swap swap defaults 0 0/#\/swapfile swap swap defaults 0 0/' /etc/fstab
swapoff -a
}
```
### Step 2: Download and Install Container Networking

The commands in this lab must be run on each instance: `control plane`, and `worker`. Login to each instance using SSH Terminal.

The versions chosen here align with those that are installed by the current kubernetes-cni package for a v1.29 cluster.

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update 
sudo apt-get install -y containerd.io

```

Next, Create a containerd configuration file

```bash
sudo mkdir -p /etc/containerd

sudo containerd config default | sudo tee /etc/containerd/config.toml
```

#### Set the cgroup driver for runc to systemd

Set the cgroup driver for runc to systemd, which is required for the kubelet.

At the end of this section in /etc/containerd/config.toml
``
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      ...
``

Change the value for SystemCgroup from false to true.
``
            SystemdCgroup = true
``
If you like, you can use sed to swap it out in the file without having to edit the file manually.

```bash 
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
```

#### Restart containerd with the new configuration

```bash

sudo systemctl restart containerd
```

### Step 3: Install Kubernetes components

Install `kubeadm`, `kubectl`, and `kubelet` on all nodes:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo mkdir -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# To see the new version labels
sudo apt-cache madison kubeadm

sudo apt-get install -y kubelet=1.29.0-1.1 kubeadm=1.29.0-1.1 kubectl=1.29.0-1.1

sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 4: Initialize the Kubernetes Control Plane (Master Node)

Run the following command on the control plane node to initialize the Kubernetes control plane:

```bash
# Get the IP address allocated to enp1s0 | ensure that's the correct interface name

IP_ADDR=$(ip addr show enp1s0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

# Initialize the control plane node
kubeadm init --apiserver-cert-extra-sans=controlplane --apiserver-advertise-address $IP_ADDR --pod-network-cidr=10.244.0.0/16

```

### Step 5: Configure Kubeconfig for Control Plane

Once the initialization is done, set up the default kubeconfig file on the control plane node:

```bash

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you are the root user, you can run:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### Step 6: Join Worker Nodes to the Cluster

After the control plane initialization, you will receive a `kubeadm join` command with a token. SSH to the worker node and run the `kubeadm join` command as root:

```bash
# Example of the join command received from the control plane node
kubeadm join 192.23.108.8:6443 --token thubfg.2zq20f5ttooz27op --discovery-token-ca-cert-hash sha256:e7755ede5d6f0bae08bca4e13ccce8923995245ca68bba5f7ccc53c9b9728cdb
```


### Step 7: Deploy Pod Network Plugin

Run the following commands on the control plane node to deploy the Flannel network plugin:

```bash
# Download the Flannel YAML manifest
curl -LO https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml

# Open kube-flannel.yml with a text editor
# Find the args section within the kube-flannel container definition It should look like this
args:
  - --ip-masq
  - --kube-subnet-mgr
#Add the additional argument `- --iface=enp1s0` to the existing list of arguments
# Apply the modified manifest
kubectl apply -f kube-flannel.yml
```

### Step 8: Verify the Cluster Status

Run the following command on the control plane node to check the status of the nodes:

```bash
kubectl get nodes
```

You should see both the control plane and worker nodes with a status of "Ready."

Your Kubernetes cluster is now set up and ready to use! For more information on

 deploying applications and other Kubernetes features, visit the official Kubernetes documentation. Happy clustering!
