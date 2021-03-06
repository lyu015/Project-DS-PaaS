# **Set Up External etcd Cluster**
- Reference : http://play.etcd.io/install

## **Install and deploy etcd**
- Version : 3.4.0- Recommended System Requirement : 8 vCPU, 16GB RAM, 50GB SSD GCE instances
- Test Environment :
    - 2 vCPU, 4GB RAM
    - 3 etcd nodes

### **Define Environment** 

1. Define Environment-  etcd node 1
- etcd name : **etcd1**
    - etcd data directory : **/tmp/etcd/s1**
    - IP address : ***169.44.182.212***
    - client port : 2379
    - peer port : 2380
    - cluster token : tkn
    - cluster state : new
    - etcd certs directory : ${HOME}/certs
    - client trusted root CA : **etcd-root-ca.pem**
    - client cert (public) : **etcd1.pem**
    - client key (public) : **etcd1-key.pem**
    - peer trusted root CA : **etcd-root-ca.pem**
    - peer cert (public) : **etcd1.pem**
    - peer key (public) : **etcd1-key.pem**
- etcd node 2
    - etcd name : **etcd2**
    - etcd data directory : **/tmp/etcd/s2**
    - IP address : ***169.44.182.204***
    - client port : 2379
    - peer port : 2380
    - cluster token : tkn
    - cluster state : new
    - etcd certs directory : ${HOME}/certs
    - client trusted root CA : **etcd-root-ca.pem**
    - client cert (public) : **etcd1.pem**
    - client key (public) : **etcd1-key.pem**
    - peer trusted root CA : **etcd-root-ca.pem**
    - peer cert (public) : **etcd2.pem**
    - peer key (public) : **etcd2-key.pem**
- etcd node 3
    - etcd name : **etcd3**
    - etcd data directory : **/tmp/etcd/s3**
    - IP address : ***169.44.182.206***
    - client port : 2379
    - peer port : 2380
    - cluster token : tkn
    - cluster state : new
    - etcd certs directory : ${HOME}/certs
    - client trusted root CA : **etcd-root-ca.pem**
    - client cert (public) : **etcd3.pem**
    - client key (public) : **etcd3-key.pem**
    - peer trusted root CA : **etcd-root-ca.pem**
    - peer cert (public) : **etcd3.pem**
    - peer key (public) : **etcd3-key.pem**

### **Generate Certification Files**

2. Prepare to Generate Certification Files
    
- cfssl setting   
    - Host : CA server ? or etcd 1 ?
    - cfssl version : R1.2
    - cfssl exec dir : /usr/local/bin
    - source certs dir : /tmp/certs
- Install cfssl

        rm -f /tmp/cfssl* && rm -rf /tmp/certs && mkdir -p /tmp/certs

        curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /tmp/cfssl
        chmod +x /tmp/cfssl
        sudo mv /tmp/cfssl /usr/local/bin/cfssl

        curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /tmp/cfssljson
        chmod +x /tmp/cfssljson
        sudo mv /tmp/cfssljson /usr/local/bin/cfssljson

        /usr/local/bin/cfssl version
        /usr/local/bin/cfssljson -h

        mkdir -p /tmp/certs

3. Step 3: Generate self-signed root CA certificate

- Root CA certifiacte setting
    - Organization : **etcd**
    - Organization Unit : **etcd Security**
    - City : **San Fransisco**
    - State : **California**
    - Country : **USA**
    - Key Algorithm : rsa
    - Key Size : 2048
    - Key Expiration (hours) : 87600 (10years ?)
    - Common Name : **etcd-root-ca**
- Generate 

        mkdir -p /tmp/certs

        cat > /tmp/certs/etcd-root-ca-csr.json <<EOF
        {
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
            "O": "etcd",
            "OU": "etcd Security",
            "L": "San Francisco",
            "ST": "California",
            "C": "USA"
            }
        ],
        "CN": "etcd-root-ca"
        }
        EOF

        cfssl gencert --initca=true /tmp/certs/etcd-root-ca-csr.json | cfssljson --bare /tmp/certs/etcd-root-ca

- verify 
    
        openssl x509 -in /tmp/certs/etcd-root-ca.pem -text -noout


- cert-generation configuration       
        
        cat > /tmp/certs/etcd-gencert.json <<EOF
        {
        "signing": {
            "default": {
                "usages": [
                "signing",
                "key encipherment",
                "server auth",
                "client auth"
                ],
                "expiry": "87600h"
            }
        }
        }
        EOF

- Result
    - CSR configuration : /tmp/certs/etcd-root-ca-csr.json
    - CSR : /tmp/certs/etcd-root-ca.csr
    - self-signed root CA public key : /tmp/certs/etcd-root-ca.pem
    - self-signed root CA private key : /tmp/certs/etcd-root-ca-key.pem
    - cert-generation configuration for other TLS assets : /tmp/certs/etcd-gencert.json

4. Step 4: Generate local-issued certificates with private keys
        
- For etcd1

        mkdir -p /tmp/certs

        cat > /tmp/certs/etcd1-ca-csr.json <<EOF
        {
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
            "O": "etcd",
            "OU": "etcd Security",
            "L": "San Francisco",
            "ST": "California",
            "C": "USA"
            }
        ],
        "CN": "etcd1",
        "hosts": [
            "127.0.0.1",
            "localhost",
            "169.44.182.212"
        ]
        }
        EOF

        cfssl gencert \
        --ca /tmp/certs/etcd-root-ca.pem \
        --ca-key /tmp/certs/etcd-root-ca-key.pem \
        --config /tmp/certs/etcd-gencert.json \
        /tmp/certs/etcd1-ca-csr.json | cfssljson --bare /tmp/certs/etcd1

- Verify key for etcd1

        openssl x509 -in /tmp/certs/etcd1.pem -text -noout

- Copy Certification Files to **etcd node 1** (**169.44.182.212**)

        scp /tmp/certs/etcd-root-ca.pem /tmp/certs/etcd1.pem /tmp/certs/etcd1-key.pem root@169.44.182.212:~/


- For etcd2

        mkdir -p /tmp/certs

        cat > /tmp/certs/etcd2-ca-csr.json <<EOF
        {
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
            "O": "etcd",
            "OU": "etcd Security",
            "L": "San Francisco",
            "ST": "California",
            "C": "USA"
            }
        ],
        "CN": "etcd2",
        "hosts": [
            "127.0.0.1",
            "localhost",
            "169.44.182.204"
        ]
        }
        EOF
        cfssl gencert \
        --ca /tmp/certs/etcd-root-ca.pem \
        --ca-key /tmp/certs/etcd-root-ca-key.pem \
        --config /tmp/certs/etcd-gencert.json \
        /tmp/certs/etcd2-ca-csr.json | cfssljson --bare /tmp/certs/etcd2

- Verify key for etcd2

        openssl x509 -in /tmp/certs/etcd2.pem -text -noout

- Copy Certification Files to **etcd node 2** (**169.44.182.204**)

        scp /tmp/certs/etcd-root-ca.pem /tmp/certs/etcd1.pem /tmp/certs/etcd1-key.pem root@169.44.182.204:~/

- For etcd3

        mkdir -p /tmp/certs

        cat > /tmp/certs/etcd3-ca-csr.json <<EOF
        {
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
            "O": "etcd",
            "OU": "etcd Security",
            "L": "San Francisco",
            "ST": "California",
            "C": "USA"
            }
        ],
        "CN": "etcd3",
        "hosts": [
            "127.0.0.1",
            "localhost",
            "169.44.182.206"
        ]
        }
        EOF
        cfssl gencert \
        --ca /tmp/certs/etcd-root-ca.pem \
        --ca-key /tmp/certs/etcd-root-ca-key.pem \
        --config /tmp/certs/etcd-gencert.json \
        /tmp/certs/etcd3-ca-csr.json | cfssljson --bare /tmp/certs/etcd3

- Verify key for etcd3

        openssl x509 -in /tmp/certs/etcd3.pem -text -noout

- Copy Certification Files to **etcd node 1** (**169.44.182.206**)

        scp /tmp/certs/etcd-root-ca.pem /tmp/certs/etcd1.pem /tmp/certs/etcd1-key.pem root@169.44.182.206:~/

### **Setup etcd nodes**
- etcd
    - etcd version : v3.4.0
    - etcd exec dir : /tmp/test-etcd

5. set up etcd node 1 
- Log in into **169.44.182.212** as the root :

- Move Certification to certs folder

        mkdir -p ${HOME}/certs
        mv /tmp/certs/* ${HOME}/certs

- Declare Parameters

        ETCD_VER=v3.4.0
        
        GITHUB_URL=https://github.com/coreos/etcd/releases/download
        DOWNLOAD_URL=${GITHUB_URL}

- Download etcd files        

        rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
        rm -rf /tmp/test-etcd && mkdir -p /tmp/test-etcd

        curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
        tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/test-etcd --strip-components=1

- Check the downloaded etcd

        /tmp/test-etcd/etcd --version
        ETCDCTL_API=3 /tmp/test-etcd/etcdctl version


- Run with Systemd

        cat > /tmp/etcd1.service <<EOF
        [Unit]
        Description=etcd
        Documentation=https://github.com/coreos/etcd
        Conflicts=etcd.service
        Conflicts=etcd2.service

        [Service]
        Type=notify
        Restart=always
        RestartSec=5s
        LimitNOFILE=40000
        TimeoutStartSec=0

        ExecStart=/tmp/test-etcd/etcd --name etcd1 \
        --data-dir /tmp/etcd/s1 \
        --listen-client-urls https://169.44.182.212:2379 \
        --advertise-client-urls https://169.44.182.212:2379 \
        --listen-peer-urls https://169.44.182.212:2380 \
        --initial-advertise-peer-urls https://169.44.182.212:2380 \
        --initial-cluster etcd1=https://169.44.182.212:2380,etcd2=https://169.44.182.204:2380,etcd3=https://169.44.182.206:2380 \
        --initial-cluster-token tkn \
        --initial-cluster-state new \
        --client-cert-auth \
        --trusted-ca-file ${HOME}/certs/etcd-root-ca.pem \
        --cert-file ${HOME}/certs/etcd1.pem \
        --key-file ${HOME}/certs/etcd1-key.pem \
        --peer-client-cert-auth \
        --peer-trusted-ca-file ${HOME}/certs/etcd-root-ca.pem \
        --peer-cert-file ${HOME}/certs/etcd1.pem \
        --peer-key-file ${HOME}/certs/etcd1-key.pem

        [Install]
        WantedBy=multi-user.target
        EOF

        sudo mv /tmp/etcd1.service /etc/systemd/system/etcd1.service

- to start service

        sudo systemctl daemon-reload
        sudo systemctl cat etcd1.service
        sudo systemctl enable etcd1.service
        sudo systemctl start etcd1.service

- to get logs from service

        sudo systemctl status etcd1.service -l --no-pager
        sudo journalctl -u etcd1.service -l --no-pager|less
        sudo journalctl -f -u etcd1.service

- to stop service

        sudo systemctl stop etcd1.service
        sudo systemctl disable etcd1.service

6. set up etcd node 2
- Log in into **169.44.182.204** as the root :

- Move Certification to certs folder

        mkdir -p ${HOME}/certs
        mv /tmp/certs/* ${HOME}/certs

- Declare Parameters

        ETCD_VER=v3.4.0
        
        GITHUB_URL=https://github.com/coreos/etcd/releases/download
        DOWNLOAD_URL=${GITHUB_URL}

- Download etcd files        

        rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
        rm -rf /tmp/test-etcd && mkdir -p /tmp/test-etcd

        curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
        tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/test-etcd --strip-components=1

- Check the downloaded etcd

        /tmp/test-etcd/etcd --version
        ETCDCTL_API=3 /tmp/test-etcd/etcdctl version


- Run with Systemd

        cat > /tmp/etcd2.service <<EOF
        [Unit]
        Description=etcd
        Documentation=https://github.com/coreos/etcd
        Conflicts=etcd.service
        Conflicts=etcd2.service

        [Service]
        Type=notify
        Restart=always
        RestartSec=5s
        LimitNOFILE=40000
        TimeoutStartSec=0

        ExecStart=/tmp/test-etcd/etcd --name etcd2 \
        --data-dir /tmp/etcd/s2 \
        --listen-client-urls https://169.44.182.204:2379 \
        --advertise-client-urls https://169.44.182.204:2379 \
        --listen-peer-urls https://169.44.182.204:2380 \
        --initial-advertise-peer-urls https://169.44.182.204:2380 \
        --initial-cluster etcd1=https://169.44.182.212:2380,etcd2=https://169.44.182.204:2380,etcd3=https://169.44.182.206:2380 \
        --initial-cluster-token tkn \
        --initial-cluster-state new \
        --client-cert-auth \
        --trusted-ca-file ${HOME}/certs/etcd-root-ca.pem \
        --cert-file ${HOME}/certs/etcd2.pem \
        --key-file ${HOME}/certs/etcd2-key.pem \
        --peer-client-cert-auth \
        --peer-trusted-ca-file ${HOME}/certs/etcd-root-ca.pem \
        --peer-cert-file ${HOME}/certs/etcd2.pem \
        --peer-key-file ${HOME}/certs/etcd2-key.pem

        [Install]
        WantedBy=multi-user.target
        EOF

        sudo mv /tmp/etcd2.service /etc/systemd/system/etcd2.service



- to start service

        sudo systemctl daemon-reload
        sudo systemctl cat etcd2.service
        sudo systemctl enable etcd2.service
        sudo systemctl start etcd2.service

- to get logs from service

        sudo systemctl status etcd2.service -l --no-pager
        sudo journalctl -u etcd2.service -l --no-pager|less
        sudo journalctl -f -u etcd2.service

- to stop service

        sudo systemctl stop etcd2.service
        sudo systemctl disable etcd2.service

7. set up etcd node 3
- Log in into **169.44.182.206** as the root :

- Move Certification to certs folder

        mkdir -p ${HOME}/certs
        mv /tmp/certs/* ${HOME}/certs

- Declare Parameters

        ETCD_VER=v3.4.0
        
        GITHUB_URL=https://github.com/coreos/etcd/releases/download
        DOWNLOAD_URL=${GITHUB_URL}

- Download etcd files        

        rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
        rm -rf /tmp/test-etcd && mkdir -p /tmp/test-etcd

        curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
        tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/test-etcd --strip-components=1

- Check the downloaded etcd

        /tmp/test-etcd/etcd --version
        ETCDCTL_API=3 /tmp/test-etcd/etcdctl version


- Run with Systemd

        cat > /tmp/etcd3.service <<EOF
        [Unit]
        Description=etcd
        Documentation=https://github.com/coreos/etcd
        Conflicts=etcd.service
        Conflicts=etcd2.service

        [Service]
        Type=notify
        Restart=always
        RestartSec=5s
        LimitNOFILE=40000
        TimeoutStartSec=0

        ExecStart=/tmp/test-etcd/etcd --name etcd3 \
        --data-dir /tmp/etcd/s3 \
        --listen-client-urls https://169.44.182.206:2379 \
        --advertise-client-urls https://169.44.182.206:2379 \
        --listen-peer-urls https://169.44.182.206:2380 \
        --initial-advertise-peer-urls https://169.44.182.206:2380 \
        --initial-cluster etcd1=https://169.44.182.212:2380,etcd2=https://169.44.182.204:2380,etcd3=https://169.44.182.206:2380 \
        --initial-cluster-token tkn \
        --initial-cluster-state new \
        --client-cert-auth \
        --trusted-ca-file ${HOME}/certs/etcd-root-ca.pem \
        --cert-file ${HOME}/certs/etcd3.pem \
        --key-file ${HOME}/certs/etcd3-key.pem \
        --peer-client-cert-auth \
        --peer-trusted-ca-file ${HOME}/certs/etcd-root-ca.pem \
        --peer-cert-file ${HOME}/certs/etcd3.pem \
        --peer-key-file ${HOME}/certs/etcd3-key.pem

        [Install]
        WantedBy=multi-user.target
        EOF

        sudo mv /tmp/etcd3.service /etc/systemd/system/etcd3.service

- to start service

        sudo systemctl daemon-reload
        sudo systemctl cat etcd3.service
        sudo systemctl enable etcd3.service
        sudo systemctl start etcd3.service

- to get logs from service

        sudo systemctl status etcd3.service -l --no-pager
        sudo journalctl -u etcd3.service -l --no-pager|less
        sudo journalctl -f -u etcd3.service

- to stop service

        sudo systemctl stop etcd3.service
        sudo systemctl disable etcd3.service

8. Check etcd status : 
- At etcd node 1

        ETCDCTL_API=3 /tmp/test-etcd/etcdctl \
        --endpoints 169.44.182.212:2379,169.44.182.204:2379,169.44.182.206:2379 \
        --cacert ${HOME}/certs/etcd-root-ca.pem \
        --cert ${HOME}/certs/etcd1.pem \
        --key ${HOME}/certs/etcd1-key.pem \
        endpoint health
        
- At etcd node 2

        ETCDCTL_API=3 /tmp/test-etcd/etcdctl \
        --endpoints 169.44.182.212:2379,169.44.182.204:2379,169.44.182.206:2379 \
        --cacert ${HOME}/certs/etcd-root-ca.pem \
        --cert ${HOME}/certs/etcd2.pem \
        --key ${HOME}/certs/etcd2-key.pem \
        endpoint health
        
- At etcd node 3

        ETCDCTL_API=3 /tmp/test-etcd/etcdctl \
        --endpoints 169.44.182.212:2379,169.44.182.204:2379,169.44.182.206:2379 \
        --cacert ${HOME}/certs/etcd-root-ca.pem \
        --cert ${HOME}/certs/etcd3.pem \
        --key ${HOME}/certs/etcd3-key.pem \
        endpoint health
        
9. Test etcd
- put value at the node 1

        ETCDCTL_API=3 /tmp/test-etcd/etcdctl \
        --endpoints 169.44.182.212:2379  \
        --cacert ${HOME}/certs/etcd-root-ca.pem \
        --cert ${HOME}/certs/etcd1.pem \
        --key ${HOME}/certs/etcd1-key.pem \
        put name ymlee

- get value at the node 1

        ETCDCTL_API=3 /tmp/test-etcd/etcdctl \
        --endpoints 169.44.182.212:2379  \
        --cacert ${HOME}/certs/etcd-root-ca.pem \
        --cert ${HOME}/certs/etcd1.pem \
        --key ${HOME}/certs/etcd1-key.pem \
        get name

- get value at the node 2

        ETCDCTL_API=3 /tmp/test-etcd/etcdctl \
        --endpoints 169.44.182.206:2379  \
        --cacert ${HOME}/certs/etcd-root-ca.pem \
        --cert ${HOME}/certs/etcd2.pem \
        --key ${HOME}/certs/etcd2-key.pem \
        get name

- get value at the node 3

        ETCDCTL_API=3 /tmp/test-etcd/etcdctl \
        --endpoints 169.44.182.204:2379  \
        --cacert ${HOME}/certs/etcd-root-ca.pem \
        --cert ${HOME}/certs/etcd3.pem \
        --key ${HOME}/certs/etcd3-key.pem \
        get name


# **Done**



