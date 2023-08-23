![Setup_HA_Kubernetes_Cluster](/assets/Setup_HA_Kubernetes_Cluster.png)


# Set up a Highly Available Kubernetes Cluster

## Before you start you should follow these step

[Preinstall of k8s cluster](01-install_k8s_1_26_on_ubuntu_22_04.md)

[Init cluster](01-master_node.md)

## Set up load balancer node
##### Install Haproxy

```
apt update && apt install -y haproxy
```
##### Configure haproxy
Append the below lines to **/etc/haproxy/haproxy.cfg**
```
frontend kubernetes-frontend
    bind <PUBLIC_IP OR PRIVATE_IP>:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server km1 172.16.0.100:6443 check fall 3 rise 2
    server km2 172.16.0.105:6443 check fall 3 rise 2
```
##### Restart haproxy service
```
systemctl restart haproxy
```

## On any one of the Kubernetes master node (Eg: km1)
##### Initialize Kubernetes Cluster
```
kubeadm init --control-plane-endpoint="<PUBLIC_IP OR PRIVATE_IP>:6443" --upload-certs --apiserver-advertise-address=172.16.0.100 --pod-network-cidr=192.168.0.0/16
```
Copy the commands to join other master nodes and worker nodes.
##### Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f "https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml"
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f "https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml"
```

## Join other nodes to the cluster (km2 & kw1)
> Use the respective kubeadm join commands you copied from the output of kubeadm init command on the first master.

> IMPORTANT: You also need to pass --apiserver-advertise-address to the join command when you join the other master node.

Password for root account is kubeadmin (if you used my Vagrant setup)

## Verifying the cluster
```
kubectl cluster-info
kubectl get nodes
kubectl get cs
```
