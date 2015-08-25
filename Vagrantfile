# -*- mode: ruby -*-
# vi: set ft=ruby :

# Check final ssh config with: "vagrant ssh-config"

hosts = {
  "host0" => "172.28.128.10",
  "host1" => "172.28.128.11",
  "host2" => "172.28.128.12"
}

Vagrant.configure(2) do |config|
  
  config.vm.box = "deb/jessie-amd64"
  config.ssh.insert_key = false
 
  config.vm.provider "virtualbox" do |vb|
    vb.cpus = 1
	vb.memory = 512
  end
  
  hosts.each do |name, ip|
    config.vm.define name do |machine|
	  machine.vm.hostname = "%s.example" % name
	  machine.vm.network :private_network, ip: ip
	  if name == "host0"
	     machine.vm.provision "shell", inline: "mkdir -p ~/ansible_test"
	     machine.vm.provision "file", source: File.join(File.expand_path("~/.vagrant.d"), 'insecure_private_key'), destination: "~/ansible_test/priv.key"
		 machine.vm.provision "shell", inline: "chmod 600 /home/vagrant/ansible_test/priv.key"
		 machine.vm.provision "shell", inline: "cp /vagrant/data/* /home/vagrant/ansible_test/"
		 machine.vm.provision "shell", inline: "sudo apt-get update && sudo apt-get install python-pip python-dev git -y"
		 machine.vm.provision "shell", inline: "sudo pip install PyYAML jinja2 paramiko"
		 machine.vm.provision "shell", inline: "cd /tmp && git clone https://github.com/ansible/ansible.git"
		 machine.vm.provision "shell", inline: "cd /tmp/ansible && git submodule update --init --recursive"
		 machine.vm.provision "shell", inline: "cd /tmp/ansible && make install"
		 machine.vm.provision "shell", inline: "sudo mkdir /etc/ansible && sudo cp /tmp/ansible/examples/hosts /etc/ansible/"
		 machine.vm.provision "shell", inline: "rm -rf /tmp/ansible"
		 # to test ansible, try ansible all -m ping 
	  end
	end
  end
end
