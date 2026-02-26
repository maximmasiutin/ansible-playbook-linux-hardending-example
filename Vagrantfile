servers=[
  {
    :hostname => "ubuntu",
    :ip => "192.168.0.2",
    :box => "ubuntu/focal64",
    :ssh_port => 2001
  },
  {
    :hostname => "centos",
    :ip => "192.168.0.3",
    :box => "centos/8",
    :ssh_port => 2002
  },
  {
    :hostname => "alpine",
    :ip => "192.168.0.4",
    :box => "generic/alpine38",
    :ssh_port => 2003
  },
  {
    :hostname => "gentoo",
    :ip => "192.168.0.5",
    :box => "generic/gentoo",
    :ssh_port => 2004
  }
]

RAM = 2048
CPU = 2
CONFIGURATION_VERSION = "2"

Vagrant.configure(CONFIGURATION_VERSION) do |config|
    config.vm.synced_folder '.', '/vagrant', disabled: true
    servers.each do |machine|
        config.vm.define machine[:hostname] do |node|
            node.vm.box = machine[:box]
            node.vm.hostname = machine[:hostname]
            # Securely provision the initial SSH key using Vagrant's defaults instead of 
            # hardcoding a copy of the host's private key. This prevents unintended 
            # access from the host and ensures portability across different environments.
            node.vm.network "private_network", ip: machine[:ip]
            node.vm.provider "virtualbox" do |vb|
                vb.memory = RAM
                vb.cpus = CPU
                # Restrict forwarded ports to localhost (127.0.0.1) to prevent 
                # unauthorized access from the external network while allowing
                # the host machine to still access the guest via SSH.
                config.vm.network :forwarded_port, guest: 22, host: machine[:ssh_port], id: "ssh", host_ip: "127.0.0.1"
            end
        end
    end
end
