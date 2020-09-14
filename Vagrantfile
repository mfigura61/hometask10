# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {
    :lvm => {
        :box_name => "centos/7",
        :box_version => "2004.01",
        :ip_addr => '192.168.11.101',
        :disks => {
            :sata1 => {
                :dfile => './sata1.vdi',
                :size => 4096, # Megabytes
                :port => 1
            },
        }
    },
}

Vagrant.configure("2") do |config|

    config.vm.box_version = "2004.01"
    MACHINES.each do |boxname, boxconfig|
  
        config.vm.define boxname do |box|
  
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
            box.vm.network "private_network", ip: boxconfig[:ip_addr]
  
            box.vm.provider :virtualbox do |vb|
                    vb.customize ["modifyvm", :id, "--memory", "512"]
                    needsController = false
            boxconfig[:disks].each do |dname, dconf|
                unless File.exist?(dconf[:dfile])
                  vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                                  needsController =  true
                            end
  
            end
                    if needsController == true
                       vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                       boxconfig[:disks].each do |dname, dconf|
                           vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
                       end
                    end
            end
  
        box.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh
            cp ~vagrant/.ssh/auth* ~root/.ssh
          SHELL
        box.vm.provision "shell", path: "watchlog.sh"
        box.vm.provision "shell", path: "fcgi.sh"
        box.vm.provision "shell", path: "httpd.sh"
  
        end
    end
  end
