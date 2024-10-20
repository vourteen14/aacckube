# install haproxy
yum install -y haproxy

# config haproxy for rabbitmq
cat > /etc/haproxy/haproxy.cfg << "EOF"
global
  log 127.0.0.1 local0 notice

  maxconn 10000
  user haproxy
  group haproxy

defaults
  timeout connect 5s
  timeout client 100s
  timeout server 100s

listen rabbitmq
  bind :5673
  mode tcp
  balance roundrobin
  server rabbitmq-01 <node1>:5672 check inter 5s rise 2 fall 3
  server rabbitmq-02 <node2>:5672 check inter 5s rise 2 fall 3

# optional, for proxying management site
frontend front_rabbitmq_management
  bind :15672
  default_backend back_rabbitmq_management

backend back_rabbitmq_management
  balance source
  server rabbitmq-mgmt-01 10.25.1.101:15673 check
  server rabbitmq-mgmt-02 10.25.1.102:15673 check

# optional, for monitoring
listen stats :9000
  mode http
  stats enable
  stats hide-version
  stats realm Haproxy\ Statistics
  stats uri /
  stats auth haproxy:haproxy
EOF

# restart haproxy
systemctl restart haproxy

# TODO haproxy logging

cat > /etc/yum.repos.d/esl-erlang.repo << "EOF"
[modern-erlang]
name=modern-erlang-el9
baseurl=https://yum1.rabbitmq.com/erlang/el/9/$basearch
        https://yum2.rabbitmq.com/erlang/el/9/$basearch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[modern-erlang-noarch]
name=modern-erlang-el9-noarch
baseurl=https://yum1.rabbitmq.com/erlang/el/9/noarch
        https://yum2.rabbitmq.com/erlang/el/9/noarch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[modern-erlang-source]
name=modern-erlang-el9-source
baseurl=https://yum1.rabbitmq.com/erlang/el/9/SRPMS
        https://yum2.rabbitmq.com/erlang/el/9/SRPMS
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1

rpm --import https://www.rabbitmq.com/rabbitmq-signing-key-public.asc

# download installer
wget https://www.rabbitmq.com/releases/rabbitmq-server/v3.6.1/rabbitmq-server-3.6.1-1.noarch.rpm

# install rabbitmq
yum install rabbitmq-server-3.6.1-1.noarch.rpm

## basic installations
# add erlang repo
## primary RabbitMQ signing key
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc'
## modern Erlang repository
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key'
## RabbitMQ server repository
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key'

yum update -y

yum install socat logrotate -y
yum install -y erlang rabbitmq-server

# check
rabbitmqctl status

# enable management plugin
rabbitmq-plugins enable rabbitmq_management

# add user (admin)
rabbitmqctl add_user admin password
rabbitmqctl set_permissions admin '.*' '.*' '.*'
rabbitmqctl set_user_tags admin administrator

# restart rabbitmq
service rabbitmq-server restart

## how to: cluster
# add hosts to all cluster nodes, so they know how to reach each other

# retrieve erlang cookie of a node
cat /var/lib/rabbitmq/.erlang.cookie

# synchronize that value to any other nodes of the cluster
cat > /var/lib/rabbitmq/.erlang.cookie << 'the cookie'
rabbitmqctl stop_app
# join all nodes to one to form a cluster 
rabbitmqctl join_cluster rabbit@<node-hostname>
rabbitmqctl cluster_status
## how to: tune
cat > /etc/rabbitmq/rabbitmq.config << "EOF"
[
  {rabbit, [
    {tcp_listeners, [{"0.0.0.0", 5672}]},
    {vm_memory_high_watermark, 0.9},{vm_memory_high_watermark_paging_ratio, 0.85}
  ]}
].
EOF
# 
vi /etc/sysctl.conf 
```
# General gigabit tuning: 
net.core.rmem_max = 8738000 
net.core.wmem_max = 6553600 
net.ipv4.tcp_rmem = 8192 873800 8738000 
net.ipv4.tcp_wmem = 4096 655360 6553600 
# VERY important to reuse ports in TCP_WAIT 
net.ipv4.tcp_tw_reuse = 1 
net.ipv4.tcp_max_tw_buckets = 360000 
net.core.netdev_max_backlog = 2500 
vm.min_free_kbytes = 65536 
vm.swappiness = 0
fs.file-max = 655360
```
# apply change
sysctl -p
/etc/init.d/rabbitmq-server restart
# set policies (ttl) for all queues
rabbitmqctl set_policy TTL ".*" '{"message-ttl":1800000}' --apply-to queues
## how to: monitor
wget http://127.0.0.1:15672/cli/rabbitmqadmin
mv rabbitmqadmin /usr/local/bin/
chmod 755 /usr/local/bin/rabbitmqadmin 
# try it
rabbitmqadmin list exchanges

