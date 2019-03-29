---
title: "Generate OpenStack TripleO Inventory with Ansible"
tags: ansible openstack
---
On a daily basis, I interact with many OpenStack environments.
Some of them I'm deploying from scratch and some of them I get already deployed.
Most of the time I need to perform some tasks on the Overcloud nodes (Controllers/Computes/etc...).
As I'm using `Ansible` for these kinds of tasks, I need an inventory to be able to interact with the overcloud nodes.
Here, the role I wrote comes to play.

**Note** - The role generates the required inventory only for the [TripleO][tripleo] based deployment.

### What the role does
The [openstack-tripleo-inventory][openstack-tripleo-inventory] role generates the inventory file for the overcloud nodes based on the few provided inputs.

### Structure
At the end of the play, the following inventory structure will appear.
```
|-> inventory file (soft link to the last environment)
|-> ansible.ssh.config (soft link to the last environment)
|
|-> environments
    |
    |-> First environment
    |     |-> inventory file
    |     |-> ansible.ssh.config file
    |     |-> ssh keys
    |
    |-> Second environment
    |     |-> inventory file
    |     |-> ansible.ssh.config file
    |     |-> ssh keys
```

<ins>Inventory file</ins> - Contains the nodes details like - ip address, username, ssh key path, etc...  
<ins>ansible.ssh.config</ins> - Contains ssh details for the connection to the Overcloud nodes over the Undercloud.
The file used by the inventory, but could be used separately for the connection.
```
ssh -F ansible.ssh.config controller-0
```

### Deployments types
Before diving to the role details, I would like to mention the types of deployments I work with.
* Virt
* Hybrid
* Baremetal

<ins>**Virt**</ins> - This is a single node, all-in-one deployment. Used mostly for development and simple testing.
The Undercloud and all the Overcloud nodes are virtual and reside on a single node which acts as a hypervisor.
<br>
<br>
<ins>**Hybrid**</ins> - In a hybrid deployment, Undercloud and some of the overcloud nodes are virtual and resides on a
single node which acts as hypervisor and other nodes are baremetal.
For example, Undercloud and Controllers are virtual and Compute nodes are physical.  
The importance of such deployment is that the virtual servers must be exposed to the real network to be able to interact with the baremetal nodes.
<br>
<br>
<ins>**Baremetal**</ins> - All the nodes (Undercloud, Controllers, Computes, etc...) are baremetal.
{: .notice--info}

\* In another post I will dive into the structure of the hybrid environment.
{: .notice--info}

### Play flow
User input to the role requires just one single host details.  
* In case of the `virt` or `hybrid` environment, provide the details of the **hypervisor** host.  
* In case of the `baremetal` environment, provide the details of the **Undercloud** host.

1. Connection established to the provided host (depends on the environment).
2. * In case, the environment is baremetal, the node is added to the inventory as undercloud.
   * In case, the environment is virt/hybrid, the following steps performed:
      * Hypervisor node added to the inventory as a hypervisor.
      * Play search for the Undercloud node.  
        The name of the Undercloud should be - undercloud or undercloud-0 (TripleO convention).
      * When found, the Undercloud node is added to the inventory as undercloud.
3. If the password provided instead of an ssh key, the ssh key is generated. More details in the next section.
4. Undercloud node facts gathered and virtualenv with all the requirements are installed.
5. Undercloud SSH key fetched to the host that executes the play. The key will be used for the connection to the Overcloud nodes.
6. Overcloud nodes data gathered.
7. Inventory and ansible.ssh.config file generated with the Overcloud nodes details.

### Authentication method
The play could be executed in two available scenarios:
1. The public key of your ssh key already located on the Undercloud/Hypervisor host.  
   The play will generate three files:
      * inventory
      * ansible.ssh.config
      * id_rsa_overcloud
2. You have only the password of the Undercloud/Hypervisor host.  
   The play will generate five files:
      * inventory
      * ansible.ssh.config
      * id_rsa_overcloud
      * id_rsa_undercloud
      * id_rsa_undercloud.pub

***
### Mandatory play variables
Type of the environment.  
Allowed variables: ['baremetal', 'virt']  
```yaml
setup_type: baremetal
```

Undercloud host name.  
```yaml
host: hostname
```

The Undercloud ssh private key file path.  
Use this variable in case ssh key already exists on the host. Otherwise, use the `ssh_pass` variable.    
**One of two parameters should be provided:** ssh_key or ssh_pass.
```yaml
ssh_key: /path/to/the/ssh_key
```

Specify the password for the Undercloud host.  
Use this variable in case there is no public key on the Undercloud host.  
**One of two parameters should be provided:** ssh_key or ssh_pass.
```yaml
ssh_pass: pass
```

Other variables are optional and could be viewed on the role [README][openstack-tripleo-inventory-readme] file.

***
### Examples
The example of running the TripleO Inventory playbook.  
With SSH key file for baremetal environment:
```yaml
ansible-playbook playbooks/tripleo_inventory.yml -e host=undercloud-host-fqdn/ip -e ssh_key=/path/to/ssh/private/file -e setup_type=baremetal
```
With SSH key file for hybrid or virt environment:
```yaml
ansible-playbook playbooks/tripleo_inventory.yml -e host=undercloud-host-fqdn/ip -e user=root -e ssh_key=/path/to/ssh/private/file -e setup_type=virt
```
With password:
```yaml
ansible-playbook playbooks/tripleo_inventory.yml -e host=undercloud-host-fqdn/ip -e user=root -e ssh_pass=undercloud_password
```

***
The role is reachable on the [GitHub][openstack-tripleo-inventory] or [Ansible Galaxy][openstack-tripleo-invenotry-galaxy].

[openstack-tripleo-inventory]: https://github.com/MaxBab/openstack-tripleo-inventory
[openstack-tripleo-inventory-readme]: https://github.com/MaxBab/openstack-tripleo-inventory/blob/master/README.md
[openstack-tripleo-invenotry-galaxy]: https://galaxy.ansible.com/maxbab/openstack-tripleo-inventory
[tripleo]: https://docs.openstack.org/tripleo-docs/latest/
