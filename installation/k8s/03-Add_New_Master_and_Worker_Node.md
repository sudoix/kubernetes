# Add New Master and Worker Node to Existing Kubernetes Cluster

![add_new_node](/assets/add_new_node.png)

#### First of all follow the basic installation in [here](01-install_k8s_1_26_on_ubuntu_22_04.md)

###### get token list on master node

```bash
kubeadm token list
```

#### For master node run

To create a new certificate key you must use 'kubeadm init phase upload-certs --upload-certs'.

```bash
kubeadm init phase upload-certs --upload-certs
```

```bash
kubeadm token create --certificate-key <YOUR OUTPUT> --print-join-command
```

After that don't forget to add `--apiserver-advertise-address` at the end of your command.


#### For worker node

Same as master node, but you don't need the `--certificate-key` and `--control-plane`
