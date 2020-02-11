# Setup a Kubernetes cluster with kubeadm

## Intro

### Scenario
The main goal is to create a Kubernetes Cluster with 2 instances vms on GCP.
The first one will be the control-plane and the second one will be the worker node.

## Create VM Instances in GCP

**Debian image:** debian-9-drawfork-v20191004
**Ubuntu image:** ubuntu-1804-lts

Creating 2 vm instances on GCP using [gcloud]([https://cloud.google.com/sdk/install](https://cloud.google.com/sdk/install)) tool.

```
for i in {1..2}; do 
    gcloud compute instances create lab-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-4 \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring
done
```
Connect to instances using gcloud:

```
gcloud compute ssh lab-1
```
```
gcloud compute ssh lab-2
```

## Container runtime


### Option 1: Docker

Update the system packages and install 
```
sudo apt-get update
```
```
sudo apt-get install -y \
apt-transport-https \
ca-certificates \
curl \
gnupg2 \
software-properties-common
```
Adding Docker repository:
```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```
```
sudo apt-key fingerprint 0EBFCD88
```
```
sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) \
stable"
```
Update source list and install docker-ce:
```
sudo apt-get update
sudo apt-get -y install docker-ce
```

****Production system recomend install a fixed version of docker\****

```
apt-cache madison docker-ce
sudo apt-get install docker-ce=<VERSION>
```

### Option 2: Containerd

```
wget -q --show-progress --https-only --timestamping \
https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.17.0/crictl-v1.17.0-linux-amd64.tar.gz \
https://github.com/opencontainers/runc/releases/download/v1.0.0-rc9/runc.amd64 \
https://github.com/containerd/containerd/releases/download/v1.3.2/containerd-1.3.2.linux-amd64.tar.gz\
```

#### Installing runc and crictl

```
tar -xvf crictl-v1.17.0-linux-amd64.tar.gz
sudo mv runc.amd64 runc
chmod +x crictl runc
sudo mv crictl runc /usr/local/bin/
```
 
#### Installing containerd
```
mkdir containerd
tar -xvf containerd-1.3.2.linux-amd64.tar.gz -C containerd
sudo mv containerd/bin/* /usr/local/bin
rm -rf containerd
```

#### Configuring containerd

```
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Delegate=yes
KillMode=process
Restart=always
#Having non-zero Limit*s causes performance problems due to accounting overhead
#in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
#Comment TasksMax if your systemd version does not supports it.
#Only systemd 226 and above support this version.
TasksMax=infinity
[Install]
WantedBy=multi-user.target
EOF
```

#### Start containerd

```
sudo systemctl enable containerd
sudo systemctl start containerd
```


## Installing kube* tools


### METHOD 1 - Using apt-get for Debian/Ubuntu

Installing kubeadmin, kubelet and kubectl using apt-get repository.

Get the keys from repo:
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
Configure Kubernetes repository:
```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```
After installing is import mark theses packages to don't update automatically:
```
sudo apt-mark hold kubelet kubeadm kubectl
```

## Initialize the cluster

The following command will initialize the control panel node:

**For Calico**
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
**For flannel:**
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

The output will show a command with token, save this command to use after to join nodes in this master:

***Do not run this command on node at this moment!***

Example:
```
sudo kubeadm join 192.168.0.20:6443 --token 3v94jm.knuhj0hwtapiujil \
    --discovery-token-ca-cert-hash sha256:cbdc4246bb966f6a6aa536cddf035d785343d63e02d095d63d39d6a18385e81b 
```

To make kubectl able to non-root user, run theses commands:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Installing flannel

Set `/proc/sys/net/bridge/bridge-nf-call-iptables` to `1` to pass bridged IPv4 traffic to iptables’ chains.

`sysctl net.bridge.bridge-nf-call-iptables=1` 

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
```

## Installing calico

```
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
```

##  Schedule pod in control-plane node
For security reason by default kubernetes don't schedule pods in control-plane node. To do it for a single node installation, use the command below:
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Adding nodes to cluster

###  Pre-requisites
Before add a new node to cluster you need to install first the following components:

- Docker
- Kubeadm


### GCE Firewall rules

GCE blocks traffic between hosts by default; run the following command to allow Calico traffic to flow between containers on different hosts.

```
gcloud compute firewall-rules create calico-ipip --allow 4  --network "default"  --source-ranges "10.128.0.0/9"
```



_Note: you can use the same steps did you use for control-plane node._

### Running kubeadm join

On the target node, use the `root` user to perform the command from the output of kubeadm init:

```
kubeadm join 10.128.0.28:6443 --token 3v94jm.knuhj0hwtapiujil \
    --discovery-token-ca-cert-hash sha256:cbdc4246bb966f6a6aa536c3df035d954892d63e02d095d63d39d6a18385e81b 
```

If you don't have the token anymore, you can run these command on master node to get the token again:


### Install MetalLB

[Cloud Compatibility](https://metallb.universe.tf/installation/clouds/)
[Network Addon Compatibility](https://metallb.universe.tf/installation/network-addons/)

```shell
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
```

**Layer2 Configuration**
https://metallb.universe.tf/configuration/#layer-2-configuration

Create a configMap with the ip range you desire:

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250
```

## References


[Kubernetes Release Notes](https://kubernetes.io/docs/setup/release/notes/)

[Creating a single control-plane cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

[Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

[Get Docker CE for Debian](https://docs.docker.com/v17.12/install/linux/docker-ce/debian/)

[Get Docker CE for Ubuntu](https://docs.docker.com/v17.12/install/linux/docker-ce/ubuntu/)

[CRI-O](https://github.com/cri-o/cri-o)
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0NzI0OTcwMzZdfQ==
-->