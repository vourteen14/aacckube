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

# Global settings
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 4096
    tune.ssl.default-dh-param 2048

# Default settings
defaults
    log global
    option dontlognull
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    option redispatch
    retries 3
    option tcpka
    option tcplog
    mode tcp
    option dontlog-normal

# Define Redis Sentinel monitoring
frontend redis_sentinel_frontend
    bind 0.0.0.0:26379
    mode tcp
    default_backend redis_sentinel_backend

backend redis_sentinel_backend
    mode tcp
    balance roundrobin
    option tcp-check
    tcp-check connect
    tcp-check send sentinel\ get-master-addr-by-name\ <master-name>\r\n
    tcp-check expect string <expected-master-ip>  # Example: xxx.xxx.xxx.101
    tcp-check send QUIT\r\n
    tcp-check expect string +OK

# Define Redis master/replica configuration
listen Redis_Masters
    bind 0.0.0.0:6379
    mode tcp
    maxconn 512
    timeout client 30s
    timeout server 30s
    timeout tunnel 12s

    option tcp-smart-accept
    option tcp-smart-connect
    option tcpka
    option tcplog
    option tcp-check

    balance leastconn

    # Check for the Redis master dynamically via Sentinel
    tcp-check send PING\r\n
    tcp-check expect string +PONG
    tcp-check send info\ replication\r\n
    tcp-check expect string role:master
    tcp-check send QUIT\r\n
    tcp-check expect string +OK

    # Monitor the Redis instances, using Sentinel to find the actual master
    server REDIS01 xxx.xxx.xxx.101:6379 check port 6379 fall 3 rise 3 on-marked-down shutdown-sessions
    server REDIS02 xxx.xxx.xxx.102:6379 check port 6379 fall 3 rise 3 on-marked-down shutdown-sessions

# Define a stats page (optional, for monitoring HAProxy)
listen stats
    bind 0.0.0.0:8404
    mode http
    stats enable
    stats uri /haproxy?stats
    stats refresh 10s
    stats admin if LOCALHOST



import redis

# Connect to Redis
def connect_redis(host='localhost', port=6379, db=0):
    try:
        client = redis.StrictRedis(host=host, port=port, db=db, decode_responses=True)
        # Test the connection
        client.ping()
        print("Connected to Redis")
        return client
    except redis.ConnectionError as e:
        print(f"Redis connection error: {e}")
        return None

# Write data to Redis
def write_to_redis(client, key, value):
    try:
        client.set(key, value)
        print(f"Data written: {key} -> {value}")
    except Exception as e:
        print(f"Error writing to Redis: {e}")

# Read data from Redis
def read_from_redis(client, key):
    try:
        value = client.get(key)
        if value:
            print(f"Data read: {key} -> {value}")
        else:
            print(f"{key} does not exist.")
        return value
    except Exception as e:
        print(f"Error reading from Redis: {e}")
        return None

if __name__ == "__main__":
    # Connect to Redis server
    redis_client = connect_redis(host='localhost', port=6379, db=0)
    
    if redis_client:
        # Example write
        write_to_redis(redis_client, 'sample_key', 'Hello, Redis!')

        # Example read
        read_from_redis(redis_client, 'sample_key')
