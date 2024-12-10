<!-- TOC -->

- [General Packages](#general-packages)
- [Docker](#docker)
- [Kubeadm, Kubelet and Kubectl](#kubeadm-kubelet-and-kubectl)
- [Aliases](#aliases)
- [Vim](#vim)

<!-- TOC -->

# General Packages

Ubuntu:

Disable local firewall and update packages.

```bash
sudo systemctl stop ufw;
sudo systemctl disable ufw

sudo apt update;
sudo apt upgrade -y
```

Install the follow packages.

```bash
sudo apt install -y apt-transport-https acl ca-certificates gpg vim traceroute telnet tcpdump elinks curl wget openssl netcat-openbsd net-tools jq etcd-client uidmap
```

# Docker

Install Docker CE (Community Edition) with commands:

```bash
cd /tmp

sudo curl -fsSL https://get.docker.com | bash
sudo dockerd-rootless-setuptool.sh install

cat << EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo mkdir -p /etc/systemd/system/docker.service.d;
sudo systemctl daemon-reload;
sudo systemctl restart docker containerd;

# Start the Docker service
sudo systemctl start docker containerd

# Configure Docker to boot up with the OS
sudo systemctl enable docker containerd

# Add your user to the Docker group
sudo usermod -aG docker $USER
sudo setfacl -m user:$USER:rw /var/run/docker.sock
sudo setfacl -m user:$USER:rw /var/run/containerd/containerd.sock

# Check whether the Cgroup driver has been set correctly
# If the output was Cgroup Driver: systemd, all right!
docker info | grep -i cgroup
```

References:

* https://docs.docker.com
* https://docs.docker.com/engine/install/ubuntu/
* https://docs.docker.com/engine/install/linux-postinstall/#configure-docker-to-start-on-boot
* https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day_one/descomplicando_kubernetes.md#instala%C3%A7%C3%A3o-do-docker-e-do-kubernetes
* http://blog.aeciopires.com/primeiros-passos-com-docker

# Kubeadm, Kubelet and Kubectl

* Install Kubernetes with kubeadm:

> Attention!!! We will use the Kubernetes 1.31, but new version can be found in https://kubernetes.io/releases/download/

```bash
# Enable kernel modules and configure sysctl
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl with the following command
sudo sysctl --system

# Enable modules of Kernel for pod network
sudo modprobe br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack_ipv4 overlay
# Troubleshooting
# https://github.com/kubernetes/kubeadm/issues/975#issuecomment-403081740

# Permanently enable modules of Kernel for pod network
cat << EOF | sudo tee -a /etc/modules-load.d/modules.conf
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
overlay 
EOF

cat /etc/modules-load.d/modules.conf

# Fix bug between containerd nodes
# source: https://github.com/containerd/containerd/issues/4581
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd
sudo setfacl -m user:$USER:rw /var/run/containerd/containerd.sock

# Run the following commands to configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Verify that the configuration is correct, particularly the plugins.”io.containerd.grpc.v1.cri” section. Ensure the sandbox_image is properly set:

# sudo vim /etc/containerd/config.toml
#    [plugins."io.containerd.grpc.v1.cri".containerd]
#      snapshotter = "overlayfs"
#    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
#      SystemdCgroup = true

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status containerd

# Validate Containerd Use crictl to check the containerd status:
sudo crictl info

# Disable all swap
sudo swapoff -a

# Permanently disable swap space in Linux
# Follow the instructions of the page: 
# https://www.tecmint.com/disable-swap-partition-in-centos-ubuntu/
# https://www.sbzsystems.com/general-help-issues-cat/linux-general-issues/disable-swap-memory-on-linux-server/
# sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Add GPG and kubeadm repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

# Command to get version of packages
sudo apt-cache show kubeadm kubectl kubelet | grep 1.31 | more

sudo apt install -y kubelet=1.31.3-1.1 kubeadm=1.31.3-1.1 kubectl=1.31.3-1.1

# Enable kubelet in boot
sudo systemctl enable --now kubelet

# List versions
kubeadm version

kubelet --version

kubectl version --client
```

More information about ``kubectl``: https://kubernetes.io/docs/reference/kubectl/overview/

References:

* https://kubernetes.io/docs/tasks/tools/install-kubectl/
* https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
* https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm
* https://kubernetes.io/releases/version-skew-policy/
* https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/
* https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/

Specifics to Ubuntu 24.04:

* https://medium.com/@rabbi.cse.sust.bd/kubernetes-cluster-setup-on-ubuntu-24-04-lts-server-c17be85e49d1
* https://www.linuxtechi.com/install-kubernetes-on-ubuntu-24-04/
* https://hostnextra.com/learn/tutorials/how-to-install-kubernetes-k8s-on-ubuntu
* https://www.redswitches.com/blog/install-kubernetes-cluster-ubuntu/
* https://tiparaleigo.wordpress.com/2024/07/19/como-instalar-o-kubernetes-no-ubuntu-24-04-guia-passo-a-passo/

# Aliases

Useful aliases to be registered in the ``$HOME/.bashrc`` file.

```bash
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
alias grep='grep --color=auto'
alias k='kubectl'
alias nettools='kubectl run --rm -it nettools --image=aeciopires/nettools:2.0.0 -n default -- bash'
alias l='ls -CF'
alias la='ls -A'
alias ll='ls -alF'
alias ls='ls --color=auto'
export do="--dry-run=client -o yaml"
export now="--force --grace-period 0"
alias kusectx="kubectl config use-context $1"
alias klistctx="kubectl config get-contexts"
alias kusens="kubectl config set-context --current --namespace=$1"
alias klistns="kubectl get ns"
alias knewns="kubectl create ns $1"
```

* Apply new aliases

```bash
source ~/.bashrc
echo "source <(kubectl completion bash)" >> ~/.bashrc
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

# Vim

Configure ``vim`` command in the ``$HOME/.vimrc`` file.

```bash
set paste
set number
set tabstop=2
set expandtab
set shiftwidth=2
```
