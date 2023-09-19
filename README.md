## Note ##
This setup is done on **CentOS 7** using **Vagrant** and **Hyper-V** as a hypervisor for the virtual machines. All the configuration for installing and configuring the: **httpd**(Apache), **HAProxy**, **keepalived**(keepalived) is in the **Vagrantfile**.

## Explanation ##
This document describes how to set up a two-node load balancer in an active/passive configuration 
with HAProxy and heartbeat monitoring the state of load balancers.

The load balancer acts between the user and two(or more) Apache web servers that hold the same content.
The load balancer passes the requests to the web servers and also checks their health. If one of them is down, all requests will automatically be redirected
to the remaining web servers. 

In addition to that for another layer of redundancy, the two load balancer nodes are configured with a **virtual IP** shared between and monitor each other using a heartbeat. 
If the master fails, the slave becomes the master - users won't notice any disruption of the service.

	            +-----------------+
	            |  192.168.0.110  |
	            |   Floating IP   |
	            +--------+--------+
	                     |
	         +-----------------------+
	         |                       |
	+--------+--------+     +--------+--------+
	|  192.168.0.100  |     |  192.168.0.101  |
	|    HAProxy 1    |     |    HAProxy 2    | 
	+--------+--------+     +--------+--------+
	
	+--------+--------+     +--------+--------+
	|  192.168.0.102  |     |  192.168.0.103  |
	|  Web Server 1   |     |  Web Server 2   |
	+-----------------+     +-----------------+


### Apache SETUP on both Web Servers ###

	yum install httpd -y 
	systemctl enable --now httpd
	echo "Server 1" > /var/www/html/index.html
	echo "Server 2" > /var/www/html/index.html

### Firewall SETUP on both Web Servers ###

	systemctl enable --now firewalld
	systemctl start firewalld
	firewall-cmd --add-service=http --permanent
	firewall-cmd --add-service=https --permanent
	firewall-cmd --reload


### HAProxy Setup on both Load Balancers ###

Edit the configuration file for haproxy located in: /etc/haproxy/haproxy.conf.

	frontend http-balance
	bind *:80
	mode http
	stats enable
	stats auth admin:123
	stats uri /stats
	stats refresh 10s
	stats admin if LOCALHOST
	default_backend web-servers
	backend web-servers
	mode http
	balance roundrobin
	server server1 192.168.0.102:80 check
	server server2 192.168.0.103:80 check


### Allow IP Forwarding and non-local IP binding ###

Edit the **/etc/sysctl.conf** file to allow IP forwarding and non-local IP binding which will allow the servers to serve/bind to the virtual IP.

	net.ipv4.ip_forward = 1
	net.ipv4.ip_nonlocal_bind = 1
 
For the changes to take place run the below line in the terminal.
 
	 sysctl -p /etc/sysctl.conf

### Keepalived Setup on both Load Balancers ###

Edit the config file located in: **/etc/keepalived/keepalived.conf**

*** MASTER ***

vrrp_script chk_haproxy {
	script "killall -0 haproxy"
	interval 2
	weight 2
}

vrrp_instance VI_1 {
	state MASTER
	interface eth0
	virtual_router_id 51
	priority 150
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	unicast_src_ip 192.168.0.100
	unicast_peer {
		192.168.0.101
}

virtual_ipaddress {
	192.168.0.110
}
track_script {
chk_haproxy
}
}


*** SLAVE ***

vrrp_script chk_haproxy {
	script "killall -0 haproxy"
	interval 2
	weight 2
}

vrrp_instance VI_1 {
	state MASTER
	interface eth0
	virtual_router_id 51
	priority 140
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	unicast_src_ip 192.168.0.101
	unicast_peer {
		192.168.0.100
}

virtual_ipaddress {
	192.168.0.110
}
track_script {
chk_haproxy
}
}












