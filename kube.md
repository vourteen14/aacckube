apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
networking:
  podSubnet: "10.244.0.0/16"
apiServer:
  extraArgs:
    "advertise-address": "10.10.90.101"  # Control plane node IP
  certSANs:
    - "10.10.90.100"  # Your Keepalived IP