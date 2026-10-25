# -*- mode: ruby -*-
# vim: set ft=ruby :
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :pxesrv => {
        :box_name => "ubuntu/jammy64",
        :vm_name => "pxesrv",
        :net => [   
                    #ip, adpter, netmask, virtualbox__intnet
					["192.168.20.10",  2, "255.255.255.0",   "pxenet"]
                ]
  },
  :pxecl => {
        :box_name => "ubuntu/jammy64",
        :vm_name => "pxecl",
        :net => [
				   ["192.168.20.11",  2, "255.255.255.0",  "pxenet"]
                ]
  },
  
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|
	config.vm.box_check_update = false
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.hostname = boxconfig[:vm_name]
      
      box.vm.provider "virtualbox" do |v|
        v.memory = 768
        v.cpus = 1
		v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
		if boxconfig[:vm_name] == "pxecl"
			v.customize ["modifyvm", :id, "--graphicscontroller", "vmsvga"]
			v.customize ["modifyvm", :id, "--vram", "32"]
			v.customize ["modifyvm", :id, "--boot1", "disk", "--boot2", "none", "--boot3", "none", "--boot4", "none"]
			v.customize ["modifyvm", :id, "--cableconnected1", "off"]
			v.memory = 2048
			v.cpus = 2
			box.vm.communicator = "none"
		end
       end
      boxconfig[:net].each do |ipconf|
		if ipconf[3].nil?
			box.vm.network("private_network", ip: ipconf[0], adapter: ipconf[1], netmask: ipconf[2])
		else
			box.vm.network("private_network", ip: ipconf[0], adapter: ipconf[1], netmask: ipconf[2], virtualbox__intnet: ipconf[3])
		end
      end

      if boxconfig.key?(:public)
        box.vm.network "public_network", boxconfig[:public]
      end
	  box.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh
        cp ~vagrant/.ssh/auth* ~root/.ssh
      SHELL
  	  if boxconfig[:vm_name] == "pxesrv"
        box.vm.provision "shell", inline: <<-SHELL
			apt-get update
			apt-get install -y dnsmasq
			apt-get install -y nginx
			apt-get install -y nfs-kernel-server
        SHELL
      end

    end
  end
end
