# **Basic K8s Cluster installation**

## **Target Environment**

- Architecture
    - No. of Control Plane : 1
    - No. of Worker Nodes : 2

- Softwares:
    - Container Runtime : **containerd**
    - Container Network Interface : **weave net**
    - O/S : CentOS 7.7 - SECDSPaaS!

- Resource 
    - Provider : **ibm cloud**
    - Location : **Sanjose 3**
    - Public Network : 169.44.128.160/28
    - Type : **Classic Public Virtual Machine**
    - size : 2 vCPU / 4 GB ram

    - VM 1 : **Control Plane**
        - IP: 169.44.128.169
        - Hostname : YML-01
    - VM 2 : **Worker Node 1**
        - IP: 169.44.128.167
        - Hostname : YML-02
    - VM 3 : **Worker Node 2**
        - IP: 169.44.128.174
        - Hostname : YML-03
        
## **Installation**

### **Install Container Runtime (containerd)**

- Ref: https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd

1. Install and configure prerequisites:

        cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
        overlay
        br_netfilter
        EOF

        sudo modprobe overlay
        sudo modprobe br_netfilter

        # Setup required sysctl params, these persist across reboots.
        cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
        net.bridge.bridge-nf-call-iptables  = 1
        net.ipv4.ip_forward                 = 1
        net.bridge.bridge-nf-call-ip6tables = 1
        EOF

        # Apply sysctl params without reboot
        sudo sysctl --system

2. Install containerd: (for CentOS/RHEL 7.4+)

        # (Install containerd)
        ## Set up the repository
        ### Install required packages
        sudo yum install -y yum-utils device-mapper-persistent-data lvm2

        ## Add docker repository
        sudo yum-config-manager \
            --add-repo \
            https://download.docker.com/linux/centos/docker-ce.repo

        ## Install containerd
        sudo yum update -y && sudo yum install -y containerd.io

        ## Configure containerd
        sudo mkdir -p /etc/containerd
        containerd config default | sudo tee /etc/containerd/config.toml

        # Restart containerd
        sudo systemctl restart containerd

3. systemd

    To use the systemd cgroup driver in **/etc/containerd/config.toml** with **runc**, set

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
        ...
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true

    When using kubeadm, manually configure the cgroup driver for kubelet.  

### **Install kubeadm**

- ref : https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/


1. Letting iptables see bridged traffic
    
    Make sure that the br_netfilter module is loaded. This can be done by running lsmod | grep br_netfilter. To load it explicitly call sudo modprobe br_netfilter.

    As a requirement for your Linux Node's iptables to correctly see bridged traffic, you should ensure net.bridge.bridge-nf-call-iptables is set to 1 in your sysctl config, e.g.

        X cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
        X br_netfilter
        X EOF

        cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
        EOF

        sudo sysctl --system

2. Installing kubeadm, kubelet and kubectl : (for CentOS, RHEL or Fedora)

            cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
        [kubernetes]
        name=Kubernetes
        baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
        enabled=1
        gpgcheck=1
        repo_gpgcheck=1
        gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
        exclude=kubelet kubeadm kubectl
        EOF

        # Set SELinux in permissive mode (effectively disabling it)
        sudo setenforce 0
        sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

        sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

        sudo systemctl enable --now kubelet


3. Don't forget **Swapoff**

        sudo swapoff -a
        sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

### **Creating a cluster with kubeadm**

- **For Control Plane Only**
1. kubeadm init

        sudo kubeadm init --pod-network-cidr 192.168.0.0/16

    Then, Your Kubernetes control-plane has initialized successfully!
2. To start using your cluster, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

3. You should now deploy a Pod network to the cluster.
- ref : https://www.weave.works/docs/net/latest/kubernetes/kube-addon/

        kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

- **For Worker Node**

4. Join worker nodes to the cluster

        kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

### **Check the cluster**
1. Check the nodes

        kubectl get nodes

2. Check the pods in the kube-systemp namespace

        kubectl get pods -o wide --all-namespaces
        
# **Done**