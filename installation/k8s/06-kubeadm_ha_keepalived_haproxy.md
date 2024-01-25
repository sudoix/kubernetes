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
    server km1 172.16.0.100:6443 check fall 3 rise 2
    server km2 172.16.0.105:6443 check fall 3 rise 2

```
##### Enable & restart haproxy service
```
systemctl enable haproxy && systemctl restart haproxy
```
## Pre-requisites on all kubernetes nodes (masters & workers)
##### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
##### Disable Firewall
```
systemctl disable --now ufw
```
##### Enable and Load Kernel modules
```
{
cat >> /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
}
```
##### Add Kernel settings
```
{
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
}
```
##### Install containerd runtime
```
{
  apt update
  apt install -y containerd apt-transport-https
  mkdir /etc/containerd
  containerd config default > /etc/containerd/config.toml
  systemctl restart containerd
  systemctl enable containerd
}
```
##### Add apt repo for kubernetes
```
{
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
}
```
##### Install Kubernetes components
```
{
  apt update
  apt install -y kubeadm=1.26.0-00 kubelet=1.26.0-00 kubectl=1.26.0-00
}
```


##### Initialize Kubernetes Cluster on one of master node

```
kubeadm init --control-plane-endpoint="KEPALIVE_IP:6443" --upload-certs --apiserver-advertise-address=MASTER_IP --pod-network-cidr=192.168.0.0/16
```

###### Copy the commands to join other master nodes and worker nodes.

## Join other master nodes to the cluster
> Use the respective kubeadm join commands you copied from the output of kubeadm init command on the first master.

> IMPORTANT: Don't forget the `--apiserver-advertise-address` option to the join command when you join the other master nodes.

## Join worker nodes to the cluster
> Use the kubeadm join command you copied from the output of kubeadm init command on the first master

##### Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f "https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml"
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f "https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml"
```



## Downloading kube config to your local machine
On your host machine

## Verifying the cluster


```
kubectl cluster-info
kubectl get nodes
```


https://www.ibm.com/docs/en/storage-ceph/5?topic=prerequisites-installing-configuring-keepalived

https://tecadmin.net/setup-ip-failover-on-ubuntu-with-keepalived/