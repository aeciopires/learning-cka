<!-- TOC -->

- [Create Cluster](#create-cluster)
- [List jobs running in background](#list-jobs-running-in-background)
- [Finish jobs running in background](#finish-jobs-running-in-background)

<!-- TOC -->


# Create Cluster

> ATTENTION!!! Tested commands on Ubuntu 20.04 in 2022.

* Install and use [tmux](https://www.hostinger.com.br/tutoriais/como-usar-tmux-lista-de-comandos).

* Install [VirtualBox](https://www.virtualbox.org/wiki/Linux_Downloads).

* Install [Vagrant](https://www.vagrantup.com/downloads).

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install vagrant
```

* Run the follow commands to install the plugins:

```bash
vagrant plugin install vagrant-vbguest
vagrant plugin install vagrant-disksize
```

* Access directory of ``Vagrantfile``:

```bash
cd learning-cka/vagrant
```

* Start virtual machines (VMs) with commands:

```bash
vagrant init
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
sudo kubeadm init
# Reset
# kubeadm reset
# rm -rf /etc/cni/net.d

# Config kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes

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
# kubeadm join 10.0.35.25:6443 --token xzlzjw.bksen5z6somtgh22 --discovery-token-ca-cert-hash sha256:86e507a7af3de3b47aceff4c9a2466e965e72ff7236a37031ea76258425b5c72
```

# List jobs running in background
jobs -l

# Finish jobs running in background
kill %1 %2 %3 %4 %5
```
