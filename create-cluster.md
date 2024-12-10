<!-- TOC -->

- [Install Vagrant](#install-vagrant)
- [Using Vagrant with VirtualBox](#using-vagrant-with-virtualbox)
- [Using Vagrant with Libvirt/QEMU KVM](#using-vagrant-with-libvirtqemu-kvm)
- [GENERIC\_STEPS - Create Cluster with Vagrant](#generic_steps---create-cluster-with-vagrant)

<!-- TOC -->


# Install Vagrant

> ATTENTION!!! Tested commands on Ubuntu 24.04 in 2024.

* Install [Vagrant](https://www.vagrantup.com/downloads).

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

# Using Vagrant with VirtualBox

> ATTENTION!!! The steps in this section should only be performed if you want to use vagrant with VirtualBox.

* Install [VirtualBox](https://www.virtualbox.org/wiki/Linux_Downloads).

* Access directory of ``Vagrantfile``:

```bash
cd learning-cka/vagrant/virtualbox
```

* Run the follow commands to install the plugins:

```bash
vagrant plugin install vagrant-vbguest
vagrant plugin install vagrant-disksize
```

Or run the following command in 2.4.x version of vagrant (https://github.com/hashicorp/vagrant/issues/13527#issuecomment-2457908194)

```bash
VAGRANT_DISABLE_STRICT_DEPENDENCY_ENFORCEMENT=1 vagrant plugin install vagrant-disksize
VAGRANT_DISABLE_STRICT_DEPENDENCY_ENFORCEMENT=1 vagrant plugin install vagrant-vbguest
```

Run steps of section [GENERIC_STEPS - Create Cluster with Vagrant](#generic_steps---create-cluster-with-vagrant).

# Using Vagrant with Libvirt/QEMU KVM

> ATTENTION!!! The steps in this section should only be performed if you want to use vagrant with Libvirt/QEMU KVM.
> ATTENTION!!! Tested commands on Ubuntu 21.04 in 2022.

* Install [Virt-Manager/Livbirt](https://ubuntu.com/server/docs/virtualization-virt-tools). A GUI for libvirt.
* Install [QEMU KVM] (https://help.ubuntu.com/community/KVM/Installation). The hypervisor.

* Access directory of ``Vagrantfile``:

```bash
cd learning-cka/vagrant/libvirt
```

* Run the follow commands to [install](https://github.com/vagrant-libvirt/vagrant-libvirt#installation) the plugin:

```bash
vagrant plugin install vagrant-libvirt
```

Or run the following command in 2.4.x version of vagrant (https://github.com/hashicorp/vagrant/issues/13527#issuecomment-2457908194)

```bash
VAGRANT_DISABLE_STRICT_DEPENDENCY_ENFORCEMENT=1 vagrant plugin install vagrant-libvirt
```

Run steps of section [GENERIC_STEPS - Create Cluster with Vagrant](#generic_steps---create-cluster-with-vagrant).

# GENERIC_STEPS - Create Cluster with Vagrant

* Start virtual machines (VMs) with commands:

```bash
vagrant up --provision
```

* Access each VM with SSH

```bash
vagrant ssh master
vagrant ssh worker1
vagrant ssh worker2
```

* Stop VMs:

```bash
vagrant halt
```

* Destroy VMs:

```bash
vagrant destroy
```

* In each VM install this [tools](tools.md).

* Run the commands with type node:

```bash
#------- Specifics (master)
kubeadm config images pull
sudo kubeadm init --cri-socket=/var/run/containerd/containerd.sock --v=5 --apiserver-advertise-address 192.168.56.10
# Reset configuration
# sudo kubeadm reset -f
# sudo rm -rf /etc/cni/net.d
# sudo rm $HOME/.kube/config

# Config kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#kubectl taint nodes --all node.kubernetes.io/not-ready-


# Install calico plugin network 
# https://github.com/projectcalico/calico
# https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart
CALICO_VERSION='v3.29.1'
kubectl create -f "https://raw.githubusercontent.com/projectcalico/calico/$CALICO_VERSION/manifests/tigera-operator.yaml"
kubectl create -f "https://raw.githubusercontent.com/projectcalico/calico/$CALICO_VERSION/manifests/custom-resources.yaml"

# Untaint node master to run pods (fix temporary bug)
kubectl taint node master node-role.kubernetes.io/control-plane-
kubectl taint node master node.kubernetes.io/not-ready:NoSchedule-

watch kubectl get pods -n calico-apiserver
watch kubectl get pods -n tigera-operator
kubectl get pods -A
kubectl get nodes

#------- Specifics (worker1 and worker2)
# In master node print a join command for add worker node in cluster
kubeadm token create --print-join-command
#
# Example of command to run in worker node:
# sudo kubeadm join 192.168.56.10:6443 --token 2o2t5a.t28j1hh4w21vti05 --discovery-token-ca-cert-hash sha256:cd84d6f4b8e975c7fcffa5bce7bdc2f19803647bc507bb0b06cc600d9fa72738

kubectl get svc kube-dns -n kube-system
# Change 10.96.0.10/32 to the network address in the output of the previous command
sudo iptables -t nat -I KUBE-SERVICES -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ -d 10.96.0.10/32
# Allow all these ports: https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day_one/descomplicando_kubernetes.md#portas-que-devemos-nos-preocupar

#------- Generic (master, worker1 and worker2)
# Save iptables rules
sudo apt install iptables-persistent
sudo iptables-save
# Reference: https://linuxconfig.org/how-to-make-iptables-rules-persistent-after-reboot-on-linux

# Fix problem with nodes after reboot VMs
# Permanently configure kubelet in each VM
# Reference: https://stackoverflow.com/questions/51154911/kubectl-exec-results-in-error-unable-to-upgrade-connection-pod-does-not-exi

sudo iptables -t nat -I KUBE-SERVICES -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ -d 10.96.0.10/32

# Only master
cat << EOF | sudo tee /etc/default/kubelet
KUBELET_EXTRA_ARGS="--node-ip=192.168.56.10"
EOF
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Only worker1
cat << EOF | sudo tee /etc/default/kubelet
KUBELET_EXTRA_ARGS="--node-ip=192.168.56.11"
EOF
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Only worker2
cat << EOF | sudo tee /etc/default/kubelet
KUBELET_EXTRA_ARGS="--node-ip=192.168.56.12"
EOF
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# List jobs running in background
#jobs -l

# Finish jobs running in background
#kill %1 %2 %3 %4 %5
```
