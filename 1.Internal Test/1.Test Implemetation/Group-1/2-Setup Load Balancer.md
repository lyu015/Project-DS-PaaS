# **Setup Load Balancer**

## **Environment**

- VIP : 169.44.128.162
- LB-1 : 169.44.182.214
- LB-2 : 169.44.182.199


0. Check 

        cat /etc/hosts

1. install Keepalived
- Install

        sudo yum install keepalived

- Configuration

        cat /etc/keepalived/keepalived.conf
        vi /etc/keepalived/keepalived.conf

- Set LB-1

        vrrp_instance VI_1 {
            state MASTER
            interface eth0
            virtual_router_id 51
            priority 110
            advert_int 1
            authentication {
                auth_type PASS
                auth_pass 1111
            }
            virtual_ipaddress {
                169.44.128.162
            }
        }

- Set LB-1

        vrrp_instance VI_2 {
            state MASTER
            interface eth0
            virtual_router_id 51
            priority 100
            advert_int 1
            authentication {
                auth_type PASS
                auth_pass 1111
            }
            virtual_ipaddress {
                169.44.128.162
            }
        }

2 Start keepalived
- for both nodes

        systemctl stop keepalived
        systemctl start keepalived
        systemctl status keepalived


3. Install HAProxy 
- Install

        yum install haproxy
 
- Configuration

        cat /etc/haproxy/haproxy.cfg
        vi /etc/haproxy/haproxy.cfg




 


