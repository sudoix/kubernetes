![Setup_HA_Kubernetes_Cluster_with_keepalived](/assets/Setup_HA_Kubernetes_Cluster_with_keepalived.png)

# Set up a Highly Available Kubernetes Cluster using kubeadm
Follow this documentation to set up a highly available Kubernetes cluster using __Ubuntu 20.04 LTS__ with keepalived and haproxy

This documentation guides you in setting up a cluster with three master nodes, one worker node and two load balancer node using HAProxy and Keepalived.

### Virtual IP managed by Keepalived on the load balancer nodes
|Virtual IP|
|----|
|172.16.100.100|


## Set up load balancer nodes (lb1 & lb2)
##### Install Keepalived & Haproxy
```
apt update && apt install -y keepalived haproxy
```
##### configure keepalived
On both nodes create the health check script /etc/keepalived/check_apiserver.sh
```
vim /etc/keepalived/check_apiserver.sh

#!/bin/sh

errorExit() {
  echo "*** $@" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 172.16.100.100; then
  curl --silent --max-time 2 --insecure https://172.16.100.100:6443/ -o /dev/null || errorExit "Error GET https://172.16.100.100:6443/"
fi

chmod +x /etc/keepalived/check_apiserver.sh
```

Create keepalived config /etc/keepalived/keepalived.conf

```
vim /etc/keepalived/keepalived.conf

vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3 # check api server every 3 seconds
  timeout 10 # timeout second if api server doesn't answered
  fall 5 # failed time
  rise 2 # success 2 times
  weight -2 # if failed is done it reduce 2 of the weight
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth1 # set your interface
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        172.16.100.100
    }
    track_script {
        check_apiserver
    }
}
```
##### Enable & start keepalived service
```
systemctl enable --now keepalived
```

##### Configure haproxy
Update **/etc/haproxy/haproxy.cfg**

```
vim /etc/haproxy/haproxy.cfg

frontend kubernetes-frontend
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-backend

backend kubernetes-backend
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
    server k8s-test-master1 172.16.100.11:6443 check fall 3 rise 2
    server k8s-test-master2 172.16.100.21:6443 check fall 3 rise 2
    server k8s-test-master3 172.16.100.31:6443 check fall 3 rise 2

```
##### Enable & restart haproxy service
```
systemctl enable haproxy && systemctl restart haproxy
```

#### Add virtual ip to all of the `/etc/hosts` nodes like this 

```bash 
172.16.100.11 k8s-test-master1 k8s-test-master1.sudoix.com
172.16.100.21 k8s-test-master2 k8s-test-master2.sudoix.com
172.16.100.31 k8s-test-master3 k8s-test-master3.sudoix.com
172.16.100.41 k8s-test-worker1 k8s-test-worker1.sudoix.com
172.16.100.81 lb1-test lb1-test.sudoix.com
172.16.100.91 lb2-test lb2-test.sudoix.com
172.16.100.100 espenu.sudoix.com
```
##### Verify 

Run `watch kubectl get node ` on one node and down one ha proxy or master nodes.

enjoy :)