# Kubernetes Cluster Setup (v1.26) on Ubuntu VM
 Used vagrant scripts to provision VM refered kodekloud CKA instructions.(1 master + 2 workers node)

  **Execute below commands on all nodes:**
 

    sudo apt-get update && apt-get upgrade -y

 
 ## Load the Kernel modules on all the nodes
    sudo tee /etc/modules-load.d/containerd.conf <<EOF
    overlay
    br_netfilter
    EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter

## Set the following Kernel params for K8s
    sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF

##### **Reload the system changes**
    sudo sysctl --system

## Install containerd run time
    sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
 
##### **Enable docker repository**
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
##### **Run apt cmd to install containerd**
    sudo apt update
    sudo apt install -y containerd.io
##### **Configure containerd using systemd as cgroup**
    containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
    sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
##### **Restart and enable containerd service**
    sudo systemctl restart containerd
    sudo systemctl enable containerd
## Add apt repository for k8s
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
## Install K8s components kubectl, kubeadm & kubelet
    sudo apt update
    sudo apt install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl

    sudo apt-get install -y jq
    local_ip="$(ip --json a s | jq -r '.[] | if .ifname == "eth1" then .addr_info[] | if .family == "inet" then .local else empty end else empty end')"

    cat > /etc/default/kubelet << EOF
    KUBELET_EXTRA_ARGS=--node-ip=$local_ip
    EO
 
  ## Initialize K8s cluster with Kubeadm
    
    IPADDR="192.168.56.2"
    NODENAME=$(hostname -s)
    POD_CIDR="10.244.0.0/16"
   

    sudo kubeadm init --apiserver-advertise-address=$IPADDR --apiserver-cert-extra-sans=$IPADDR --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap

## Run below commands as a non-root user in master node
**please note down join command from previous output as we need it to execute it on worker node**

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    kubectl cluster-info
    kubectl get nodes

    ** output shows master node is not ready. we need to install network addon for the same. **

## Execute below commands in all worker node:
    sudo kubeadm join ipaddr:6443 --token vt4ua6.wcma2y8pl4menxh2 --discovery-token-ca-cert-hash sha256:0494aa7fc6ced8f8e7b20137ec0c5d2699dc5f8e616656932ff9173c94962a36

## Install Pod Network addon:
    curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
    kubectl apply -f calico.yaml
## Verify the status of nodes
    kubectl get nodes
## Verify the status of pods:
    kubectl get po -n kube-system

