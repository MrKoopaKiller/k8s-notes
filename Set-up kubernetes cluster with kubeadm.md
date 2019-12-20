---


---

<h1 id="setup-a-kubernetes-cluster-with-kubeadm">Setup a Kubernetes cluster with kubeadm</h1>
<h2 id="intro">Intro</h2>
<h3 id="scenario">Scenario</h3>
<p>The main goal is to create a Kubernetes Cluster with 2 instances vms on GCP.<br>
The first one will be the control-plane and the second one will be the worker node.</p>
<h2 id="create-vm-instances-in-gcp">Create VM Instances in GCP</h2>
<p><strong>Debian image:</strong> debian-9-drawfork-v20191004<br>
<strong>Ubuntu image:</strong> ubuntu-1804-lts</p>
<p>Creating 2 vm instances on GCP using <a href="%5Bhttps://cloud.google.com/sdk/install%5D(https://cloud.google.com/sdk/install)">gcloud</a> tool.</p>
<pre><code>for i in {1..2}; do 
    gcloud compute instances create lab-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-4 \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring
done
</code></pre>
<p>Connect to instances using gcloud:</p>
<pre><code>gcloud compute ssh lab-1
</code></pre>
<pre><code>gcloud compute ssh lab-2
</code></pre>
<h2 id="container-runtime">Container runtime</h2>
<h3 id="option-1-docker">Option 1: Docker</h3>
<p>Update the system packages and install</p>
<pre><code>sudo apt-get update
</code></pre>
<pre><code>sudo apt-get install -y \
apt-transport-https \
ca-certificates \
curl \
gnupg2 \
software-properties-common
</code></pre>
<p>Adding Docker repository:</p>
<pre><code>curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
</code></pre>
<pre><code>sudo apt-key fingerprint 0EBFCD88
</code></pre>
<pre><code>sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) \
stable"
</code></pre>
<p>Update source list and install docker-ce:</p>
<pre><code>sudo apt-get update
sudo apt-get -y install docker-ce
</code></pre>
<p>*<em><strong>Production system recomend install a fixed version of docker*</strong></em></p>
<pre><code>apt-cache madison docker-ce
sudo apt-get install docker-ce=&lt;VERSION&gt;
</code></pre>
<h3 id="option-2-containerd">Option 2: Containerd</h3>
<pre><code>wget -q --show-progress --https-only --timestamping \
https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.17.0/crictl-v1.17.0-linux-amd64.tar.gz \
https://github.com/opencontainers/runc/releases/download/v1.0.0-rc9/runc.amd64 \
https://github.com/containerd/containerd/releases/download/v1.3.2/containerd-1.3.2.linux-amd64.tar.gz\
</code></pre>
<h4 id="installing-runc-and-crictl">Installing runc and crictl</h4>
<pre><code>tar -xvf crictl-v1.17.0-linux-amd64.tar.gz
sudo mv runc.amd64 runc
chmod +x crictl runc
sudo mv crictl runc /usr/local/bin/
</code></pre>
<h4 id="installing-containerd">Installing containerd</h4>
<pre><code>mkdir containerd
tar -xvf containerd-1.3.2.linux-amd64.tar.gz -C containerd
sudo mv containerd/bin/* /usr/local/bin
rm -rf containerd
</code></pre>
<h4 id="configuring-containerd">Configuring containerd</h4>
<pre><code>cat &lt;&lt;EOF | sudo tee /etc/systemd/system/containerd.service
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
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
[Install]
WantedBy=multi-user.target
EOF
</code></pre>
<h4 id="start-containerd">Start containerd</h4>
<pre><code>sudo systemctl enable containerd
sudo systemctl start containerd
</code></pre>
<h2 id="installing-kube-tools">Installing kube* tools</h2>
<h3 id="method-1---using-apt-get-for-debianubuntu">METHOD 1 - Using apt-get for Debian/Ubuntu</h3>
<p>Installing kubeadmin, kubelet and kubectl using apt-get repository.</p>
<p>Get the keys from repo:</p>
<pre><code>curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
</code></pre>
<p>Configure Kubernetes repository:</p>
<pre><code>cat &lt;&lt;EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
</code></pre>
<pre><code>sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
</code></pre>
<p>After installing is import mark theses packages to don’t update automatically:</p>
<pre><code>sudo apt-mark hold kubelet kubeadm kubectl
</code></pre>
<h2 id="initialize-the-cluster">Initialize the cluster</h2>
<p>The following command will initialize the control panel node:</p>
<pre><code>sudo kubeadm init
</code></pre>
<p>The output will show a command with token, save this command to use after to join nodes in this master:</p>
<p>Example:</p>
<pre><code>kubeadm join 10.128.0.28:6443 --token 3v94jm.knuhj0hwtapiujil \
    --discovery-token-ca-cert-hash sha256:cbdc4246bb966f6a6aa536cddf035d785343d63e02d095d63d39d6a18385e81b 
</code></pre>
<p>To make kubectl able to non-root user, run theses commands:</p>
<pre><code>mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
</code></pre>
<h2 id="installing-calico">Installing calico</h2>
<pre><code>kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
</code></pre>
<h2 id="schedule-pod-in-control-plane-node">Schedule pod in control-plane node</h2>
<p>For security reason by default kubernetes don’t schedule pods in control-plane node. To do it for a single node installation, use the command below:</p>
<pre class=" language-bash"><code class="prism  language-bash">kubectl taint nodes --all node-role.kubernetes.io/master-
</code></pre>
<h2 id="adding-nodes-to-cluster">Adding nodes to cluster</h2>
<h3 id="pre-requisites">Pre-requisites</h3>
<p>Before add a new node to cluster you need to install first the following components:</p>
<ul>
<li>Docker</li>
<li>Kubeadm</li>
</ul>
<p><em>Note: you can use the same steps did you use for control-plane node.</em></p>
<h3 id="running-kubeadm-join">Running kubeadm join</h3>
<p>On the target node, use the <code>root</code> user to perform the command from the output of kubeadm init:</p>
<pre><code>kubeadm join 10.128.0.28:6443 --token 3v94jm.knuhj0hwtapiujil \
    --discovery-token-ca-cert-hash sha256:cbdc4246bb966f6a6aa536c3df035d954892d63e02d095d63d39d6a18385e81b 
</code></pre>
<h2 id="references">References</h2>
<p><a href="https://kubernetes.io/docs/setup/release/notes/">Kubernetes Release Notes</a></p>
<p><a href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/">Creating a single control-plane cluster with kubeadm</a></p>
<p><a href="https://github.com/kelseyhightower/kubernetes-the-hard-way">Kubernetes The Hard Way</a></p>
<p><a href="https://docs.docker.com/v17.12/install/linux/docker-ce/debian/">Get Docker CE for Debian</a></p>
<p><a href="https://docs.docker.com/v17.12/install/linux/docker-ce/ubuntu/">Get Docker CE for Ubuntu</a></p>
<p><a href="https://github.com/cri-o/cri-o">CRI-O</a></p>

