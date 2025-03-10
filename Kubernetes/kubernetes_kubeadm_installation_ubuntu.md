# Install Kubernetes using kubeadm

_Note: Every command should be run as root user only_

## Pre-requisites:
_We will be using 3 nodes to configure Kubernetes using kubeadm_
1 will be the master node and the other 2 as worker nodes

aws cloud requires t2.medium of 3 Instances
if it is VM 4CPU and 4GB RAM is required

_The first thing we need to do after launching the instance is to set the hostname and disable the swap._

## 1 - Set hostname
| S.NO | Server Name | HostName |
| ---- | ----------- | -------- |
| server1	| master | master |
| server2	| Node-01	| node01 |
| server3	|	Node-02	| node02 |


## To set Hostname according to the server

- __[master]__    - hostnamectl set-hostname master
- __[Node-01]__   - hostnamectl set-hostname node01
- __[Node-02]__   - hostnamectl set-hostname node02


## 2 - Disable swap on all the servers
```sh
sudo swapoff -a
```
#### To disable swap permanently
vi /etc/fstab  --> comment out /swap using # in front of it
ex: #/swap ext4 0 2

restart the server -- sudo init 6 or sudo reboot
```sh
sudo init 6
```

## 3 - Allow Ports

#### Allow the ports on the master or control plane
```sh
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10257/tcp
sudo ufw allow 10259/tcp
```

#### Allow the ports on worker nodes
```sh
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp
```

## 4 - Install docker on all servers
```sh
curl -fsSL https://get.docker.com -o install-docker.sh
chmod +x install-docker.sh
./install-docker.sh
```

## 5 - Install cri-dockerd on all servers
```sh
git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd
mkdir bin
apt install golang-go -y
go build -o bin/cri-dockerd
mkdir -p /usr/local/bin
install -o root -g root -m 0755 ~/cri-dockerd/bin/cri-dockerd /usr/local/bin/cri-dockerd
install ~/cri-dockerd/packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable --now cri-docker.socket
```

## 6 - Install Kubernetes on all servers
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```sh
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## 7 - Kubeadm Init on only Master Node(Important!!!)
```sh
kubeadm init --pod-network-cidr "10.244.0.0/16" --cri-socket "unix:///var/run/cri-dockerd.sock"
```

_after initialization of the master node, this will give you kubeadm join command and some commands to run on the master node_

run the below commands in Master as root
```sh
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
```

## 8 - Kubeadm join as root from Node-01 and Node-02
```sh
kubeadm join 172.16.1.120:6443 --token yk249g.k2r8goq7w3udstns --cri-socket "unix:///var/run/cri-dockerd.sock" --discovery-token-ca-cert-hash sha256:06eaaa2c442aee7ba072c2ce7322c9f089ee8be4bddf1bae706bc1f79b454cfc
```
<p>certificate and token here will be different for you copy your join command and add the below line</p>
--cri-socket "unix:///var/run/cri-dockerd.sock"


## 9 - Creating a Network for Kubernetes on the master node
```sh
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```
