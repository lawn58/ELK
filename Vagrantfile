  
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :"web" => {
    :box_name => "centos/7",
    :ip_addr => "172.20.10.50",
    :memory => "768",
  },
  :"elk" => {
    :box_name => "centos/7",
    :ip_addr => "172.20.10.52",
    :memory => "2048",
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s
      box.vm.network "private_network", ip: boxconfig[:ip_addr]
      box.vm.provider "virtualbox" do |vb|
        vb.name = boxname.to_s
        vb.memory = boxconfig[:memory]
        end
    end
  end
end
