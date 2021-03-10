# **Common Setting**

## **Create K8s admin user**

1. OS root Password Change

        passwd

    - PassWord : ********

2. OS User : k8sadm

        adduser k8sadm
        passwd k8sadm

    - PassWord : ********

3. to update /etc/sudoers


        chmod u+w /etc/sudoers
        vi /etc/sudoers


4. add at 101 line 

        ":101"
        k8sadm  ALL=(ALL)       ALL
        ":wq"

5. finish update and login as "k8sadm"

        chmod u-w /etc/sudoers
        su - k8sadm


## **Update CentOS version from 7.7 to 7.8**

later...

## **VM Resource Information**


- etcd-01.Youngmin-Lee-s-Account.cloud - 169.44.182.201, 10.168.40.30		
- etcd-02.Youngmin-Lee-s-Account.cloud - 169.44.182.196, 10.168.40.39	
- etcd-03.Youngmin-Lee-s-Account.cloud - 169.44.182.195, 10.168.40.37	

- etcd-1.YML4953.cloud - 169.44.182.212, 10.168.40.19		
- etcd-2.YML4953.cloud - 169.44.182.204, 10.168.40.9		
- etcd-3.YML4953.cloud - 169.44.182.206, 10.168.40.58	

- LB-01.Youngmin-Lee-s-Account.cloud - 169.44.128.164, 10.168.40.11	
