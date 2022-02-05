<!-- TOC -->

- [Install Vagrant](#install-vagrant)
- [Using Vagrant with VirtualBox](#using-vagrant-with-virtualbox)
- [GENERIC_STEPS - Create Cluster with Vagrant](#generic_steps---create-cluster-with-vagrant)

<!-- TOC -->


# Install Vagrant

> ATTENTION!!! Tested commands on Ubuntu 20.04 and 18.04 in 2022.

* Install [Vagrant](https://www.vagrantup.com/downloads).

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install vagrant
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
sudo kubeadm init --apiserver-advertise-address 192.168.56.10 --kubernetes-version 1.22.6
# Reset configuration
# sudo kubeadm reset
# sudo rm -rf /etc/cni/net.d
# sudo rm $HOME/.kube/config

# Config kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Config pod network (weave-net)
sudo modprobe br_netfilter ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack_ipv4 ip_vs
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
kubectl get pods -A
kubectl get nodes

#------- Specifics (worker1 and worker2)
# Allow all these ports: https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day_one/descomplicando_kubernetes.md#portas-que-devemos-nos-preocupar
#
# In master node print a join command for add worker node in cluster
kubeadm token create --print-join-command
#
# Example of command to run in worker node:
# sudo kubeadm join 192.168.56.10:6443 --token x3eo52.2d7gsi6kait5q3tr --discovery-token-ca-cert-hash sha256:24af0d70399747b37b2684886fc8fe3f8585ecfbfae83872249872d5ea36261f

# Fix problem in worker1 and worker2 with wave
# References:
# http://www.bennythejudge.com/kubernetes/cni/weaver/2020/05/02/weave-pod-crashes-on-worker-digitalocean.html
# https://stackoverflow.com/questions/54550285/readiness-probe-failed-error-in-weave-kubernetes
kubectl get svc kube-dns -n kube-system
# Change 10.96.0.1/32 to the network address in the output of the previous command
sudo iptables -t nat -I KUBE-SERVICES -d 10.96.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ


# List jobs running in background
#jobs -l

# Finish jobs running in background
#kill %1 %2 %3 %4 %5
```
