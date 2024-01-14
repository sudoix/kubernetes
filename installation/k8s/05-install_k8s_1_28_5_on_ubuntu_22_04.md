# Install kubernetes 1.26 on ubuntu 22.04

![Install kubernetes](/assets/install_kubernetes.png)

These steps should be run on all servers (master and worker)

disable swap on all server

```

vim /etc/fstab --> delete all swap line
swapoff -a

swapon -s --> for verify
```

Disable firewall (ufw)

```
systemctl disable --now ufw
```

Add kernel modules

```
vim /etc/modules-load.d/containerd.conf --> add
overlay
br_netfilter
```

```
modprobe overlay
modprobe br_netfilter
```

Kernel setting

```
vim /etc/sysctl.d/kubernetes.conf

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
```

```
sysctl --system
```

Install containerd

```
apt update
apt install -y ca-certificates curl gnupg lsb-release
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list

apt update
apt install -y containerd.io
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
```

Add repository for kubernetes

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt install -y kubeadm=1.28.5-1.1 kubelet=1.28.5-1.1 kubectl=1.28.5-1.1
```

Pull image

```
kubeadm config images pull
```

Init cluster (Use this for cluster without HAProxy)

```
kubeadm init --control-plane-endpoint YOURDOMAINADDRESS --apiserver-advertise-address=YOURDOMAINADDRESS --pod-network-cidr=192.168.0.0/16 >> /root/kubeinit.log
```

Update `/etc/hosts/`

```
cat >>/etc/hosts<<EOF
172.16.0.100   km1.sudoix.com    km1
172.16.0.200   kw1.sudoix.com    kw1
YOURLOADBALANCER OR YOUR CONTROLPLAIN NODE     YOUR DOMAIN 
172.16.0.250   espenu.sudoix.com
EOF
```

https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
