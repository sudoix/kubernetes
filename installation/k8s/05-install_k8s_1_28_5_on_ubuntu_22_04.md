# Install kubernetes 1.28  on ubuntu 22.04

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

Update `/etc/hosts/`

```
cat >>/etc/hosts<<EOF
172.16.0.100   km1.sudoix.com    km1
172.16.0.200   kw1.sudoix.com    kw1
YOURLOADBALANCER OR YOUR CONTROLPLAIN NODE     YOUR DOMAIN 
172.16.0.250   espenu.sudoix.com
EOF
```

Init cluster (Use this for cluster without HAProxy)

```
kubeadm init --control-plane-endpoint YOURDOMAINADDRESS --apiserver-advertise-address=YOURDOMAINADDRESS --pod-network-cidr=192.168.0.0/16 >> /root/kubeinit.log
```


### Deploy calico network 

```bash 
kubectl create -f "https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml"

kubectl create -f "https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml"
```

## Join master and worker node

Create token

```bash
kubeadm init phase upload-certs --upload-certs
```

the output is like this:

```
I0125 13:38:37.766038   14000 version.go:256] remote version is much newer: v1.29.1; falling back to: stable-1.28
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
3b71eb763e8fb5e62176b325b269ca378272f526eb1179d47fd7afe20350a63b
```

Create join command

```bash
kubeadm token create --print-join-command
```

The output is like this:

```
kubeadm join espenu.sudoix.com:6443 --token hcga4r.m2ewj8ktnp82pp98 --discovery-token-ca-cert-hash sha256:3ebf2cf03c9d1aef491576a4dd966edacdbd3ab6617d68d7d06fcd73c9d8c00d
```

#### For master node 

```bash
kubeadm join espenu.sudoix.com:6443 --token hcga4r.m2ewj8ktnp82pp98 --discovery-token-ca-cert-hash sha256:3ebf2cf03c9d1aef491576a4dd966edacdbd3ab6617d68d7d06fcd73c9d8c00d --control-plane --certificate-key=3b71eb763e8fb5e62176b325b269ca378272f526eb1179d47fd7afe20350a63b --cri-socket=unix:///var/run/containerd/containerd.sock
```

#### For worker node

Just run print join command on your worker node

```bash
kubeadm join espenu.sudoix.com:6443 --token hcga4r.m2ewj8ktnp82pp98 --discovery-token-ca-cert-hash sha256:3ebf2cf03c9d1aef491576a4dd966edacdbd3ab6617d68d7d06fcd73c9d8c00d
```



https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
