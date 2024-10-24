# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.synced_folder "files", "/etc/files"
  config.ssh.insert_key = false

  # Master DNS
  config.vm.define "tierra" do |tierra|
    tierra.vm.hostname = "tierra.sistema.test"
    tierra.vm.network "private_network", ip: "192.168.57.103"
    tierra.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y bind9 dnsutils
    SHELL
    tierra.vm.provision "shell", inline: <<-SHELL
      cp -v /files/named /etc/default
      cp -v /files/named.conf.options /etc/bind
      cp -v /files/named.conf.local /etc/bind
      cp -v /files/solarsystem.es.dns /var/lib
      cp -v /files/solarsystem.es.rev /var/lib
    SHELL
  end #master

  # Slave DNS
  config.vm.define "venus" do |venus|
    venus.vm.hostname = "venus.sistema.test"
    venus.vm.network "private_network", ip: "192.168.57.102"
    venus.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y dnsutils
    SHELL
  end #slave

end