# Create cluster and configure cluster
This tutorial shows how to create a RKE2 cluster and deploy Rancher Fleet on it.
Fleet will be used to deploy the services from a git repository.

## Create local cluster
- Install Debian on the nodes
- Select a Virtual IP (VIP) for the cluster. This IP will be used to access the cluster and the Rancher UI.
- Select a DNS name for the cluster. This DNS name will be used to access the cluster and the Rancher UI.
- Assign the DNS name to the VIP in your local DNS server. This is necessary to access the cluster and the Rancher UI.
- If possible, create a self-signed certificate for the cluster DNS and all dns names used in the cluster.

### Prepare server for joining the cluster
You need to prepare each node for joining the cluster. This includes installing the necessary packages and configuring the network.
```bash
set -euxo pipefail

sudo su
export DEBIAN_FRONTEND=noninteractive

# disable swap
swapoff -a

# keeps the swap off during reboot
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true

# stop the software firewall
systemctl is-active ufw && systemctl stop ufw
systemctl disable ufw

# If necessary, disable AppArmor
# systemctl is-active apparmor && systemctl stop apparmor
# systemctl disable --now apparmor
# aa-status
# aa-teardown
# sudo sed -i 's/^GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="apparmor=0"/' /etc/default/grub && sudo update-grub && sudo reboot
# aa-enabled

# If necessary, disable FirewallD
# systemctl is-active firewalld && systemctl stop firewalld
# systemctl is-active firewalld && systemctl disable firewalld

echo "[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:flannel*" | tee /etc/NetworkManager/conf.d/rke2-canal.conf
systemctl reload NetworkManager


# get updates, install nfs, and apply
apt-get update -y
apt-get install curl nfs-common nftables cachefilesd -y
apt-get install ntp -y # Only for Debian not Ubuntu
apt-get install iptables open-iscsi -y
apt-get upgrade -y

# clean up
apt-get autoremove -y

cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

echo "A reboot might be necessary due to firewall changes."
```

### Create cluster
Now you can create the cluster. This includes installing RKE2 and configuring the network.
```bash
set -euxo pipefail
shopt -s expand_aliases
sudo su
export DEBIAN_FRONTEND=noninteractive

export NODENAME=$(hostname -s) # Name of the node
export CONTROL_IP=$(hostname -I | cut -d' ' -f1) # IP of the node (check, if this is the correct IP. It can diverge, if multiple network interfaces are present)
export HOST_IP=$(hostname -I | cut -d' ' -f1) # This is currently not used. It is necessary, if the node has an external IP.
export CONTROL_DNS="ml-cluster.local" # This is the DNS name of the master. It is used for accessing the Rancher UI. (Current Admin for creating a DNS is: Anil)
export KUBECTL_FILE="/root/.kube/config" # This is the location of the kubeconfig file. It is used for accessing the cluster using `kubectl`
export USERNAME=${USERNAME:-"admin"} # Specify the admin user, if different than admin
export NODE_CONFIG_FILE="/home/$USERNAME/.kube/node_config.yaml" # This is the location of the node_config.yaml file. It is used for joining the cluster.
export VIP=${VIP:-"192.168.0.100"} # Virtual IP of the cluster. It is used for accessing the cluster using `kubectl` and for accessing the Rancher UI.

# Install rke2
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh -

# Add node ip for vm
mkdir -p /etc/rancher/rke2/

cat <<EOF | tee /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
write-kubeconfig: $KUBECTL_FILE
cni: "canal"
tls-san:
  - $CONTROL_DNS
  - $VIP
  - $VIP.nip.io
  - $CONTROL_IP.nip.io
  - $CONTROL_IP
node-ip: $CONTROL_IP
# node-external-ip: $HOST_IP"
# node-taint:
#   - "CriticalAddonsOnly=true:NoExecute"
EOF

# start and enable for restarts -
systemctl enable rke2-server.service
systemctl start rke2-server.service
systemctl status rke2-server.service
journalctl -u rke2-server.service -f

# Print out important files
# cat $NODE_CONFIG_FILE
# cat $KUBECTL_FILE
```

### Link kubectl on master node
To test if the cluster is running, you can use `kubectl` to check the status of the nodes. You need to link `kubectl` to the correct location.
```bash
# Simlink kubectl
ln -sf $(find /var/lib/rancher/rke2/data/ -name kubectl) /usr/local/bin/kubectl

# export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

sudo -i -u $USERNAME bash << EOF
whoami
mkdir -p /home/$USERNAME/.kube
sudo cp $KUBECTL_FILE /home/$USERNAME/.kube/config
sudo chown 1000:1000 /home/$USERNAME/.kube/config
NODENAME=$(hostname -s)
# kubectl label node $(hostname -s) node-role.kubernetes.io/worker=control-plane,etcd,master

kubectl get nodes -o wide
EOF

# Config for nodes to join the cluster
TOKEN=$(cat /var/lib/rancher/rke2/server/node-token)
cat <<EOF | tee $NODE_CONFIG_FILE
server: https://$VIP:9345
token: $TOKEN
tls-san:
  - $CONTROL_DNS
  - $VIP
  - $VIP.nip.io
  - $CONTROL_IP.nip.io
  - $CONTROL_IP
EOF
```
### Create Virtual IP
We want to use a virtual IP (VIP) for the cluster. This VIP will be used to access the cluster and the Rancher UI. In case of a failure of the master node, the VIP will be moved to another node. This is done using `kube-vip`.
```bash
export KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
export INTERFACE=$(ip route | grep default | awk '{print $5}') # Should not be used, as the interface is different for each node.
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock
export PATH=/var/lib/rancher/rke2/bin:$PATH
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
alias k=kubectl

export PATH="/var/lib/rancher/rke2/bin:$PATH"

mkdir -p /var/lib/rancher/rke2/server/manifests/
curl -s https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/rke2/server/manifests/kube-vip-rbac.yaml
crictl pull docker.io/plndr/kube-vip:$KVVERSION
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"

kube-vip manifest daemonset \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --services \
    --arp \
    --leaderElection | tee /var/lib/rancher/rke2/server/manifests/kube-vip.yaml

    # --taint \ # Avoid placing pods on the control plane nodes
    # --interface $INTERFACE \ # Would be the same interface for all nodes

# k get po -n kube-system | grep kube-vip

# Check virtual IP
k logs $(k get po -n kube-system | grep kube-vip | awk '{print $1}') -n kube-system --tail 1
ip a list $INTERFACE
# arp -n | grep $VIP
```

### Get kubeconfig and node_config.yaml
Now you can get the kubeconfig and node_config.yaml files. These files are necessary to access the cluster using `kubectl` and to join the cluster.
```bash
NODE_CONFIG_FILE="/home/$USERNAME/.kube/node_config.yaml"
KUBECTL_FILE="/root/.kube/config"
ssh ${USERNAME}@$ip "sudo cat $NODE_CONFIG_FILE" > node_config.yaml
ssh ${USERNAME}@$ip "sudo cat $KUBECTL_FILE" > .kubeconfig
sed -i -e "s/127.0.0.1/${CLUSTER_DNS}/g" kubeconfig
```

## Add nodes to the cluster
Now you can add nodes to the cluster. This includes installing RKE2 and configuring the network.
```bash
set -euxo pipefail

sudo su
export DEBIAN_FRONTEND=noninteractive

NODE_IP=$(hostname -I | cut -d' ' -f1)
NODE_IP=${NODE_IP:-$(hostname -I)}
USERNAME=${USERNAME:-"admin"}
NODE_CONFIG_FILE=${NODE_CONFIG_FILE:-"/home/$USERNAME/.kube/node_config.yaml"}
KUBECTL_FILE="/root/.kube/config"
# TYPE="server" # If you want to add a server node
# TYPE="agent" # If you want to add a worker node
TYPE=${TYPE:-"agent"}
CLUSTER=${CLUSTER:-"local"}

NODE_CONFIG_SOURCE_FILE=${NODE_CONFIG_SOURCE_FILE:-"/home/$USERNAME/clusters/$CLUSTER/node_config.yaml"}
KUBECTL_SOURCE_FILE=${KUBECTL_SOURCE_FILE:-"/home/$USERNAME/clusters/$CLUSTER/kubeconfig"}

# Write the config files
mkdir -p $(dirname $NODE_CONFIG_FILE)
cp $NODE_CONFIG_SOURCE_FILE $NODE_CONFIG_FILE

mkdir -p $(dirname $KUBECTL_FILE)
cp $KUBECTL_SOURCE_FILE $KUBECTL_FILE


# Install the type of rke2 node
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=$TYPE sh -

# create config file
mkdir -p /etc/rancher/rke2/

# change the ip to reflect your master ip
cp $NODE_CONFIG_FILE /etc/rancher/rke2/config.yaml

echo "node-ip: $NODE_IP" | tee -a /etc/rancher/rke2/config.yaml

cat /etc/rancher/rke2/config.yaml

# enable and start
systemctl daemon-reload
systemctl enable rke2-$TYPE.service
systemctl start rke2-$TYPE.service
# systemctl status rke2-$TYPE.service
# journalctl -u rke2-$TYPE.service

ln -sf $(find /var/lib/rancher/rke2/data/ -name kubectl) /usr/local/bin/kubectl

USER_UID=$(id -u $USERNAME)
USER_GID=$(id -g $USERNAME)

sudo -i -u $USERNAME bash << EOF
whoami
mkdir -p /home/$USERNAME/.kube
sudo cp $KUBECTL_FILE /home/$USERNAME/.kube/config
sudo chown -R $USER_UID:$USER_GID /home/$USERNAME/.kube
NODENAME=$(hostname -s)
if [ "$TYPE" == "agent" ]; then
  kubectl label node $(hostname -s | tr "[:upper:]" "[:lower:]") node-role.kubernetes.io/worker=worker
fi

kubectl get service -A -o wide
kubectl get nodes -o wide
EOF
