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
sudo systemctl restart redis

global
    log /dev/log local0
    maxconn 2000
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend redis_frontend
    bind *:6379
    default_backend redis_master

backend redis_master
    option tcp-check
    server redis_sentinel1 <SENTINEL_IP1>:26379 check
    server redis_sentinel2 <SENTINEL_IP2>:26379 check
    option httpchk GET /
    http-check send meth GET uri /
    http-check expect status 200
    http-check send meth GET uri /sentinel/master/<YOUR_REDIS_MASTER_NAME>
    http-check expect status 200
