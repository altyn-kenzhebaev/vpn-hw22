# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
    config.vm.box = "almalinux/9"
    
    config.vm.define "server" do |server|
      server.vm.hostname = "server.loc"
      server.vm.network "private_network", ip: "192.168.50.10"
      server.vm.provision "ansible" do |ansible|
        ansible.playbook = 'ansible/server.yml'
        ansible.compatibility_mode = "2.0"
      end
    end
    
    config.vm.define "client" do |client|
      client.vm.hostname = "client.loc"
      client.vm.network "private_network", ip: "192.168.50.20"
      client.vm.provision "ansible" do |ansible|
        ansible.playbook = 'ansible/client.yml'
        ansible.compatibility_mode = "2.0"
      end
    end

end