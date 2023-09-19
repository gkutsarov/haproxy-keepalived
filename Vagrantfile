# -*- mode: ruby -*-
# vi: set ft=ruby :

$haproxy_config = <<-SCRIPT
echo -e "frontend http-balance\nbind *:80\nmode http\nstats enable\nstats auth admin:123\nstats uri /stats\nstats refresh 10s\nstats admin if LOCALHOST\ndefault_backend web-servers\nbackend web-servers\nmode http\nbalance roundrobin\nserver server1 192.168.0.102:80 check\nserver server2 192.168.0.103:80 check" > /etc/haproxy/haproxy.cfg
SCRIPT

$keepalived_config_master = <<-SCRIPT
echo "vrrp_script chk_haproxy {
		script \"killall -0 haproxy\"
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
		192.168.0.110/24
}
track_script {
chk_haproxy
}
}" > /etc/keepalived/keepalived.conf
SCRIPT

$keepalived_config_slave = <<-SCRIPT
echo "vrrp_script chk_haproxy {
		script \"killall -0 haproxy\"
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
		192.168.0.110/24
}
track_script {
chk_haproxy
}
}" > /etc/keepalived/keepalived.conf
SCRIPT

Vagrant.configure("2") do |config|

  config.vm.define "proxy" do |proxy|
    proxy.vm.box = "centos/7"
	proxy.vm.hostname = "proxy"
	proxy.vm.network "public_network", bridge: "External", auto_config: false
	
	proxy.vm.provision "shell", run: "always", inline: <<-SHELL
		echo -e "TYPE=Ethernet\nDEVICE=eth0\nBOOTPROTO=static\nIPADDR=192.168.0.100\nNETMASK=255.255.255.0\nGATEWAY=192.168.0.1\nDNS1=192.168.0.1\nNAME=eth0\nUUID=00:15:5D:00:6E:14\nONBOOT=yes" > /etc/sysconfig/network-scripts/ifcfg-eth0
		grep -l '#PasswordAuthentication yes' /etc/ssh/sshd_config | xargs sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/g'
		shutdown -r 2
		yum install haproxy -y
		systemctl enable --now haproxy
		systemctl restart haproxy
		echo -e "net.ipv4.ip_forward = 1\nnet.ipv4.ip_nonlocal_bind = 1" > /etc/sysctl.conf
		sysctl -p /etc/sysctl.conf
		yum install keepalived -y
		systemctl enable --now keepalived
	SHELL
	
	proxy.vm.provision "shell", inline: $haproxy_config
	proxy.vm.provision "shell", inline: $keepalived_config_master
	proxy.vm.provision "shell", inline: <<-SHELL
		systemctl restart keepalived
	SHELL
	
	proxy.vm.provider "hyperv" do |h|
	  h.memory = "1024"
	  h.cpus = "1"
	  h.vmname = "proxy"
	end
  end
  
  config.vm.define "proxy1" do |proxy1|
    proxy1.vm.box = "centos/7"
	proxy1.vm.hostname = "proxy1"
	proxy1.vm.network "public_network", bridge: "External", auto_config: false
	
	proxy1.vm.provision "shell", run: "always", inline: <<-SHELL
		echo -e "TYPE=Ethernet\nDEVICE=eth0\nBOOTPROTO=static\nIPADDR=192.168.0.101\nNETMASK=255.255.255.0\nGATEWAY=192.168.0.1\nDNS1=192.168.0.1\nNAME=eth0\nUUID=00:15:5D:00:6E:14\nONBOOT=yes" > /etc/sysconfig/network-scripts/ifcfg-eth0
		grep -l '#PasswordAuthentication yes' /etc/ssh/sshd_config | xargs sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/g'
		shutdown -r 2
		yum install haproxy -y
		yum install keepalived -y
		systemctl enable --now haproxy
		echo -e "frontend http-balance\nbind *:80\nmode http\nstats enable\nstats auth admin:123\nstats uri /stats\nstats refresh 10s\nstats admin if LOCALHOST\ndefault_backend web-servers\nbackend web-servers\nmode http\nbalance roundrobin\nserver server1 192.168.0.102:80 check\nserver server2 192.168.0.103:80 check" > /etc/haproxy/haproxy.cfg
		systemctl restart haproxy
		echo -e "net.ipv4.ip_forward = 1\nnet.ipv4.ip_nonlocal_bind = 1" > /etc/sysctl.conf
		sysctl -p /etc/sysctl.conf
		yum install keepalived -y
		systemctl enable --now keepalived
	SHELL
	
	proxy1.vm.provision "shell", inline: $haproxy_config
	proxy1.vm.provision "shell", inline: $keepalived_config_slave
	proxy1.vm.provision "shell", inline: <<-SHELL
		systemctl restart keepalived
	SHELL
	
	proxy1.vm.provider "hyperv" do |h|
	  h.memory = "1024"
	  h.cpus = "1"
	  h.vmname = "proxy1"
	end
  end

  config.vm.define "server1" do |server1|
    server1.vm.box = "centos/7"
	server1.vm.hostname = "server1"
	server1.vm.network "public_network", bridge: "External", auto_config: false
	
	server1.vm.provision "shell", run: "always", inline: <<-SHELL
		echo -e "TYPE=Ethernet\nDEVICE=eth0\nBOOTPROTO=static\nIPADDR=192.168.0.102\nNETMASK=255.255.255.0\nGATEWAY=192.168.0.1\nDNS1=192.168.0.1\nNAME=eth0\nUUID=00:15:5D:00:6E:14\nONBOOT=yes" > /etc/sysconfig/network-scripts/ifcfg-eth0
		grep -l '#PasswordAuthentication yes' /etc/ssh/sshd_config | xargs sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/g'
		shutdown -r 1
		yum install httpd -y
		yum install keepalived -y
		systemctl enable --now httpd
		echo "Server 1" > /var/www/html/index.html
		systemctl enable --now firewalld
		systemctl start firewalld
		firewall-cmd --add-service=http --permanent
		firewall-cmd --add-service=https --permanent
		firewall-cmd --reload
	SHELL
  
	server1.vm.provider "hyperv" do |h|
	  h.memory = "1024"
	  h.cpus = "1"
	  h.vmname = "server1"
	end
  end

  config.vm.define "server2" do |server2|
    server2.vm.box = "centos/7"
	server2.vm.hostname = "server2"
	server2.vm.network "public_network", bridge: "External", auto_config: false
	
	server2.vm.provision "shell", run: "always", inline: <<-SHELL
		echo -e "TYPE=Ethernet\nDEVICE=eth0\nBOOTPROTO=static\nIPADDR=192.168.0.103\nNETMASK=255.255.255.0\nGATEWAY=192.168.0.1\nDNS1=192.168.0.1\nNAME=eth0\nUUID=00:15:5D:00:6E:14\nONBOOT=yes" > /etc/sysconfig/network-scripts/ifcfg-eth0
		grep -l '#PasswordAuthentication yes' /etc/ssh/sshd_config | xargs sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/g'
		shutdown -r 1
		yum install httpd -y
		systemctl enable --now httpd
		echo "Server 2" > /var/www/html/index.html
		systemctl enable --now firewalld
		systemctl start firewalld
		firewall-cmd --add-service=http --permanent
		firewall-cmd --add-service=https --permanent
		firewall-cmd --reload
	SHELL
	
	server2.vm.provider "hyperv" do |h|
	  h.memory = "1024"
	  h.cpus = "1"
	  h.vmname = "server2"
	end
  end
 end
