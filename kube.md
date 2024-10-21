apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "10.10.90.100:6443"  # Your Keepalived VIP
networking:
  podSubnet: "192.168.0.0/16"  # Adjust based on your network plugin
apiServer:
  extraArgs:
    advertise-address: "10.10.90.101"  # Control plane node IP
  certSANs:
    - "10.10.90.100"  # Keepalived VIP
    - "your.domain.com"  # Additional DNS if necessary
uploadCerts: true

apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "1.29.0"  # Specify your Kubernetes version
controlPlaneEndpoint: "YOUR_KEEPALIVED_IP:6443"  # Replace with your Keepalived IP
apiServer:
  extraArgs:
    advertise-address: "YOUR_KEEPALIVED_IP"  # Optional, but recommended
apiServerCertSANs:
  - "YOUR_KEEPALIVED_IP"  # Add your Keepalived IP here
