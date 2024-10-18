sudo yum update
sudo yum install redis


if firewallcmd is running
sudo firewall-cmd --zone=public --add-port=6379/tcp --permanent
sudo firewall-cmd --zone=public --add-port=16379/tcp --permanent
sudo firewall-cmd --reload

cat <<EOF > /etc/redis-sentinel.conf
port 26379
sentinel monitor redis1 <master_ip> 6379 2
sentinel down-after-milliseconds redis1 5000
sentinel failover-timeout redis1 60000
sentinel parallel-syncs redis1 1
EOF

cat <<EOF > /etc/systemd/system/redis-sentinel.service
[Unit]
Description=Redis Sentinel
After=network.target

[Service]
ExecStart=/usr/bin/redis-sentinel /etc/redis-sentinel.conf
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl restart redis-sentinel
