## Note ##
This setup is done on CentOS 7 using Vagrant and Hyper-V as a hypervisor for the virtual machines. All the configuration for installing and configuring the: httpd(Apache), HAProxy, heartbeat(keepalived) is in the Vagrantfile.

## Explanation ##
This document describes how to set up a two-node load balancer in an active/passive configuration 
with HAProxy and heartbeat monitoring the state of load balancers.

The load balancer acts between the user and two(or more) Apache web servers that hold the same content.
The load balancer passes the requests to the web servers and also checks their health. If one of them is down, all requests will automatically be redirected
to the remaining web servers. 

In addition to that, the two load balancer nodes monitor each other using a heartbeat. 
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



Apache SETUP on both Web Servers ***

yum install httpd -y # Installing the Apache Web Server
systemctl enable --now httpd # Enable the Apache Web Server to start on boot
echo "Server 1" > /var/www/html/index.html # Adding a dummy content for the default website. Same is done on Server 2.

--------------------------------------------------
|  *** Firewall SETUP on both Web Servers ***       |
--------------------------------------------------
systemctl enable --now firewalld #Enable firewalld to start on boot
systemctl start firewalld #Starting firewalld
firewall-cmd --add-service=http --permanent #Adding http to the allowed services
firewall-cmd --add-service=https --permanent #Adding https to the allowed services
firewall-cmd --reload #Reloading the firewall for the changes to be applied.

--------------------------------------------------
|  *** HAProxy Setup on both Load Balancers ***  |
--------------------------------------------------
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

--------------------------------------------------
|*** Keepalived Setup on both Load Balancers *** |
--------------------------------------------------
yum install keepalived -y

Configure IP forwarding and non-local binding
To enable Keepalived service to forward network packets to the backend servers, you need to enable IP forwarding.
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf

Similarly, you need to enable HAProxy and Keepalived to bind to non-local IP address, that is to bind to the failover IP address (Floating IP).
echo "net.ipv4.ip_nonlocal_bind = 1" >> /etc/sysctl.conf

Reload sysctl settings;
sysctl -p

Configure Keepalived

vim /etc/keepalived/keepalived.conf

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
	priority 101
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	unicast_src_ip 192.168.0.110
	unicast_peer {
		192.168.0.111
}

virtual_ipaddress {
	192.168.0.120
}
track_script {
chk_haproxy
}
}

*** END MASTER ***

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
	priority 100
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	unicast_src_ip 192.168.0.111
	unicast_peer {
		192.168.0.110
}

virtual_ipaddress {
	192.168.0.120
}
track_script {
chk_haproxy
}
}

*** END SLAVE ***












