# -*- mode: ruby -*-
# vi: set ft=ruby :

CONSUL_BOOTSTRAP_CLOUD_CONFIG_PATH = File.join(File.dirname(__FILE__), "nocloud.consul-bootstrap.iso")
CONSUL_SERVER_CLOUD_CONFIG_PATH = File.join(File.dirname(__FILE__), "nocloud.consul-server.iso")
COMPUTE_CLOUD_CONFIG_PATH = File.join(File.dirname(__FILE__), "nocloud.compute.iso")

Vagrant.require_version ">= 1.9.0"

if ARGV[0] == "up"
  `make`
end

def create_vm config, host
    # set the host name
    config.vm.hostname = host["hostname"]

    # and the private IP address
    config.vm.network :private_network, ip: host["ip"]

    config.vm.box = "ubuntu/xenial64"

    # Forward ssh keys
    config.ssh.forward_agent = true

    # Disable SSH password for 16.04 - we'll add the insecure Vagrant key
    # (don't worry, it's just an example and gets replaced anyway)
    config.ssh.password = nil

    # To use your main public/private key pair, uncomment these lines:
    # config.ssh.private_key_path = File.expand_path("~/.ssh/id_rsa")
    # config.ssh.insert_key = false

    # Disable shared folders
    config.vm.synced_folder ".", "/vagrant", disabled: true

    config.vm.provider "virtualbox" do |vb|
        # Display the VirtualBox GUI when booting the machine
        vb.gui = false

        # Customize the amount of memory on the VM:
        vb.memory = host["memory"]
    end

    # Tweak virtualbox
    config.vm.provider :virtualbox do |vb|
        # Attach nocloud.iso to the virtual machine
        vb.customize [
            "storageattach", :id,
            "--storagectl", "SCSI",
            "--port", "1",
            "--type", "dvddrive",
            "--medium", host["iso"]
        ]
    end

    config.vm.provider "virtualbox" do |vb|
       vb.customize [ "modifyvm", :id, "--uart1", "0x3F8", "4" ]
       vb.customize [ "modifyvm", :id, "--uartmode1", "file", File.join(Dir.pwd, "console.#{host['hostname']}.log") ]
    end
end

Vagrant.configure(2) do |config|

    [
        {
            "hostname" => "bootstrap",
            "memory" => "512", #mb
            "ip" => "10.200.0.10",
            "iso" => CONSUL_BOOTSTRAP_CLOUD_CONFIG_PATH
        }
    ].each do |host|
        config.vm.define host["hostname"] do |config|
            create_vm(config, host)
        end
    end

    (1..5).each do |id|
        base_ip = 10
        host = {
            "hostname" => "consul-%02d" % [id],
            "memory" => "512", #mb
            "ip" => "10.200.0.%d" % [base_ip + id],
            "iso" => CONSUL_SERVER_CLOUD_CONFIG_PATH
        }
        config.vm.define host["hostname"] do |config|
            create_vm(config, host)
        end
    end

    (1..2).each do |id|
        base_ip = 20
        host = {
            "hostname" => "compute-%02d" % [id],
            "memory" => "1024", #mb
            "ip" => "10.200.0.%d" % [base_ip + id],
            "iso" => COMPUTE_CLOUD_CONFIG_PATH
        }
        config.vm.define host["hostname"] do |config|
            create_vm(config, host)
        end
    end
end
