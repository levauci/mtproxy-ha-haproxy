Vagrant.configure("2") do |config|
  config.vm.box = "perk/ubuntu-2204-arm64"
  config.vm.provider "qemu" do |qemu|
    qemu.arch = "aarch64"
    qemu.machine = "virt"
    qemu.cpu = "max"
    qemu.memory = "4096"
    qemu.cpus = 2
    qemu.net_device = "virtio-net-pci"
  end
  config.vm.network "private_network", ip: "192.168.56.10"
  config.vm.hostname = "mtproxy-server"
  config.vm.provision "shell", inline: <<-SHELL
    sudo apt-get update
    sudo apt-get install -y curl git python3-pip
  SHELL
end
