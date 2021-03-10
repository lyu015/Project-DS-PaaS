# **Set Up External etcd**
## **Set up a High Availability etcd cluster with kubeadm**

- Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/



### **Preparation**
- provision 3 hosts for 3-nodes etcd cluster
    - etcd-01 : 169.44.182.201	10.168.40.30
    - etcd-02 : 169.44.182.196	10.168.40.39
    - etcd-03 : 169.44.182.195	10.168.40.37

- install docker, kubelet and kubeadm
    - Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#letting-iptables-see-bridged-traffic   
    
    **1. Installing Docker Runtime**


        # (Install Docker CE)
        ## Set up the repository
        ### Install required packages
        
        sudo yum install -y yum-utils device-mapper-persistent-data lvm2

        ## Add the Docker repository
        sudo yum-config-manager --add-repo \
                    https://download.docker.com/linux/centos/docker-ce.repo
        
        # Install Docker CE
        sudo yum update -y && sudo yum install -y \
        containerd.io-1.2.13 \
        docker-ce-19.03.11 \
        docker-ce-cli-19.03.11
        
        ## Create /etc/docker
        sudo mkdir /etc/docker
        
        # Set up the Docker daemon
        cat <<EOF | sudo tee /etc/docker/daemon.json
        {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
            "max-size": "100m"
        },
        "storage-driver": "overlay2",
        "storage-opts": [
            "overlay2.override_kernel_check=true"
        ]
        }
        EOF
        
        # Create /etc/systemd/system/docker.service.d
        sudo mkdir -p /etc/systemd/system/docker.service.d
        
        # Restart Docker
        sudo systemctl daemon-reload
        sudo systemctl restart docker
        
        If you want the docker service to start on boot, run the following command:

        sudo systemctl enable docker

    **2. Letting iptables see bridged traffic**
    - Make sure that the br_netfilter module is loaded. This can be done by running lsmod | grep br_netfilter. To load it explicitly call sudo modprobe br_netfilter.
    - As a requirement for your Linux Node's iptables to correctly see bridged traffic, you should ensure net.bridge.bridge-nf-call-iptables is set to 1 in your sysctl config, e.g.

            cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
            br_netfilter
            EOF
      
            cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
            net.bridge.bridge-nf-call-ip6tables = 1
            net.bridge.bridge-nf-call-iptables = 1
            EOF
            sudo sysctl --system
    
    **3. Install kubeadm, kubelet and kubectl**

        sudo swapoff -a
        sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab


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

    **4. Set SELinux in permissive mode (effectively disabling it)**


        sudo setenforce 0
        sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

        sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

        sudo systemctl enable --now kubelet


    **5. Setting up the cluster**
    
    - There is error in the document. This directory must be created before following tasks, or kubelet won't run
    - as the root user
    
            sudo mkdir -p /etc/systemd/system/kubelet.service.d

    - Configure the kubelet to be a service manager for etcd.
    - as the root user

            cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
            [Service]
            ExecStart=
            #  Replace "systemd" with the cgroup driver of your container runtime. The default value in the kubelet is "cgroupfs".
            ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd
            Restart=always
            EOF
    - and then,

            systemctl daemon-reload
            systemctl restart kubelet

    - Check the kubelet status to ensure it is running.

            systemctl status kubelet

    **6. Create configuration files for kubeadm.**
    
    - Update HOST0, HOST1, and HOST2 with the IPs or resolvable names of your hosts
    -   private IP's of etcd nodes,


            export HOST0=10.168.40.30
            export HOST1=10.168.40.39
            export HOST2=10.168.40.37
    
    - Create temp directories to store files that will end up on other hosts.

            mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

            ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
            NAMES=("infra0" "infra1" "infra2")

            for i in "${!ETCDHOSTS[@]}"; do
            HOST=${ETCDHOSTS[$i]}
            NAME=${NAMES[$i]}
            cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
            apiVersion: "kubeadm.k8s.io/v1beta2"
            kind: ClusterConfiguration
            etcd:
                local:
                    serverCertSANs:
                    - "${HOST}"
                    peerCertSANs:
                    - "${HOST}"
                    extraArgs:
                        initial-cluster: ${NAMES[0]}=https://${ETCDHOSTS[0]}:2380,${NAMES[1]}=https://${ETCDHOSTS[1]}:2380,${NAMES[2]}=https://${ETCDHOSTS[2]}:2380
                        initial-cluster-state: new
                        name: ${NAME}
                        listen-peer-urls: https://${HOST}:2380
                        listen-client-urls: https://${HOST}:2379
                        advertise-client-urls: https://${HOST}:2379
                        initial-advertise-peer-urls: https://${HOST}:2380
            EOF
            done
