global
	daemon
	maxconn 256


defaults
	mode tcp
	timeout connect 5000ms
	timeout client 50000ms
	timeout server 50000ms


frontend http
	bind :8080
	default_backend stats


backend stats
	mode http
	stats enable

	stats enable
	stats uri /
	stats refresh 1s
	stats show-legends
	stats admin if TRUE


frontend redis-read
    bind *:6379
    default_backend redis-online


frontend redis-write
	bind *:6380
	default_backend redis-primary


backend redis-primary
	mode tcp
	balance first
	option tcp-check

	tcp-check send info\ replication\r\n
	tcp-check expect string role:master

	server redis-01:192.168.1.11:6379 192.168.1.11:6379 maxconn 1024 check inter 1s
	server redis-02:192.168.1.12:6379 192.168.1.12:6379 maxconn 1024 check inter 1s


backend redis-online
	mode tcp
	balance roundrobin
	option tcp-check

	tcp-check send PING\r\n
	tcp-check expect string +PONG

	server redis-01:192.168.1.11:6379 192.168.1.11:6379 maxconn 1024 check inter 1s
	server redis-02:192.168.1.12:6379 192.168.1.12:6379 maxconn 1024 check inter 1s