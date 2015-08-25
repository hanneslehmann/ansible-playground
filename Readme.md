# Provisioning a test cluster for experimenting with Ansible

For PoC purposes I wanted to setup a small cluster to test configuration management using Ansible. Therefore I created a Vagrant file, plus additional setup files for Ansible to start testing (nearly) immediately.

Just clone the repository, run "vagrant up", wait some minutes to let Vagrant create 3 boxes:

- The box with ansible to manage the other boxes (host0)
- The boxes which we will use to configure (host1 + host2)

## Starting Vagrant

- Starts 1 box with ansible to manage the other boxes (host0)
- Starts 2 boxes which we will use to configure (host1 + host2)

The Vagrant file:
```bash
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
```

Follow along to see what happens during provisioning!

## Ansible

It will install Ansible from source, you can copy and paste the commandos into a shell script and execute it as root:

```bash
#!/usr/bin/env bash
apt-get update
apt-get install python-pip python-dev git -y
pip install PyYAML jinja2 paramiko
cd /tmp && git clone https://github.com/ansible/ansible.git
cd /tmp/ansible && git submodule update --init --recursive
cd /tmp/ansible && make install
mkdir /etc/ansible
cp /tmp/ansible/examples/hosts /etc/ansible/
```

After installation validate installation by checking version by

```bash
ansible --version
```

For further verification, you need to create a file called "ansible.cfg" in the directory where you are actually are going to issue the commands. My ansible.cfg looks like this (and should be placed for Vagrant into sub-dir "data"):
```bash
[defaults]
hostfile = inventory
remote_user = vagrant
private_key_file = priv.key
host_key_checking = False
```

The path to the private key can be obtained by issueing the vagrant command:
```bash
vagrant ssh-config
```

I justed copy & pasted the content of the private key provided by Vagrant into the mentioned file in the same directory. Dont't forget to change the priviliges of the key file to 600 ("chmod 600 priv.key")

Additionally you need a file which is referenced in the config file (and should be placed for Vagrant into sub-dir "data"): the inventory (basically a file containing the IP / SSH ports of the clients you want to manage):
```bash
host1 ansible_ssh_host=172.28.128.11 ansible_ssh_port=22
host2 ansible_ssh_host=172.28.128.12 ansible_ssh_port=22
```

In my case I have set up 2 virtual boxes, they've got the above mentioned host-only IP adresses.

So, finally the content of the directory looks like this:
```bash
vagrant@host0:~/ansible_testing$ ll
-rw-r--r-- 1 vagrant vagrant  160 Aug 25 09:00 ansible.cfg
-rw-r--r-- 1 vagrant vagrant  116 Aug 25 09:00 inventory
-rw------- 1 vagrant vagrant 1676 Aug 25 08:50 priv.key
```

Run the first test:
```bash
vagrant@host0:~/ansible_testing$ ansible all -m ping
host1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
host2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```


For testing you might need to manually ssh into your machines, to do so the ssh config file should look something like this (I am using cygwin on windows to ssh):
```bash
Host host0
HostName 172.28.128.10
Port 22
IdentityFile    /cygdrive/c/Users/Name/.vagrant.d/insecure_private_key
User vagrant
```


Install nginx on every machine, by sending a one liner:
```bash
cd ansible_testing
ansible all -s -m apt -a 'pkg=nginx state=installed update_cache=true'
```

Better to use Playbooks, so you can reproduce complex commands and source control them as well:

Sample Playbook to install nginx
```yml
- tasks:
   - name: Install Nginx
     apt: pkg=nginx state=installed update_cache=true
```

Then run following command to install on all machines defined in the inventory:
```bash
ansible-playbook -s nginx.yml
```

...or just run it on one host (e.g. host1):
```bash
vagrant@host0:~/ansible_testing$ ansible-playbook -s nginx.yml --limit host1

PLAY ***************************************************************************

TASK [setup] *******************************************************************
ok: [host1]

TASK [Install Nginx] ***********************************************************
ok: [host1]

PLAY RECAP *********************************************************************
host1                      : ok=2    changed=0    unreachable=0    failed=0
```

Some further ideas:
- to improve performance in running apt commandos you might want to cache the packages locally (or a central cache server like "approx")