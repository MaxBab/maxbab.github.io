---
title: "How I Used Complex Jinja2 Filters in Ansible"
tags: ansible jinja2
---
The [jinja2 ansible filters][jinja2_ansible_filters] could simplify our task when it comes to a need to
construct some data array or convert input data to a different output.
Recently I worked on some ansible tasks where the use of the filters was very handy.

### Story
When working with the OpenStack environment, different tasks should be accomplished, such as networks,
flavors, keypair and security groups creation, instances boot and much more.
For that purpose, I'm using an `ansible` role called 'openstack_tasks'.  

When creating all the network relates objects, I'm providing the following bulk of variables to the play:
{% highlight yaml linenos %}
networks:
  - name: 'access'
    physical_network: 'access'
    segmentation_id: '22'
    allocation_pool_start: '10.0.0.113'
    allocation_pool_end: '10.0.0.125'
    cidr: '10.0.0.112/28'
    enable_dhcp: true
    gateway_ip: '10.0.0.126'
    network_type: vlan
    shared: true
    external: true
    router_name: router1

  - name: 'dpdk-mgmt'
    allocation_pool_start: '10.10.110.100'
    allocation_pool_end: '10.10.110.200'
    cidr: '10.10.110.0/24'
    enable_dhcp: true
    gateway_ip: '10.10.110.254'
    network_type: vxlan
    external: false
    router_name: router1

  - name: 'sriov-1'
    allocation_pool_start: '40.0.0.100'
    allocation_pool_end: '40.0.0.200'
    physical_network: sriov-1
    cidr: '40.0.0.0/24'
    enable_dhcp: false
    gateway_ip: '40.0.0.254'
    network_type: vlan
    external: false
    router_name: router1

  - name: 'sriov-2'
    allocation_pool_start: '50.0.0.100'
    allocation_pool_end: '50.0.0.200'
    physical_network: sriov-2
    cidr: '50.0.0.0/24'
    enable_dhcp: false
    gateway_ip: '50.0.0.254'
    network_type: vlan
    external: false
    router_name: router2
{% endhighlight %}

The play takes these variables input and provision related network objects.
All the networks, subnets and routers created when the tasks are running in a loop
over the variables above.

### Task
All the router relates tasks are done using the [os_router][os_router_module] module.
The module creates a router, sets default gateway and attach the interfaces to the router.

The issue I faced, was that when attaching multiple interfaces to a single router, all of them
should be provided at once as a list. If providing the interfaces in a loop manner, just
the last interface took place in the router.

It means that I needed to take all the networks at once, extract the relevant data from it
and convert it to the list. Then that list could be given to the router at once.

First of all, I needed to decide how the state of the network should be set related to the router.
Every network could have 3 states.
>
* A network that is set as a default gateway for the router.
* A network that is set as an interface for the router.
* A network that should not be attached to the router in any case.

**Note** - The line numbers below are taken from the networks variables section provided above.

I decided that if the network will have `external`(***line 12***) and `router_name`(***line 13***)
attribute definition, the network will be attached to the router. If none of the above will be specified
in the network, it will not be attached to any router.

If the `external` parameter will be `external: true`(***line 12***), it will serve as the router
default gateway. If the `external` parameter will be `external: false`(***line 22***), it will serve
as the router interface. The `router_name`(***line 13***) parameter, will set the name of the router
that the network should be attached to.

As we can see from the networks above, the following scenario should happen:
* A router `router1` should be created. ***Lines 13, 23 and 34.***
* The first network `access` will be set as the default gateway for the `router1`. ***Line 12.***
* The second `dpdk-mgmt` and third `sriov-1` networks will be set as the interfaces of the `router1`. ***Lines 22 and 33.***
* A router `router2` should be created. ***Line 45.***
* The fourth `sriov-2` network will be set as the interface of the `router2`. ***Line 44.***

**Note** - From the scenario above, the `dpdk-mgmt` and `sriov-1` networks must be combined to a list and
provided to the router together.

### Final task definition
From the above description, the final data should look like the following:
1. A list of routers.
2. A list of lists of the interfaces.

\* The list of lists needs to be created to iterate using the [with together][ansible_with_together_loop] loop over the routers and
a list of interfaces that need to be attached to the router.

```
Example:
               ['router1'],                 ['router2']
                    |                            |
[['dpdk-mgmt_subnet', 'sriov-1_subnet'], ['sriov-2_subnet']]
```

### Solution
Below are the tasks that I used to build the required lists.
The tasks will be explained line by line.
All the data filtering is done on the "networks" variables provided at the top of the post.

{% highlight yaml linenos %}
{% raw %}
- name: Set routers names
  set_fact:
    routers_names: "{{ networks
      | selectattr('router_name', 'defined')
      | map(attribute='router_name')
      | list
      | unique }}"
{% endraw %}
{% endhighlight %}

<ins>Task "Set routers names":</ins>
* ***Line 3*** - Set fact of "routers_names" variable and loads the "networks" variable content as a bulk.
* ***Line 4*** - Filter networks with the attribute "router_name" defined.
* ***Line 5*** - From the above output, takes only the "router_name" attribute.
* ***Line 6*** - Convert the routers names to a list.
* ***Line 7*** - Removes the duplicates from the routers names list.

{% highlight yaml linenos %}
{% raw %}
- name: Set subnets for routers interfaces
  set_fact:
    subnets_for_routers_interfaces: "{{ subnets_for_routers_interfaces
      | default([]) }}
      + [ {{ networks
      | selectattr('router_name', 'defined')
      | selectattr('external', 'defined')
      | selectattr('external', 'equalto', False)
      | selectattr('router_name', 'match', item)
      | map(attribute='name')
      | map('regex_replace', '^(.*)$', '\\1_subnet')
      | list }} ]"
  loop: "{{ routers_names | flatten(levels=1) }}"
{% endraw %}
{% endhighlight %}

<ins>Task "Set subnets for routers interfaces":</ins>
* ***Line 3 - 4*** - Set fact of "subnets_for_routers_interfaces" variable and defines an empty list if
the variable does not exist.
* ***Line 5*** - The "+" sign appends all the data generated below to the list for every iteration.
Loads the "networks" variable content as a bulk.
* ***Line 6*** - Filter networks with the attribute "router_name" defined.
* ***Line 7*** - From the above output, filter the networks which have attribute "external" defined.
* ***Line 8*** - From the above output, filter the networks which attribute "external" equals to "false".
* ***Line 9*** - Perform another filter with the attribute of "router_name" which match to the loop item.
* ***Line 10*** - From the generated output takes only the name of the network.
* ***Line 11*** - Add the "_subnet" suffix to the name of the network to match the early created subnet.
* ***Line 12*** - Convert all the output to the list.
* ***Line 13*** - Loop over the "router_names" list created previously. (Used for the router name match).

{% highlight yaml linenos %}
{% raw %}
- name: Set routers list
  set_fact:
    routers_lists: "{{ routers_lists | default([]) + [[ item ]] }}"
  loop: "{{ routers_names | flatten(levels=1) }}"
{% endraw %}
{% endhighlight %}

<ins>Task "Set routers list":</ins>
* ***Line 3*** - Set fact of "routers_lists" variable and defined an empty list if the variable does not exist.
Add a loop item as a list of the list.
* ***Line 4*** - Loop over the "router_names" list created in the first task.

The final output of the generated variables will look like the following:
```yaml
TASK [routers_names] ***********
ok: [localhost] => {
    "msg": [
        "router1",
        "router2"
    ]
}

TASK [subnets_for_routers_interfaces] ***********
ok: [localhost] => {
    "msg": [
        [
            "dpdk-mgmt_subnet",
            "sriov-1_subnet"
        ],
        [
            "sriov-2_subnet"
        ]
    ]
}
```

The whole play of the routers and interfaces related tasks provided below:

{% highlight yaml linenos %}
{% raw %}
- name: Create a router and set router gateway
  vars:
    ansible_python_interpreter: "{{ venv_path }}/bin/python"
  os_router:
    cloud: "{{ overcloud_name }}"
    name: "{{ item.router_name }}"
    network: "{{ item.name }}"
    state: present
  when:
    - item.external is defined
    - item.external
    - item.router_name is defined
  loop: "{{ networks | flatten(levels=1) }}"

- name: Set routers names
  set_fact:
    routers_names: "{{ networks
      | selectattr('router_name', 'defined')
      | map(attribute='router_name')
      | list
      | unique }}"

- name: Set subnets for routers interfaces
  set_fact:
    subnets_for_routers_interfaces: "{{ subnets_for_routers_interfaces
      | default([]) }}
      + [ {{ networks
      | selectattr('router_name', 'defined')
      | selectattr('external', 'defined')
      | selectattr('external', 'equalto', False)
      | selectattr('router_name', 'match', item)
      | map(attribute='name')
      | map('regex_replace', '^(.*)$', '\\1_subnet')
      | list }} ]"
  loop: "{{ routers_names | flatten(levels=1) }}"

- name: Set routers list
  set_fact:
    routers_lists: "{{ routers_lists | default([]) + [[ item ]] }}"
  loop: "{{ routers_names | flatten(levels=1) }}"

- name: Create a router if required and set router interfaces
  vars:
    ansible_python_interpreter: "{{ venv_path }}/bin/python"
  os_router:
    cloud: "{{ overcloud_name }}"
    name: "{{ item.0.0 }}"
    interfaces: "{{ item.1 }}"
    state: present
  loop: "{{ routers_lists | zip(subnets_for_routers_interfaces) | list }}"
  when: routers_lists is defined
{% endraw %}
{% endhighlight %}

### Conclusion
The Jinaj2 filters are a very powerful feature and could help us in various tasks for the manipulation with the data.
You are welcome to find more filters in the [ansible documentation][jinja2_ansible_filters].


[//]: # Reference links
[jinja2_ansible_filters]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html
[os_router_module]: https://docs.ansible.com/ansible/latest/modules/os_router_module.html
[ansible_with_together_loop]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#with-together
