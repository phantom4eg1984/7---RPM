Vagrant.configure("2") do |config|
     config.vm.box = "centos/stream8"
     config.vm.box_version = "20210210.0"

    config.vm.provider "virtualbox" do |v|
       v.memory = 256
       v.cpus = 1
 end

    config.vm.define "nginx" do |nginx|
    nginx.vm.network "private_network", ip: "192.168.56.90",
      virtualbox_intnet: "net1"
      nginx.vm.hostname = "nginxssl"
end
