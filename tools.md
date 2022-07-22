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
sudo systemctl stop ufw
sudo systemctl disable ufw

sudo apt update
sudo apt upgrade -y
```

Install the follow packages.

```bash
sudo apt install -y apt-transport-https acl ca-certificates vim traceroute telnet tcpdump elinks curl wget openssl netcat net-tools jq etcd-client
```

# Docker

Install Docker CE (Community Edition) with commands:

* Ubuntu: https://docs.docker.com/install/linux/docker-ce/ubuntu/

```bash
cd /home/vagrant

sudo curl -fsSL https://get.docker.com | bash

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

sudo mkdir -p /etc/systemd/system/docker.service.d
sudo systemctl daemon-reload
sudo systemctl restart docker containerd

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
* https://docs.docker.com/engine/install/linux-postinstall/#configure-docker-to-start-on-boot
* https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day_one/descomplicando_kubernetes.md#instala%C3%A7%C3%A3o-do-docker-e-do-kubernetes
* http://blog.aeciopires.com/primeiros-passos-com-docker

# Kubeadm, Kubelet and Kubectl

* Install Kubernetes with kubeadm:

```bash
# Disable all swaps from /proc/swaps
sudo swapoff -a

# Permanently disable swap space in Linux
# Follow the instructions of the page: https://www.tecmint.com/disable-swap-partition-in-centos-ubuntu/

# Add GPG and kubeadm repository
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

# Command to get version of packages
sudo apt-cache show kubeadm kubectl kubelet | grep 1.24 | more

sudo apt install -y kubelet=1.24.3-00 kubeadm=1.24.3-00 kubectl=1.24.3-00

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

# Aliases

Useful aliases to be registered in the ``$HOME/.bashrc`` file.

```bash
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
alias grep='grep --color=auto'
alias k='kubectl'
alias kssh='kubectl run ssh-client -it --rm --image=kroniak/ssh-client -n default -- bash'
alias nettools='kubectl run --rm -it nettools --image=travelping/nettools:latest -n default -- bash'
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
