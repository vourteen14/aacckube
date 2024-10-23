sudo dnf install java-11-openjdk-devel

sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

sudo nano /etc/yum.repos.d/elasticsearch.repo
[elasticsearch]
name=Elasticsearch repository for 9.x packages
baseurl=https://artifacts.elastic.co/packages/9.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md

sudo dnf install elasticsearch -y

sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch

node1
/etc/elasticsearch/elasticsearch.yml

cluster.name: ussd-preprod-elastic
node.name: primary-node
network.host: <PRIMARY_NODE_IP>
discovery.seed_hosts: ["<PRIMARY_NODE_IP>", "<REPLICA_NODE_IP>"]
cluster.initial_master_nodes: ["primary-node"]
node.master: true
node.data: true
index.number_of_replicas: 1

node2
etc/elasticsearch/elasticsearch.yml

cluster.name: ussd-preprod-elastic
node.name: replica-node
network.host: <REPLICA_NODE_IP>
discovery.seed_hosts: ["<PRIMARY_NODE_IP>", "<REPLICA_NODE_IP>"]
cluster.initial_master_nodes: ["primary-node"]
node.master: false
node.data: true

sudo systemctl restart elasticsearch

frontend elasticsearch_frontend
    bind *:9200
    mode http
    default_backend elasticsearch_backend

backend elasticsearch_backend
    mode http
    balance roundrobin
    option httpchk GET /_cluster/health
    server node1 node-1-ip:9200 check
    server node2 node-2-ip:9200 check
    server node3 node-3-ip:9200 check