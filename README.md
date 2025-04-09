# 1) Set Host Name and Update hosts file
```bash
sudo hostnamectl set-hostname "master0"  // Master Node  
sudo hostnamectl set-hostname "worker0" // Worker Node 1  
sudo hostnamectl set-hostname "worker1" // Worker Node 2  
```


add the following lines to /etc/hosts file on each node  
```
sudo vim /etc/hosts
```

```
192.168.0.11  master0  
192.168.0.17  worker0  
192.168.0.14  worker1  
192.168.0.16  worker2  
192.168.0.13  worker3  
192.168.0.15  worker4
```
-> IPv6 아래에 넣지 말라.  


# 2) Disable Swap and Load Kernel Modules
```
sudo swapoff -a && sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab  
```

### Now, load the following kernel modules using modprobe command.
```
sudo modprobe overlay && sudo modprobe br_netfilter  
```

### For the permanent loading of these modules, create the file with following content.
```
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

### Next, add the kernel parameters like IP forwarding. Create a file and load the parameters using sysctl command,
```
sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOT
```

### To load the above kernel parameters, run
```
sudo sysctl --system  
```

# 3) Install Containerd
### First install containerd dependencies,
```
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates  
```

### Next, add containerd repository using following set of commands.
```
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg  | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/containerd.gpg  
```
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"  
```
```
sudo apt update && sudo apt install containerd.io -y  
```

### Next, configure containerd so that it starts using Systemdgroup. Run beneath commands.
```
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1  
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml  
```
### Restart containerd service so that above changes come into the affect.
```
sudo systemctl restart containerd  

sudo systemctl status containerd  
```

# 4) Add Kubernetes Package Repository
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/k8s.gpg  
echo 'deb [signed-by=/etc/apt/keyrings/k8s.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/k8s.list  
```

# 5) Install Kubernetes Components
```
sudo apt update  
```

```
sudo apt install kubelet kubeadm kubectl -y  
```

# 6) Initialize Kubernetes Cluster - ONLY MASTER
```
sudo kubeadm init --control-plane-endpoint=master0  
```

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
sudo chown $(id -u):$(id -g) $HOME/.kube/config  
```

# 7) Install Calico Network Plugin - ONLY MASTER
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/calico.yaml  
```

# 8) Join Kubernetes Cluster - ONLY WORKER
```
sudo su  
```

```
sudo kubeadm join master0:6443 --token hh7mxv.j3uhkvw14gzci7c3 \  
	--discovery-token-ca-cert-hash sha256:9ca24870ce3c5ccbc1cd069f6d1fef2514b0199b6048f67b6a24148fc0cb023d  
```
