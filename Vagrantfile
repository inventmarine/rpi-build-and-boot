# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  # https://atlas.hashicorp.com/ubuntu/boxes/trusty64 [Official Ubuntu Server 14.04 LTS (Trusty Tahr) builds]
  config.vm.box = "ubuntu/trusty64"
  config.vm.provider "virtualbox" do |vb|
    config.vbguest.iso_path = "#{ENV['HOME']}/virtualbox-images/VBoxGuestAdditions_5.0.20.iso"
    vb.customize ["modifyvm", :id, "--memory", "4096"]
    vb.customize ["modifyvm", :id, "--cpus", "2"]    
  end
  # If you want to use this system to netboot Raspberry Pi, then uncomment this line
  # config.vm.network "public_network", bridge: 'ask', ip: "10.0.0.1"
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
  end
end

