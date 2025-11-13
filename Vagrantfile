# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "bento/debian-12"
  config.vm.provider "virtualbox" do |vb| 
      vb.gui = false # headless mode | without interfaz
      vb.memory = 2048
      vb.cpus = 2
  end # vb

  config.vm.define "dns" do |dns|
    dns.vm.hostname = "dns.msg.izv"
    dns.vm.network "private_network", ip: "192.168.58.10" 
    # dns.vm.network "private_network", ip: "192.168.57.10", virtualbox__intnet: "internal"
  end #dns

  config.vm.define "dhcp" do |dhcp|
    dhcp.vm.hostname = "dhcp.msg.izv"
    dhcp.vm.network "private_network", ip: "192.168.58.20" 
    # dhcp.vm.network "private_network", ip: "192.168.57.10", virtualbox__intnet: "internal"    
  end #dhcp

  config.vm.define "c1" do |c1|
    c1.vm.hostname = "cliente1"
    # c1.vm.network "private_network", type: "dhcp", virtualbox__intnet: "internal" #auto_config: false
  end #c1

  # Provisioning con Ansible
  #config.vm.provision "ansible" do |ansible|
  #  ansible.compatibility_mode = "auto"
  #  ansible.playbook = "playbooks/playbook-setup.yaml"
  end
end