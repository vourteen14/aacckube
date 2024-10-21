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
