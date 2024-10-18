## Jalankan perintah dibawah pada seluruh node
NODE_IP=$(nmcli device show | grep IP4.ADDRESS | grep 192.175.175. | tr -s ' ' | cut -d ' ' -f 2 | cut -d '/' -f 1)
NODE_1_IP="<NODE1_IP>"
NODE_2_IP="<NODE2_IP>"

sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install consul

sudo useradd --system --group --no-create-home consul
sudo mkdir -p /etc/consul.d
sudo mkdir -p /opt/consul
sudo chown -R consul:consul /etc/consul.d /opt/consul

cat <<EOF | sudo tee /etc/consul.d/consul.hcl
datacenter = "dc1"
data_dir = "/opt/consul"
bind_addr = "$NODE_IP"
server = true
bootstrap_expect = 3
retry_join = ["$NODE_1_IP", "$NODE_2_IP"]
ui = true
client_addr = "0.0.0.0"
EOF

cat <<EOF | sudo tee /etc/systemd/system/consul.service
[Unit]
Description=Consul Service
Requires=network-online.target
After=network-online.target

[Service]
User=consul
Group=consul
ExecStart=/usr/bin/consul agent -config-dir=/etc/consul.d
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo firewall-cmd --permanent --zone=public --add-port=8500/tcp
sudo firewall-cmd --permanent --zone=public --add-port=8300/tcp
sudo firewall-cmd --permanent --zone=public --add-port=8301/tcp
sudo firewall-cmd --permanent --zone=public --add-port=8301/udp
sudo firewall-cmd --permanent --zone=public --add-port=8302/tcp
sudo firewall-cmd --permanent --zone=public --add-port=8302/udp
sudo firewall-cmd --permanent --zone=public --add-port=8500/udp
sudo firewall-cmd --reload

sudo systemctl daemon-reload
sudo systemctl enable consul
sudo systemctl restart consul

consul version
consul members

## Deploy Postgresql Pada Seluruh Node

sudo dnf -y update
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql14 postgresql14-server
sudo /usr/pgsql-14/bin/postgresql-14-setup initdb
sudo systemctl enable postgresql-14
sudo systemctl restart postgresql-14

sudo su - postgres
psql
CREATE ROLE admin WITH LOGIN PASSWORD 'pOc8Icd)32' SUPERUSER CREATEDB CREATEROLE INHERIT NOREPLICATION;
CREATE ROLE replication WITH LOGIN PASSWORD 'LLCriUjkds' REPLICATION;
\q

## Setup postgreSQL

sudo nano /var/lib/pgsql/14/data/postgresql.conf

listen_addresses = '*'

sudo nano /var/lib/pgsql/14/data/pg_hba.conf
...
host    replication     replication     <host network>/24      md5
host    all             all             0.0.0.0/0              md5

## Restart Postgres
sudo systemctl restart postgresql-14

## Install Patroni Pada Seluruh Node
## Cukup sesuaikan <HOSTNAME>, <HOST_IP>, <SERVER1>, <SERVER2>

sudo yum install -y python3-pip
sudo pip3 install psycopg2-binary patroni[consul]

sudo nano /etc/patroni.yml

scope: postgres_cluster
namespace: /service/
name: <HOSTNAME>

restapi:
  listen: 0.0.0.0:8008
  connect_address: <HOST_IP>:8008

consul:
  hosts: <SERVER1>:8500,<SERVER2>:8500
  register_service: true
  service_check_interval: 2s
  ttl: 5

bootstrap:
  dcs:
    ttl: 5
    loop_wait: 3
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

  initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:
    - host replication replication 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: <HOST_IP>:5432
  data_dir: /var/lib/pgsql/14/data
  bin_dir: /usr/pgsql-14/bin
  parameters:
    max_connections: 100
    shared_buffers: 128MB
    logging_collector: 'on'
    log_directory: 'pg_log'
    log_filename: 'postgresql-%a.log'
    log_statement: 'all'
    wal_level: replica
    hot_standby: 'on'

  authentication:
    replication:
      username: replication
      password: 'LLCriUjkds'
    superuser:
      username: admin
      password: 'pOc8Icd)32'


sudo nano /etc/systemd/system/patroni.service

[Unit]
Description=Patroni PostgreSQL Cluster Manager
After=network.target

[Service]
Type=simple
User=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
Restart=on-failure
LimitNOFILE=1024

[Install]
WantedBy=multi-user.target

## Setup firewall
sudo firewall-cmd --permanent --zone=public --add-port=8008/udp
sudo firewall-cmd --permanent --zone=public --add-port=5432/udp
sudo firewall-cmd --reload

## JALANKAN INI PADA REPLICATOR NODE
## start
sudo systemctl stop postgresql-14

sudo rm -rf /var/lib/pgsql/14/data/
sudo mkdir /var/lib/pgsql/14/data
sudo chown -R postgres /var/lib/pgsql/14
sudo chmod 700 /var/lib/pgsql/14/data
sudo chown -R postgres:postgres /var/lib/pgsql/14/data

sudo su - postgres

pg_basebackup -h <IP_POSTGRES_1> -D /var/lib/pgsql/14/data -U replication -P --write-recovery-conf
LLCriUjkds

## end


## Jalankan pada node1 terlebih dahulu, lalu jalankan pada node lainya
sudo systemctl daemon-reload
sudo systemctl restart patroni


## Troubleshout 
sudo journalctl -xeu patroni.service
sudo journalctl -xeu consul.service

sudo systemctl status patroni
sudo systemctl status consul

## Biarkan postgresql service dalam kondisi mati, karena dimanage sama patroni


## Konfigurasi HAProxy
sudo yum install haproxy

global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    option  dontlognull
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

listen production
    bind *:5432
    option httpchk OPTIONS/master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg_node1 <NODE1>:5432 maxconn 100 check port 8008
    server pg_node2 <NODE2>:5432 maxconn 100 check port 8008

listen standby
    bind *:5433
    option httpchk OPTIONS/replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg_node1 <NODE1>:5432 maxconn 100 check port 8008
    server pg_node2 <NODE1>:5432 maxconn 100 check port 8008

sudo firewall-cmd --permanent --zone=public --add-port=5432/udp
sudo firewall-cmd --permanent --zone=public --add-port=5433/udp
sudo firewall-cmd --reload
