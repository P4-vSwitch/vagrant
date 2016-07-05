# -*- mode: ruby -*-
# vi: set ft=ruby :

$script = <<SCRIPT
    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get -y install git python-pip fuse libfuse-dev dh-autoreconf openssl libssl-dev cmake
SCRIPT

$switch_script = <<SWITCH_SCRIPT
    /vagrant/setup-switch.sh
SWITCH_SCRIPT

$moongen_script = <<MOONGEN_SCRIPT
    /vagrant/setup-moongen.sh
MOONGEN_SCRIPT

Vagrant.configure("2") do |config|

    # Configure switch, i.e., device under test (DUT)
    config.vm.define "switch" do |switch|
        switch.vm.box = "ubuntu-trusty64"

        switch.vm.network "private_network", ip: "172.16.0.10", netmask: "255.255.255.0", virtualbox__intnet: "gen-sw"
        switch.vm.network "private_network", ip: "172.16.0.11", netmask: "255.255.255.0", virtualbox__intnet: "sw-rcv"

        switch.vm.provider "virtualbox" do |virtualbox|
            # Customize the amount of memory on the VM:
            virtualbox.memory = "4096"
            virtualbox.cpus = "4"
            # Enable promiscuous mode
            virtualbox.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
            virtualbox.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
        end

        # Setup switch
        switch.vm.provision "shell", inline: $switch_script
    end

    # Configure generator
    config.vm.define "generator" do |generator|
        generator.vm.box = "ubuntu-trusty64"

        generator.vm.network "private_network", ip: "172.16.0.10", netmask: "255.255.255.0", virtualbox__intnet: "gen-sw"

        generator.vm.provider "virtualbox" do |virtualbox|
            # Customize the amount of memory on the VM:
            virtualbox.memory = "2048"
            virtualbox.cpus = "2"
        end

        # Setup generator
        generator.vm.provision "shell", inline: $moongen_script
    end

    # Configure receiver
    config.vm.define "receiver" do |receiver|
        receiver.vm.box = "ubuntu-trusty64"

        receiver.vm.network "private_network", ip: "172.16.0.11", netmask: "255.255.255.0", virtualbox__intnet: "sw-rcv"

        receiver.vm.provider "virtualbox" do |virtualbox|
            # Customize the amount of memory on the VM:
            virtualbox.memory = "2048"
            virtualbox.cpus = "2"
        end

        # Setup receiver
        receiver.vm.provision "shell", inline: $moongen_script
    end

    # Install essentials
    config.vm.provision "shell", inline: $script
end
