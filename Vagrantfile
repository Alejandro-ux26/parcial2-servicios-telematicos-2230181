Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.define "srv1" do |srv1|
    srv1.vm.hostname = "srv1-2230181"
    srv1.vm.network "private_network", ip: "192.168.50.3", virtualbox__intnet: "srv-link"
    srv1.vm.network "private_network", ip: "192.168.60.3"
    srv1.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
    end
  end

  config.vm.define "srv2" do |srv2|
    srv2.vm.hostname = "srv2-2230181"
    srv2.vm.network "private_network", ip: "192.168.50.2", virtualbox__intnet: "srv-link"
    srv2.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
    end
  end

  config.vm.define "cli" do |cli|
    cli.vm.hostname = "cli-2230181"
    cli.vm.network "private_network", ip: "192.168.60.10"
    cli.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
    end
  end
end