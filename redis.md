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
