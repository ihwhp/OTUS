# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'open3'
require 'fileutils'


Vagrant.configure("2") do |config|
  config.vm.boot_timeout = 420
  config.vm.box = "centos/7"
  config.vm.box_version = "1804.02"
  config.vm.define "boot"
  config.vm.host_name = 'boot'
  config.vm.network :private_network, ip: "192.168.56.60"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
	vb.cpus = "2"
    vb.gui = true
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  config.vm.provision "shell", inline: <<-SHELL
	      mkdir -p ~root/.ssh
              cp ~vagrant/.ssh/auth* ~root/.ssh
	      yum install -y nano
	SHELL

end
