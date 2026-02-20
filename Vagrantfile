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

  config.vm.hostname = "mtproxy-server"

  config.ssh.connect_timeout = 120
  config.vm.boot_timeout = 600
  config.vm.graceful_halt_timeout = 120

  config.vm.network "forwarded_port", guest: 443,  host: 8443
  config.vm.network "forwarded_port", guest: 8404, host: 8404

  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y curl git python3-pip
  SHELL

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbooks/deploy.yml"
    ansible.compatibility_mode = "2.0"
    ansible.galaxy_role_file = "requirements.yml"
    ansible.galaxy_command = "ansible-galaxy collection install -r %{role_file} --force"
  end
end
