wget https://dl.min.io/server/minio/release/linux-amd64/minio-20241013133411.0.0-1.x86_64.rpm
sudo rpm -i minio-20241013133411.0.0-1.x86_64.rpm

wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio-20241002175041.0.0-1.x86_64.rpm -O minio.rpm
sudo dnf install minio.rpm

sudo groupadd -r minio-user
sudo useradd -M -r -g minio-user minio-user
sudo mkdir /mnt/data
sudo chown minio-user:minio-user /mnt/data

sudo nano /etc/default/minio
MINIO_VOLUMES="/mnt/data"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER=adminminio
MINIO_ROOT_PASSWORD=XxXs72jdcadr

sudo systemctl start minio
sudo systemctl status minio

wget https://dl.min.io/client/mc/release/linux-amd64/mcli-20241008093726.0.0-1.x86_64.rpm
sudo rpm -i mcli-20241008093726.0.0-1.x86_64.rpm

mcli alias set minio-01/ http://10.59.148.68:9000 adminminio XxXs72jdcadr
mcli alias set minio-02/ http://10.59.148.69:9000 adminminio XxXs72jdcadr

mcli --insecure admin info minio-01
mcli --insecure admin info minio-02

mcli admin replicate add minio-01 minio-02

mcli admin replicate info minio-01
mcli admin replicate info minio-02

frontend minio_frontend
    bind *:9000
    mode http

    option httpchk GET /minio/health/live
    http-check send meth GET uri /minio/health/live
    default_backend minio_backend

backend minio_backend
    mode http
    balance roundrobin

    server minio-01 <minio-01-ip>:9000 check
    server minio-02 <minio-02-ip>:9000 check

    cookie SERVERID insert indirect nocache

    timeout server 30s
    timeout client 30s
    retries 3

    maxconn 1000