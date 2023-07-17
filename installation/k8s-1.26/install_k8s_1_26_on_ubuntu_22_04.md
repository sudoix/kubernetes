# Install kubernetes 1.26 on ubuntu 22.04

![Install kubernetes](/assets/Install_kubernetes.png)

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
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
apt install -y kubeadm=1.26.0-00 kubelet=1.26.0-00 kubectl=1.26.0-00
```

Update `/etc/hosts/`

```
cat >>/etc/hosts<<EOF
172.16.0.100   km1.sudoix.com    km1
172.16.0.200   kw1.sudoix.com    kw1
EOF
```


https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
