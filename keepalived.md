sudo dnf install keepalived -y

sudo mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
sudo nano /etc/keepalived/keepalived.conf

master
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass KcdIIcdnkwricds
    }
    virtual_ipaddress {
        192.168.1.100
    }
}

backup
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass mysecretpassword
    }
    virtual_ipaddress {
        192.168.1.100
    }
}

sudo systemctl enable keepalived
sudo systemctl restart keepalived
