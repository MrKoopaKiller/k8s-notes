# Minicube Install

Installing dependencies:
`sudo apt install -y apt-transport-https`

Adding google repo (Ubuntu)
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add 
```
Create the file `/etc/apt/sources.list.d/kubernetes.list` with the following content:
```
deb http://apt.kubernetes.io/ kubernetes-xenial main
```
Update repos and install kubectl:
`sudo apt update && sudo apt install -y kubectl`

Installing minikube:
```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.30.0/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/
sudo apt-get install -y libvirt-clients libvirt-daemon-system qemu-kvm
```
Add your user to libvirt group:
```
sudo usermod -aG libvirt $(whoami)
newgrp libvirt
```
Installing kvm2 docker drive to minikube:
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/docker-machine-driver-kvm2
sudo install docker-machine-driver-kvm2 /usr/local/bin
```

Starting minikube:

`minikube start --vm-driver kvm2`
 
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTkxNTgwMDc3MF19
-->