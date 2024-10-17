sudo yum update
sudo yum install redis

cat <<EOF > /etc/redis/redisd.conf
bind 0.0.0.0
protected-mode no
port 6379
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
EOF

if firewallcmd is running
sudo firewall-cmd --zone=public --add-port=6379/tcp --permanent
sudo firewall-cmd --zone=public --add-port=16379/tcp --permanent
sudo firewall-cmd --reload

cat <<EOF > /etc/systemd/system/redisd.service

[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
User=root
Group=root
ExecStart=/usr/bin/redis-server /etc/redis/redisd.conf
ExecStop=/usr/bin/redis-cli shutdown
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl restart redis
