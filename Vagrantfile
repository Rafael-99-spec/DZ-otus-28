# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
 :inetRouter => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.1', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "router-net"} # intnet это vlan
               ]
  },

 :inetRouter2 => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.2', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "router-net"}
               ]
  },

  :centralRouter => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.3', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "router-net"},
                   {ip: '192.168.0.1', adapter: 3, netmask: "255.255.255.240", virtualbox__intnet: "central-net"}
                ]
  },
  
  :centralServer => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.0.2', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "central-net"}
                ]
  }

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s

        config.vm.provider "virtualbox" do |v|
          v.memory = 256
        end

        boxconfig[:net].each do |ipconf|
          box.vm.network "private_network", ipconf
        end
        
        if boxconfig.key?(:public)
          box.vm.network "public_network", boxconfig[:public]
        end

        box.vm.provision "shell", inline: <<-SHELL
          mkdir -p ~root/.ssh
                cp ~vagrant/.ssh/auth* ~root/.ssh
        SHELL
        
        case boxname.to_s
        when "inetRouter"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            
            sudo yum install -y epel-release; sudo yum install -y wget libpcap*; wget http://li.nux.ro/download/nux/misc/el7/x86_64//knock-server-0.7-1.el7.nux.x86_64.rpm -P /home/vagrant; sudo yum localinstall -y /home/vagrant/knock-server-0.7-1.el7.nux.x86_64.rpm; #sudo yum localinstall -y knock-server-0.7-1.el7.nux.x86_64.rpm; sudo rpm -ivh /home/vagrant/knock-server-0.7-1.el7.nux.x86_64.rpm
            
            sudo bash -c 'echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf'; sudo sysctl -p
            
            sudo yum install -y iptables-services; sudo systemctl enable iptables && sudo systemctl start iptables;
            
            sudo iptables -P INPUT ACCEPT
            sudo iptables -P FORWARD ACCEPT
            sudo iptables -P OUTPUT ACCEPT
            sudo iptables -t nat -F
            sudo iptables -t mangle -F
            sudo iptables -F
            sudo iptables -X
            
            sudo iptables -t nat -A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE; 
            
            sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT; sudo iptables -A INPUT -p tcp --dport 22 -j REJECT; sudo service iptables save
            
            sudo bash -c 'echo "192.168.0.0/16 via 192.168.255.2 dev eth1" > /etc/sysconfig/network-scripts/route-eth1';
            
            mv /vagrant/files/knockd-inetRouter.conf /etc/knockd.conf; mv /vagrant/files/knockd-sysconfig /etc/sysconfig/knockd
            sudo chown root:root /etc/knockd.conf; sudo chmod 600 /etc/knockd.conf
            sudo chown root:root /etc/sysconfig/knockd; sudo chmod 644 /etc/sysconfig/knockd
            
            echo "vagrant:vagrant" | chpasswd
            sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
            systemctl restart sshd
            SHELL

          box.vm.provision "shell", path: "./files/sed.sh"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            sudo service knockd start
            sudo systemctl enable knockd.service
            sudo reboot
            SHELL
        when "inetRouter2"

        box.vm.network "forwarded_port", guest: 8080, host: 1234, host_ip: "127.0.0.1", id: "nginx"
          box.vm.provision "shell", run: "always", inline: <<-SHELL

            sudo bash -c 'echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf'; sudo sysctl -p

            sudo yum install -y iptables-services; sudo systemctl enable iptables && sudo systemctl start iptables;
            
            sudo iptables -P INPUT ACCEPT
            sudo iptables -P FORWARD ACCEPT
            sudo iptables -P OUTPUT ACCEPT
            sudo iptables -t nat -F
            sudo iptables -t mangle -F
            sudo iptables -F
            sudo iptables -X
            sudo iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.0.2:80
            sudo iptables -t nat -A POSTROUTING --destination 192.168.0.2/32 -j SNAT --to-source 192.168.255.2
            sudo service iptables save
            echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0
            echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
            sudo bash -c 'echo "192.168.0.0/16 via 192.168.255.3 dev eth1" > /etc/sysconfig/network-scripts/route-eth1'
            sudo reboot
            SHELL
        when "centralRouter"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            sudo yum install -y epel-release; sudo yum install -y nmap
            sudo bash -c 'echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf'; sudo sysctl -p
            echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0 
            echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
            sudo reboot
            SHELL
        when "centralServer"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            sudo yum install -y epel-release; sudo yum install -y nginx; sudo systemctl enable nginx; sudo systemctl start nginx
            echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0 
            echo "GATEWAY=192.168.0.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
            sudo reboot
            SHELL
        
        end

      end

  end
  
  
end
