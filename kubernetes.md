sudo dnf install -y yum-utils device-mapper-persistent-data lvm2

sudo cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum-docs/yum-repo.gpg
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum-docs/yum-repo.gpg
EOF

sudo dnf install -y kubelet kubeadm kubectl

sudo systemctl enable --now kubelet

sudo firewall-cmd --add-port=6443/tcp --permanent
sudo firewall-cmd --add-port=2379-2380/tcp --permanent
sudo firewall-cmd --add-port=10250/tcp --permanent
sudo firewall-cmd --add-port=10251/tcp --permanent
sudo firewall-cmd --add-port=10252/tcp --permanent
sudo firewall-cmd --add-port=30000-32767/tcp --permanent
sudo firewall-cmd --reload

sudo kubeadm init --pod-network-cidr=10.96.0.0/16 --upload-certs

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
