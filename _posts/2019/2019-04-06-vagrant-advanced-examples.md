---
title: "Vagrant Advanced Examples"
tags: vagrant
toc: true
toc_label: "Table of Contents"
---
### Overview
When I started to work with [`Vagrant`][vagrant], I had a few requirements.
* Boot and set any number of virtual machines without a requirement to duplicate code, but be able to change the configuration for each machine.
* Integrate Ansible playbooks (my common used configuration tool) into the Vagrant flow.
* If required, define a custom ansible inventory file to be generated based on my configuration.
* When needed, use an external configuration file.

Below, you will find a few advanced examples of the scenarios I used.

The GitHub repository with the examples could be found [here][vagrant-examples-github].

The repository contains 4 examples of the Vagrantfile when each example is based on the previous and adds extra functionality.

***

### Example1
Contains the following configuration:
#### * Multiple servers creation
#### * Insert custom ssh public key to the vm

Full Vagrantfile:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

ssh_key                     = "~/.ssh/id_rsa"
box                         = "centos/7"

servers = [
  { :hostname => "server1", :ip => "10.10.10.10", :ram => 1024, :cpu => 2 },
  { :hostname => "client1", :ip => "10.10.10.11", :box => "ubuntu/xenial64" },
  { :hostname => "client2", :ip => "10.10.10.12", :port_guest => 80, :port_host => 8080 },
  { :hostname => "client3", :ip => "dhcp", :folder_guest => "/srv/website", :folder_host => "src/" }
]

Vagrant.configure(2) do |config|
  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = false

    # Resolve dynamic ip address (dhcp) of the host and guests for the /etc/hosts file.
    # https://github.com/devopsgroup-io/vagrant-hostmanager/issues/86
    servers.each do |server|
      if server[:ip] == "dhcp"
        cached_addresses = {}
        config.hostmanager.ip_resolver = proc do |vm, resolving_vm|
          if cached_addresses[vm.name].nil?
            if hostname = (vm.ssh_info && vm.ssh_info[:host])
              vm.communicate.execute("hostname -I | cut -d ' ' -f 2") do |type, contents|
                cached_addresses[vm.name] = contents.split("\n").first[/(\d+\.\d+\.\d+\.\d+)/, 1]
              end
            end
          end
          cached_addresses[vm.name]
        end
      end
    end
  end

  servers.each do |server|
    box_image = server[:box] ? server[:box] : box;
    config.vm.define server[:hostname] do |conf|
      conf.vm.box = box_image.to_s
      conf.vm.hostname = server[:hostname]

      net_config = {}
      if server[:ip] != "dhcp"
        net_config[:ip] = server[:ip]
        net_config[:netmask] = server[:netmask] || "255.255.255.0"
      else
        net_config[:type] = "dhcp"
      end
      conf.vm.network "private_network", net_config

      if !server[:port_guest].nil? && !server[:port_host].nil?
        conf.vm.network "forwarded_port", guest: server[:port_guest], host: server[:port_host]
      end

      if !server[:folder_guest].nil? && !server[:folder_host].nil?
        conf.vm.synced_folder server[:folder_host], server[:folder_guest]
      end

      cpu = server[:cpu] ? server[:cpu] : 1;
      memory = server[:ram] ? server[:ram] : 512;
      config.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--cpus", cpu.to_s]
        vb.customize ["modifyvm", :id, "--memory", memory.to_s]
      end

      config.ssh.private_key_path = ["~/.vagrant.d/insecure_private_key", ssh_key]
      config.ssh.insert_key = false
      config.vm.provision "file", source: ssh_key + ".pub", destination: "~/.ssh/authorized_keys"
    end
  end
end
```

Let's go over the parts of the file:

<ins>Set global variables</ins>  
SSH key and a box (vm image) that should be used for the servers.  
The `box` variable could be overridden for each VM.
```ruby
ssh_key                     = "~/.ssh/id_rsa"
box                         = "centos/7"
```

<ins>Define servers details</ins>  
Unlimited number of servers could be defined.  
Each server could have its own configuration options.

Mandatory variables:
- hostname
- ip (possible arguments - static ip or dhcp - "10.10.10.10"/"dhcp")

Optional variables, could be set to override the defaults for specific box:
- ram (default: 512)
- cpu (default: 1)
- box (default: defined above)
- port_guest and port_host (port forwarding - not defined by default)
- folder_guest and folder_host (synced folders - not defined by default)

```ruby
servers = [
  { :hostname => "server1", :ip => "10.10.10.10", :ram => 1024, :cpu => 2 },
  { :hostname => "client1", :ip => "10.10.10.11", :box => "ubuntu/xenial64" },
  { :hostname => "client2", :ip => "10.10.10.12", :port_guest => 80, :port_host => 8080 },
  { :hostname => "client3", :ip => "dhcp", :folder_guest => "/srv/website", :folder_host => "src/" }
]
```

<ins>The vagrant-hostmanager plugin</ins>  
I'm using a `vagrant-hostmanager` plugin for the configuration of the /etc/hosts file on my local machine and within the guests.
The following section checks for the plugin existence and if exists, apply the configuration on the host and guests.  
Refer to the plugin [configuration][vagrant-hostmanager] for the sudo less configuration on the host.
```ruby
Vagrant.configure(2) do |config|
  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = false

    # Resolve dynamic ip address (dhcp) of the host and guests for the /etc/hosts file.
    # https://github.com/devopsgroup-io/vagrant-hostmanager/issues/86
    servers.each do |server|
      if server[:ip] == "dhcp"
        cached_addresses = {}
        config.hostmanager.ip_resolver = proc do |vm, resolving_vm|
          if cached_addresses[vm.name].nil?
            if hostname = (vm.ssh_info && vm.ssh_info[:host])
              vm.communicate.execute("hostname -I | cut -d ' ' -f 2") do |type, contents|
                cached_addresses[vm.name] = contents.split("\n").first[/(\d+\.\d+\.\d+\.\d+)/, 1]
              end
            end
          end
          cached_addresses[vm.name]
        end
      end
    end
end
```

<ins>Servers loop</ins>  
Loop through the list of the servers mentioned above and perform all the configuration mentioned in the next blocks.
```ruby
servers.each do |server|
```

<ins>Set a box and hostname for the vm</ins>  
Check if a custom box is defined for the specific server within the servers array and apply it.  
Otherwise, apply the global box variable.  
```ruby
box_image = server[:box] ? server[:box] : box;
config.vm.define server[:hostname] do |conf|
  conf.vm.box = box_image.to_s
  conf.vm.hostname = server[:hostname]
```

<ins>VM network configuration</ins>  
Set network options (static/dhcp) according to the options provided by the user.
```ruby
net_config = {}
if server[:ip] != "dhcp"
  net_config[:ip] = server[:ip]
  net_config[:netmask] = server[:netmask] || "255.255.255.0"
else
  net_config[:type] = "dhcp"
end
conf.vm.network "private_network", net_config
```

<ins>Set port forwarding and synced folders</ins>  
Set port forwarding and/or synced folders if defined.
```ruby
if !server[:port_guest].nil? && !server[:port_host].nil?
  conf.vm.network "forwarded_port", guest: server[:port_guest], host: server[:port_host]
end

if !server[:folder_guest].nil? && !server[:folder_host].nil?
  conf.vm.synced_folder server[:folder_host], server[:folder_guest]
end
```

<ins>Set CPU and Memory</ins>  
Checks for the custom cpu and/or ram defined for a specific server.
**Note**, that if options not provided, default values defined below, applied (cpu=1, ram=512).
```ruby
cpu = server[:cpu] ? server[:cpu] : 1;
memory = server[:ram] ? server[:ram] : 512;
config.vm.provider "virtualbox" do |vb|
  vb.customize ["modifyvm", :id, "--cpus", cpu.to_s]
  vb.customize ["modifyvm", :id, "--memory", memory.to_s]
end
```

<ins>Insert custom ssh public key to the vm</ins>  
Takes the ssh key provided within the global variables and copying the public key to the server.
```ruby
config.ssh.private_key_path = ["~/.vagrant.d/insecure_private_key", ssh_key]
config.ssh.insert_key = false
config.vm.provision "file", source: ssh_key + ".pub", destination: "~/.ssh/authorized_keys"
```

***

### Example 2
Contains the following configuration:
#### * Multiple servers creation
#### * Insert custom ssh public key to the vm
#### * Ansible provisioner
#### * Custom groups definition for Ansible
#### * Inventory generated by Vagrant. Custom groups are added to the generated inventory

Full Vagrantfile:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

ssh_key                     = "~/.ssh/id_rsa"
box                         = "centos/7"

servers = [
  { :hostname => "server1", :ip => "10.10.10.10", :ram => 1024, :cpu => 2, :group => "servers" },
  { :hostname => "client1", :ip => "10.10.10.11", :box => "ubuntu/xenial64", :group => "clients" },
  { :hostname => "client2", :ip => "10.10.10.12", :group => "clients", :port_guest => 80, :port_host => 8080 },
  { :hostname => "client3", :ip => "dhcp", :group => "clients", :folder_guest => "/srv/website", :folder_host => "src/" }
]

ansible_playbook = "playbook.yml"

Vagrant.configure(2) do |config|
  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = false

    # Resolve dynamic ip address (dhcp) of the host and guests for the /etc/hosts file.
    # https://github.com/devopsgroup-io/vagrant-hostmanager/issues/86
    servers.each do |server|
      if server[:ip] == "dhcp"
        cached_addresses = {}
        config.hostmanager.ip_resolver = proc do |vm, resolving_vm|
          if cached_addresses[vm.name].nil?
            if hostname = (vm.ssh_info && vm.ssh_info[:host])
              vm.communicate.execute("hostname -I | cut -d ' ' -f 2") do |type, contents|
                cached_addresses[vm.name] = contents.split("\n").first[/(\d+\.\d+\.\d+\.\d+)/, 1]
              end
            end
          end
          cached_addresses[vm.name]
        end
      end
    end
  end

  groups = {"all" => []}
  servers.each do |cfg|
    if ! groups.has_key?(cfg[:group])
      groups[cfg[:group]] = [cfg[:hostname]]
    else
      groups[cfg[:group]].push(cfg[:hostname])
    end
    groups["all"].push(cfg[:hostname])
  end

  servers.each_with_index do |server, index|
    box_image = server[:box] ? server[:box] : box;
    config.vm.define server[:hostname] do |conf|
      conf.vm.box = box_image.to_s
      conf.vm.hostname = server[:hostname]

      net_config = {}
      if server[:ip] != "dhcp"
        net_config[:ip] = server[:ip]
        net_config[:netmask] = server[:netmask] || "255.255.255.0"
      else
        net_config[:type] = "dhcp"
      end
      conf.vm.network "private_network", net_config

      if !server[:port_guest].nil? && !server[:port_host].nil?
        conf.vm.network "forwarded_port", guest: server[:port_guest], host: server[:port_host]
      end

      if !server[:folder_guest].nil? && !server[:folder_host].nil?
        conf.vm.synced_folder server[:folder_host], server[:folder_guest]
      end

      cpu = server[:cpu] ? server[:cpu] : 1;
      memory = server[:ram] ? server[:ram] : 512;
      config.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--cpus", cpu.to_s]
        vb.customize ["modifyvm", :id, "--memory", memory.to_s]
      end

      config.ssh.private_key_path = ["~/.vagrant.d/insecure_private_key", ssh_key]
      config.ssh.insert_key = false
      config.vm.provision "file", source: ssh_key + ".pub", destination: "~/.ssh/authorized_keys"

      # The ubuntu/xenial64 box is missing python. Install it for ansible provision.
      if box_image == "ubuntu/xenial64"
        config.vm.provision "shell" do |s|
          s.inline = "test -e /usr/bin/python || (apt-get -qqy update && apt-get install -qqy python-minimal)"
        end
      end

      if index == servers.size - 1
        if ansible_playbook != ""
          config.vm.provision :ansible do |ansible|
            ansible.verbose = "v"
            ansible.limit = "all"
            ansible.groups = groups
            ansible.playbook = ansible_playbook
          end
        end
      end
    end
  end
end
```

>As most of the file is identical to the example above, I will mention and explain just the differences.

<ins>Define servers details</ins>  
The "**:group**" variable has been added to the servers details.  
By using these groups, custom groups are created and could be used later for the ansible playbook execution.
```ruby
servers = [
  { :hostname => "server1", :ip => "10.10.10.10", :ram => 1024, :cpu => 2, :group => "servers" },
  { :hostname => "client1", :ip => "10.10.10.11", :box => "ubuntu/xenial64", :group => "clients" },
  { :hostname => "client2", :ip => "10.10.10.12", :group => "clients", :port_guest => 80, :port_host => 8080 },
  { :hostname => "client3", :ip => "dhcp", :group => "clients", :folder_guest => "/srv/website", :folder_host => "src/" }
]
```

<ins>Create ansible groups to be used during the play execution</ins>  
```ruby
groups = {"all" => []}
servers.each do |cfg|
  if ! groups.has_key?(cfg[:group])
    groups[cfg[:group]] = [cfg[:hostname]]
  else
    groups[cfg[:group]].push(cfg[:hostname])
  end
  groups["all"].push(cfg[:hostname])
end
```

<ins>Install python if "ubuntu/xenial64" box is used</ins>  
The "ubuntu/xenial64" box is missing python. Install it for ansible provision.
```ruby
if box_image == "ubuntu/xenial64"
  config.vm.provision "shell" do |s|
    s.inline = "test -e /usr/bin/python || (apt-get -qqy update && apt-get install -qqy python-minimal)"
  end
end
```

<ins>Provision nodes with ansible</ins>  
* Define ansible playbook that should be executed  
  The path to the playbook could be relative or absolute.
* Note, that the servers loop changed to gather index.  
  The index calculation is used by the ansible provision.  
  The play execution should start after all the servers are up and running.  
  The reason behind this is to execute the play once for all the machines and not for each machine separately.
* The last section prepare the ansible provision.  
  The index condition is checked at the first line.  
  * Set the verbosity of the play.
  * Set the play limit.
  * Set the groups that have been created for that play, according to the servers details.  
    Note, that the inventory for the ansible play, by default created by the vagrant and placed in the following location:
    .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory
  * Set the play to execute.

```ruby
(output omitted...)
ansible_playbook = "playbook.yml"
(output omitted...)

(output omitted...)
servers.each_with_index do |server, index|
(output omitted...)

(output omitted...)
if index == servers.size - 1
  if ansible_playbook != ""
    config.vm.provision :ansible do |ansible|
      ansible.verbose = "v"
      ansible.limit = "all"
      ansible.groups = groups
      ansible.playbook = ansible_playbook
    end
  end
end
```

***

### Example 3
Contains the following configuration:
#### * Multiple servers creation
#### * Insert custom ssh public key to the vm
#### * Ansible provisioner
#### * Custom groups definition for Ansible
#### * Inventory generated by Vagrant.
#### * Custom Ansible inventory creation (including custom groups)

Full Vagrantfile:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

ssh_key                     = "~/.ssh/id_rsa"
box                         = "centos/7"

servers = [
  { :hostname => "server1", :ip => "10.10.10.10", :ram => 1024, :cpu => 2, :group => "servers" },
  { :hostname => "client1", :ip => "10.10.10.11", :box => "ubuntu/xenial64", :group => "clients" },
  { :hostname => "client2", :ip => "10.10.10.12", :group => "clients", :port_guest => 80, :port_host => 8080 },
  { :hostname => "client3", :ip => "dhcp", :group => "clients", :folder_guest => "/srv/website", :folder_host => "src/" }
]

ansible_playbook = "playbook.yml"
ansible_inventory_path = "inventory/hosts"
ansible_user = "vagrant"

Vagrant.configure(2) do |config|
  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = false

    # Resolve dynamic ip address (dhcp) of the host and guests for the /etc/hosts file.
    # https://github.com/devopsgroup-io/vagrant-hostmanager/issues/86
    servers.each do |server|
      if server[:ip] == "dhcp"
        cached_addresses = {}
        config.hostmanager.ip_resolver = proc do |vm, resolving_vm|
          if cached_addresses[vm.name].nil?
            if hostname = (vm.ssh_info && vm.ssh_info[:host])
              vm.communicate.execute("hostname -I | cut -d ' ' -f 2") do |type, contents|
                cached_addresses[vm.name] = contents.split("\n").first[/(\d+\.\d+\.\d+\.\d+)/, 1]
              end
            end
          end
          cached_addresses[vm.name]
        end
      end
    end
  end

  if File.dirname(ansible_inventory_path) != "."
    Dir.mkdir(File.dirname(ansible_inventory_path)) unless Dir.exist?(File.dirname(ansible_inventory_path))
  end
  File.open(ansible_inventory_path, 'w') do |f|
    servers.each do |cfg|
      if cfg[:ip] != "dhcp"
        f.write "#{cfg[:hostname]} ansible_host=#{cfg[:ip]} "
        f.write "ansible_user=#{ansible_user} ansible_ssh_private_key_file=#{ssh_key}\n"
      else
        f.write "#{cfg[:hostname]} ansible_user=#{ansible_user} ansible_ssh_private_key_file=#{ssh_key}\n"
      end
    end
    f.write "\n"
    f.write "[all]\n"
    servers.each do |cfg|
      f.write "#{cfg[:hostname]}\n"
    end
    f.write "\n"
    servers.each do |cfg|
      f.write "[#{cfg[:group]}]\n"
      f.write "#{cfg[:hostname]}\n"
      f.write "\n"
    end
  end

  servers.each_with_index do |server, index|
    box_image = server[:box] ? server[:box] : box;
    config.vm.define server[:hostname] do |conf|
      conf.vm.box = box_image.to_s
      conf.vm.hostname = server[:hostname]

      net_config = {}
      if server[:ip] != "dhcp"
        net_config[:ip] = server[:ip]
        net_config[:netmask] = server[:netmask] || "255.255.255.0"
      else
        net_config[:type] = "dhcp"
      end
      conf.vm.network "private_network", net_config

      if !server[:port_guest].nil? && !server[:port_host].nil?
        conf.vm.network "forwarded_port", guest: server[:port_guest], host: server[:port_host]
      end

      if !server[:folder_guest].nil? && !server[:folder_host].nil?
        conf.vm.synced_folder server[:folder_host], server[:folder_guest]
      end

      cpu = server[:cpu] ? server[:cpu] : 1;
      memory = server[:ram] ? server[:ram] : 512;
      config.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--cpus", cpu.to_s]
        vb.customize ["modifyvm", :id, "--memory", memory.to_s]
      end

      config.ssh.private_key_path = ["~/.vagrant.d/insecure_private_key", ssh_key]
      config.ssh.insert_key = false
      config.vm.provision "file", source: ssh_key + ".pub", destination: "~/.ssh/authorized_keys"

      if box_image == "ubuntu/xenial64"
        config.vm.provision "shell" do |s|
          s.inline = "test -e /usr/bin/python || (apt-get -qqy update && apt-get install -qqy python-minimal)"
        end
      end

      if index == servers.size - 1
        if ansible_playbook != ""
          config.vm.provision :ansible do |ansible|
            ansible.inventory_path = ansible_inventory_path
            ansible.verbose = "v"
            ansible.limit = "all"
            ansible.playbook = ansible_playbook
          end
        end
      end
    end
  end
end
```

>As most of the file is identical to the example above, I will mention and explain just the differences.

<ins>Define ansible inventory path</ins>  
Ansible inventory. The path supports nested directories or a single file
```ruby
ansible_inventory_path = "inventory/hosts"
ansible_user = "vagrant"
```

<ins>Create custom ansible inventory</ins>  
The inventory will hold servers details and groups per each server.  
My main reason for the creation of custom inventory was the ability to hold an IP address that I provided within the servers list.  
By default, `vagrant` set the "127.0.0.1" as the `ansible_host` variable.

The path of the inventory file checked. If the path does not exist, it will be created.  
Created inventory will contain the following host details:
* ansible_host
* ansible_user (specified above)
* ansible_ssh_private_key_file
* group - "all"
* a group that has been mentioned for the server

>When using the dhcp allocation for the vm, [vagrant-hostmanager][vagrant-hostmanager] plugin is required to provide the resolving of the vm hostname.

```ruby
if File.dirname(ansible_inventory_path) != "."
  Dir.mkdir(File.dirname(ansible_inventory_path)) unless Dir.exist?(File.dirname(ansible_inventory_path))
end
File.open(ansible_inventory_path, 'w') do |f|
  servers.each do |cfg|
    if cfg[:ip] != "dhcp"
      f.write "#{cfg[:hostname]} ansible_host=#{cfg[:ip]} "
      f.write "ansible_user=#{ansible_user} ansible_ssh_private_key_file=#{ssh_key}\n"
    else
      f.write "#{cfg[:hostname]} ansible_user=#{ansible_user} ansible_ssh_private_key_file=#{ssh_key}\n"
    end
  end
  f.write "\n"
  f.write "[all]\n"
  servers.each do |cfg|
    f.write "#{cfg[:hostname]}\n"
  end
  f.write "\n"
  servers.each do |cfg|
    f.write "[#{cfg[:group]}]\n"
    f.write "#{cfg[:hostname]}\n"
    f.write "\n"
  end
end
```

<ins>Define custom created inventory path</ins>  
The ansible.inventory_path sets the path to the created inventory file.
```ruby
if index == servers.size - 1
  if ansible_playbook != ""
    config.vm.provision :ansible do |ansible|
      ansible.inventory_path = ansible_inventory_path
      ansible.verbose = "v"
      ansible.limit = "all"
      ansible.playbook = ansible_playbook
    end
  end
end
```

***

### Example 4
Contains the following configuration:
#### * Multiple servers creation
#### * Insert custom ssh public key to the vm
#### * Ansible provisioner
#### * Custom groups definition for Ansible
#### * Inventory generated by Vagrant.
#### * Custom Ansible inventory creation (including custom groups)
#### * Config yml based file shared between Vagrantfile and Ansible playbook. Holds variables used by both

Full Vagrantfile:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

ansible_playbook = "playbook.yml"
ansible_inventory_path = "inventory/hosts"
config_file = "vagrant.yml"

require 'yaml'
# https://blog.scottlowe.org/2016/01/14/improved-way-yaml-vagrant/
if !File.exists?(File.join(File.dirname(__FILE__), config_file))
  puts "The vagrant.yml file config is missing"
  abort
end
vars = YAML.load_file(File.join(File.dirname(__FILE__), config_file))

servers = vars['servers']
if !servers || servers.empty?
  puts "No servers defined in " + config_file
  abort
end

Vagrant.configure(2) do |config|
  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = false

    # Resolve dynamic ip address (dhcp) of the host and guests for the /etc/hosts file.
    # https://github.com/devopsgroup-io/vagrant-hostmanager/issues/86
    servers.each do |server|
      if server["ip"] == "dhcp"
        cached_addresses = {}
        config.hostmanager.ip_resolver = proc do |vm, resolving_vm|
          if cached_addresses[vm.name].nil?
            if hostname = (vm.ssh_info && vm.ssh_info[:host])
              vm.communicate.execute("hostname -I | cut -d ' ' -f 2") do |type, contents|
                cached_addresses[vm.name] = contents.split("\n").first[/(\d+\.\d+\.\d+\.\d+)/, 1]
              end
            end
          end
          cached_addresses[vm.name]
        end
      end
    end
  end

  if File.dirname(ansible_inventory_path) != "."
    Dir.mkdir(File.dirname(ansible_inventory_path)) unless Dir.exist?(File.dirname(ansible_inventory_path))
  end
  File.open(ansible_inventory_path, 'w') do |f|
    servers.each do |cfg|
      if cfg["ip"] != "dhcp"
        f.write "#{cfg["hostname"]} ansible_host=#{cfg["ip"]} "
        f.write "ansible_user=#{vars['ansible_user']} ansible_ssh_private_key_file=#{vars['ssh_key']}\n"
      else
        f.write "#{cfg["hostname"]} ansible_user=#{vars['ansible_user']} ansible_ssh_private_key_file=#{vars['ssh_key']}\n"
      end
    end
    f.write "\n"
    f.write "[all]\n"
    servers.each do |cfg|
      f.write "#{cfg["hostname"]}\n"
    end
    f.write "\n"
    servers.each do |cfg|
      f.write "[#{cfg["group"]}]\n"
      f.write "#{cfg["hostname"]}\n"
      f.write "\n"
    end
  end

  servers.each_with_index do |server, index|
    box_image = server["box"] ? server["box"] : vars['box'];
    config.vm.define server["hostname"] do |conf|
      conf.vm.box = box_image.to_s
      conf.vm.hostname = server["hostname"]

      net_config = {}
      if server["ip"] != "dhcp"
        net_config[:ip] = server["ip"]
        net_config[:netmask] = server["netmask"] || "255.255.255.0"
      else
        net_config[:type] = "dhcp"
      end
      conf.vm.network "private_network", net_config

      if !server["port_guest"].nil? && !server["port_host"].nil?
        conf.vm.network "forwarded_port", guest: server["port_guest"], host: server["port_host"]
      end

      if !server["folder_guest"].nil? && !server["folder_host"].nil?
        conf.vm.synced_folder server["folder_host"], server["folder_guest"]
      end

      cpu = server["cpu"] ? server["cpu"] : 1;
      memory = server["ram"] ? server["ram"] : 512;
      config.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--cpus", cpu.to_s]
        vb.customize ["modifyvm", :id, "--memory", memory.to_s]
      end

      config.ssh.private_key_path = ["~/.vagrant.d/insecure_private_key", vars['ssh_key']]
      config.ssh.insert_key = false
      config.vm.provision "file", source: vars['ssh_key'] + ".pub", destination: "~/.ssh/authorized_keys"

      if box_image == "ubuntu/xenial64"
        config.vm.provision "shell" do |s|
          s.inline = "test -e /usr/bin/python || (apt-get -qqy update && apt-get install -qqy python-minimal)"
        end
      end

      if index == servers.size - 1
        if ansible_playbook != ""
          config.vm.provision :ansible do |ansible|
            ansible.inventory_path = ansible_inventory_path
            ansible.verbose = "v"
            ansible.limit = "all"
            ansible.playbook = ansible_playbook
          end
        end
      end
    end
  end
end
```

>As most of the file is identical to the example above, I will mention and explain just the differences.

Full vagrant.yml
```yaml
servers:
  - hostname: server1
    ip: 10.10.10.10
    ram: 1024
    cpu: 2
    group: servers
  - hostname: client1
    ip: 10.10.10.11
    box: "ubuntu/xenial64"
    group: clients
  - hostname: client2
    ip: 10.10.10.12
    group: clients
    port_guest: 80
    port_host: 8080
  - hostname: client3
    ip: dhcp
    group: clients
    folder_guest: "/srv/website"
    folder_host: "src/"

box: "centos/7"
ssh_key: "~/.ssh/id_rsa"

ansible_user: vagrant
new_file_content: "This is the new content of the file"
test_file: "/tmp/test_file"
```

This example is pretty different from the example above.  
The variables provided to vagrant, including servers list, global and ansible playbook variables, provided in a separate yml file.

<ins>External config file</ins>  
Configuration file shared between Vagrantfile and Ansible playbook defined as "config_file" variable.
Yaml module imported and yml based config file is loaded.
The list of servers details loaded in a "servers" variable.
The "servers" variable tested as non-empty.
```ruby
config_file = "vagrant.yml"

require 'yaml'
# https://blog.scottlowe.org/2016/01/14/improved-way-yaml-vagrant/
if !File.exists?(File.join(File.dirname(__FILE__), config_file))
  puts "The vagrant.yml file config is missing"
  abort
end
vars = YAML.load_file(File.join(File.dirname(__FILE__), config_file))

servers = vars['servers']
# Validate servers config in config_file
if !servers || servers.empty?
  puts "No servers defined in " + config_file
  abort
end
```

<ins>Refer to the servers details</ins>  
When the details of the servers taken from the external file, the reference to the details are different.  
Below is a small part of the differences between the previous and current references.
```ruby
# Previous reference
if !server[:port_guest].nil? && !server[:port_host].nil?
  conf.vm.network "forwarded_port", guest: server[:port_guest], host: server[:port_host]
end

# Current reference
if !server["port_guest"].nil? && !server["port_host"].nil?
  conf.vm.network "forwarded_port", guest: server["port_guest"], host: server["port_host"]
end
```
In previous reference, the "port_guest" mentioned in the following type: `server[:port_guest]`.  
Current reference, mention the value in another way: `server["port_guest"]`.

The `vagrant.yml` file provided above has a simple yml file structure.

[vagrant]: https://www.vagrantup.com/
[vagrant-examples-github]: https://github.com/MaxBab/vagrant-examples
[vagrant-hostmanager]: https://github.com/devopsgroup-io/vagrant-hostmanager
