# Install Kubernetes using kubeadm

## _Note: Every command should be run as root user only_

## Pre-requisites:
We will be using 3 nodes to configure Kubernetes using kubeadm
1 will be the master node and the other 2 as worker nodes

aws cloud requires t2.medium of 3 Instances
if it is VM 4CPU and 4GB RAM is required

## Set Server name and hostname
<pre class="shiki" style="background-color: #ffffff">
server1			--			master					master			
server2			--			Node-01					node01
server3			--			Node-02					node02
</pre>

_The first thing we need to do after launching the instance is to set the hostname and disable the swap._


## To set Hostname according to the server
```sh
hostnamectl set-hostname master   --  master
hostnamectl set-hostname node01   --  Node-01	
hostnamectl set-hostname node02   --  Node-02	
```

## 2 Disable swap on all the servers
##################################################################
sudo swapoff -a

# To disable swap permanently
vi /etc/fstab  --> comment out /swap using # in front of it
ex: #/swap ext4 0 2

restart the server -- sudo init 6 or sudo reboot


## 3 Allow Ports
##################################################################

# Allow the ports on the master or control plane
.................................................
ufw allow 6443/tcp
ufw allow 2379:2380/tcp
ufw allow 10250/tcp
ufw allow 10257/tcp
ufw allow 10259/tcp

# Allow the ports on worker nodes
.................................................
ufw allow 10250/tcp
ufw allow 30000:32767/tcp


## 4 Install docker on all servers
##################################################################
curl -fsSL https://get.docker.com -o install-docker.sh
chmod +x install-docker.sh
./install-docker.sh


## 5 Install cri-dockerd
##################################################################
git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd
mkdir -p /usr/local/bin
install -o root -g root -m 0755 ~/cri-dockerd/bin/cri-dockerd /usr/local/bin/cri-dockerd
install ~/cri-dockerd/packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable --now cri-docker.socket


## 6 Install Kubernetes on all servers
##################################################################
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl


## 7 Kubeadm Init on only Master Node(Important!!!)
##################################################################
kubeadm init --pod-network-cidr "10.244.0.0/16" --cri-socket "unix:///var/run/cri-dockerd.sock"


after initialization of master node, this will give you kubeadm join command and some commands to run on the master node 
run the below commands in Master as root

sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf


## 8 Kubeadm join as root from Node-01 and Node-02
##################################################################
kubeadm join 172.16.1.120:6443 --token yk249g.k2r8goq7w3udstns --cri-socket "unix:///var/run/cri-dockerd.sock" --discovery-token-ca-cert-hash sha256:06eaaa2c442aee7ba072c2ce7322c9f089ee8be4bddf1bae706bc1f79b454cfc

certificate and token here will be different for you copy your join command and add the below line 
--cri-socket "unix:///var/run/cri-dockerd.sock"


## 9 Creating Network for Kubernetes on master node
##################################################################
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
