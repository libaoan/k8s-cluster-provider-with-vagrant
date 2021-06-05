# -*- mode: ruby -*-
# vi: set ft=ruby :

# on win10, you may need `vagrant plugin install vagrant-vbguest --plugin-version 0.21`"
# reference `https://www.dissmeyer.com/2020/02/11/issue-with-centos-7-vagrant-boxes-on-windows-10/`

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

# Require a recent version of vagrant otherwise some have reported errors setting host names on boxes
Vagrant.require_version ">= 2.2.16"

Vagrant.configure("2") do |config|
  config.vm.box_check_update = false
  config.vm.provider 'virtualbox' do |vb|
  vb.customize [ "guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 1000 ]
  end  
  # config.vm.synced_folder ".", "/vagrant", type: "virtualbox"
  $num_instances = 2
  # curl https://discovery.etcd.io/new?size=3
  $etcd_cluster = "node1=http://172.17.8.201:2380"

  (1..$num_instances).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.box = "centos/7"
      ## if you set box_url online, you should set `vm.box_version`
      # node.vm.box_version = "1804.02"
      ## load box on local
      node.vm.box_url = "./CentOS-7-x86_64-Vagrant-1804_02.VirtualBox.box"
      node.vm.hostname = "node#{i}"
      ip = "172.17.8.#{i+200}"
      node.vm.network "private_network", ip: ip
      node.vm.provider "virtualbox" do |vb|
        vb.memory = "4096"
        vb.cpus = 2
        vb.name = "node#{i}"
      end
      node.vm.provision "shell", path: "install.sh", args: [i, ip, $etcd_cluster]
    end
  end
end
