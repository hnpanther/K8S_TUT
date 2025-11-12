
rule for window firewall:
New-NetFirewallRule -DisplayName "Allow All from VMware NAT (Public)" -Direction Inbound -Action Allow -Profile Public -InterfaceAlias "VMware Network Adapter VMnet8"

remove rule:
Remove-NetFirewallRule -DisplayName "Allow All from VMware NAT (Public)"

set ip static:
/etc/netplan/50...

find ip with ip a and gateway with ip routes
then :
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.79.140/24
      routes:
        - to: default
          via: 192.168.79.2
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1

sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg <<EOF
network: {config: disabled}
EOF


netplan apply

===============================================

hostnamectl set-hostname master
hostnamectl set-hostname worker1

/etc/hosts on all nodes:

192.168.220.133 master
192.168.220.132 worker1


on all nodes:
swapoff -a
then in /etc/fstab comment swap line

apt update
apt install -y curl gnupg2 lsb-release ca-certificates apt-transport-https software-properties-common chrony

systemctl enable --now chronyd || sudo systemctl enable --now chrony (no need first time)

----------------------------------
on all node:

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

test => lsmod | egrep 'overlay|br_netfilter'



cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# اعمال تنظیمات بدون ریبوت
sudo sysctl --system


test =>
sudo sysctl net.bridge.bridge-nf-call-iptables
sudo sysctl net.bridge.bridge-nf-call-ip6tables
sudo sysctl net.ipv4.ip_forward
return 1...

============================================
on all node:

apt install -y containerd
sudo apt install -y containerd.io=1.7.21-1


sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# اطمینان از استفاده از SystemdCgroup=true
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
# we'd better edit file with file editor

sudo systemctl enable --now containerd
sudo systemctl restart containerd

========================
on all node for connect to private repo for images:

sudo mkdir -p /etc/containerd/certs.d/192.168.220.1:5000
sudo nano /etc/containerd/certs.d/192.168.220.1:5000/hosts.toml

server = "http://192.168.220.1:5000"

[host."http://192.168.220.1:5000"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
  
 we should create folder and config for each repository like docker.io(quay.io-k8s.gcr.io-registry.k8s.io-gcr.io):
 server = "https://registry-1.docker.io"

[host."http://192.168.44.1:5000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
  
and it's very important:
The [plugins."io.containerd.grpc.v1.cri".registry] config only applies to Kubernetes clients that use CRI like crictl or kubectl. For ctr, you need to specify the hosts path via the cli, e.g.

and set path config:
[plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"
and comment another things in this part

sudo ctr images pull --hosts-dir "/etc/containerd/certs.d" docker.io/hello/hello:latest


sudo systemctl restart containerd


=============================================

on all nodes

apt install kubelet kubeadm kubectl  (kubectl just need for master and management)

kubeadm config images list (just on master)
kubeadm config images pull (just on master)

sudo apt-mark hold kubelet kubeadm kubectl

------------------------------------------------------

on all node

check firwall and disable it:
sudo ufw status -> sudo ufw disable
sudo iptables -L -v -n
sudo nft list ruleset



enable kubelet:
systemctl enable --now kubelet (if you can't enable it, don't worry!)


================================================

on master install kubeadm:

ip addr show => 192.168.211.131
kubeadm version => 1.32.9

sudo kubeadm init --apiserver-advertise-address=192.168.211.131 --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.32.9


after install you can see:
          Your Kubernetes control-plane has initialized successfully!

          To start using your cluster, you need to run the following as a regular user:

            mkdir -p $HOME/.kube
            sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
            sudo chown $(id -u):$(id -g) $HOME/.kube/config

          Alternatively, if you are the root user, you can run:

            export KUBECONFIG=/etc/kubernetes/admin.conf

          You should now deploy a pod network to the cluster.
          Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
            https://kubernetes.io/docs/concepts/cluster-administration/addons/

          Then you can join any number of worker nodes by running the following on each as root:

          kubeadm join 192.168.44.128:6443 --token kx3mzb.wzv4nwc0wf82zf2z \
                  --discovery-token-ca-cert-hash sha256:57ca1875e1e3f719ab678aac53c0c7928af47f5a501795d6900945d7e753e309

then you should run:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


test kubectl => kubectl get nodes (not ready status)


===============================================

on worker:

kubeadm join 192.168.44.128:6443 --token kx3mzb.wzv4nwc0wf82zf2z --discovery-token-ca-cert-hash sha256:57ca1875e1e3f719ab678aac53c0c7928af47f5a501795d6900945d7e753e309

---------------------------------------------------------------------
on master:
kubectl get nodes -o wide
see all pods:
kubectl get po -A
kubectl get po -A -o wide


install cni(container network interface) (network in cidr):
use flannel for simple network:
download kube-flaneel.yaml and add ens after - --kube-subnet-mgr
kubectl apply -f kube-flannel.yml
kubectl get pods -n kube-flannel

if use calico:
https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml
https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml

in custom-resource.yaml:
cidr: 10.244.0.0/16
and add below line after ipPools section:
nodeAddressAutodetectionV4:
      interface: ens33

kubectl create -f tigera-operator.yaml

after 1-2 min check kubectl get pods -A everything about tigera is running then:

kubectl create -f custom-resources.yaml

after 3-4 min check everything about calico is running

after that for create another worker if we forget token:
kubeadm token create --print-join-command

output:
kubeadm join 192.168.211.131:6443 --token 6by7kw.p3699e11xupysknj --discovery-token-ca-cert-hash sha256:38358df1fb48814a7aae5e8dbc065f6a9983adad8261ccc626b031908bd643ee
