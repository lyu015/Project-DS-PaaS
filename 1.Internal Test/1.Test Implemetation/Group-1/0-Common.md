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