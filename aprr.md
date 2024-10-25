frontend backoffice_front
    bind *:30030
    default_backend backoffice_backend

backend backoffice_backend
    mode http
    balance roundrobin
    server kube-worker-1 10.59.150.72:30030 check
		server kube-worker-2 10.59.150.73:30030 check
		server kube-worker-3 10.59.150.74:30030 check
		server kube-worker-4 10.59.150.75:30030 check
		server kube-worker-5 10.59.150.76:30030 check
		server kube-worker-6 10.59.150.77:30030 check
		server kube-worker-7 10.59.150.78:30030 check

frontend openapi_front
    bind *:30020
    default_backend openapi_backend

backend openapi_backend
    mode http
    balance roundrobin
    server kube-worker-1 10.59.150.72:30020 check
		server kube-worker-2 10.59.150.73:30020 check
		server kube-worker-3 10.59.150.74:30020 check
		server kube-worker-4 10.59.150.75:30020 check
		server kube-worker-5 10.59.150.76:30020 check
		server kube-worker-6 10.59.150.77:30020 check
		server kube-worker-7 10.59.150.78:30020 check

frontend api_front
    bind *:30010
    default_backend api_backend

backend api_backend
    mode http
    balance roundrobin
    server kube-worker-1 10.59.150.72:30010 check
		server kube-worker-2 10.59.150.73:30010 check
		server kube-worker-3 10.59.150.74:30010 check
		server kube-worker-4 10.59.150.75:30010 check
		server kube-worker-5 10.59.150.76:30010 check
		server kube-worker-6 10.59.150.77:30010 check
		server kube-worker-7 10.59.150.78:30010 check
