ansible-nxapi
=============
ABOUT

Ansible modules for Cisco Nexus 9000. Developed for Ansible 1.5. This library is not currently part of the Ansible distribution. Please refer to INSTALLATION section for setup.
OVERVIEW OF MODULES

Work in progress, please see library directory for modules. The modules are documented per-Ansible guidelines; look for the DOCSTRING within each file.
INSTALLATION

This repo assumes you have the DEPENDENCIES installed on your system. You can then git clone this repo and run the env-setup script in the repo directory:

user@ansible-nxapi> source env-setup

This will set your $ANSIBLE_LIBRARY variable to the repo location and the installed Ansible library path. For example:

[Ajeet-rock1@ansible-nxapi]$ echo $ANSIBLE_LIBRARY
/home/Ajeet-rock1/Ansible/ansible-nxapi/library:/usr/share/ansible

Alternatively you can copy the files in the library directory into the ansible installed directory: library/net_infrastructure


Prepare Your Cisco Nexus Switches

At this point, Ansible, the Cisco dependencies, and the custom Cisco Ansible modules should be installed on the Ansible control host if you've been following along. The last step is to ensure the Nexus switches are configured correctly to work with Ansible. This basically means to ensure two things: (1) make sure NX-API is enabled and (2) make sure the Ansible control host can ping the mgmt0 interface of the switch(es).

The feature command is used to enable NX-API. After it's enabled, ensure the device is listening on port 80. The modules in this repo only operate using http/80. Https/443 to come in the future.

n9k1# config t
Enter configuration commands, one per line.  End with CNTL/Z.
n9k1(config)# feature nxapi
n9k1(config)# exit

n9k1# show nxapi
nxapi enabled
Listen on port 80
Listen on port 443

For the mgmt0 interface and outbound connectivity, there are a few things that need to happen before using the modules.

   1:- Configure an IP address on mgmt0
   2:- Configure a default route for the management VRF (or at a minimum a route to the Ansible control host)
   3:- Test connectivity between the Ansible control host and mgmt0 (remember if you are using Virtualbox or Fusion with a NAT configuration, you won't be able to ping from your switch to your VM).
   
   
   Example Playbook

A sample playbook is shown below. Assume that this playbook is saved as nexus-automation.yml and is stored in your working directory - nxos-ansible --- the directory that was downloaded when you cloned the repo. This will continuously be referred to as the current Ansible working directory. Since this walk through is using the Cisco all-in-one onePK virtual machine, the full path to this file is: /home/cisco/nxos-ansible/nexus-automation.yml. Feel free to create it and follow along.

---

- name: Dummy playbook
  hosts: n9k1

  tasks:

    - name: default interfaces
      nxos_interface: interface={{ item }} state=default host={{ inventory_hostname }}
      with_items:
        - Ethernet1/1
        - Ethernet2/1

    # Ensure an interface is a Layer 3 port and that it has the proper description
    - name: config interface
      nxos_interface: interface=Ethernet1/1 description='Configured by Ansible' mode=layer3 host={{ inventory_hostname }}

    # Admin down an interface
    - nxos_interface: interface=Ethernet2/1 admin_state=down host={{ inventory_hostname }}
    
    
    As you'll see above, the playbook is a set of automation instructions defined in YAML. The --- denotes the start of a YAML file, and in this case, also an Ansible playbook. Just below, there is a grouping of two key-value pairs that will be used for the play that follows. name is arbritary and is text that is displayed when the playbook is run. hosts denotes the host or group of hosts that will have the automation instructions, or tasks, executed against.

If you named your switch something other than n9k1 in the /etc/hosts/ file, use your switch name! The inventory (or hosts) file is also an important component of Ansible that will be covered in the next section.

Just below those two k-v pairs, there is a list of three tasks that will be automated. They all call the nxos_interface module. Following the module name are a number of key-value pairs in the form of key=value. These k-v pairs are sent to the module for processing against the device.

    Note: Nearly all Cisco modules are idempotent, which means, if the device is already in the desired state, no change will be made. A change will only be made if it's required to get the device into the desired state.

As you can probably tell, the first task will default Eth1/1 and Eth2/1, the second tasks will ensure Ethernet1/1 has the description defined in the k-v pair and also ensure that Ethernet1/1 is a layer 3 port. The third task ensures that Ethernet2/1 is in the admin down state. As stated previously, these modules are idempotent, so if Ethernet2/1 is already in the admin down state, no commands will be sent to the device (as an example).
Hosts File

For each play, the administrator needs to define the host(s) that the set of tasks will be executed against. The example above executes two tasks against n9k1. This can be seen in the top left of the playbook where hosts: n9k1 is defined. This can be a group of hosts or a single host and maps directly back to an inventory, or hosts, file.


The inventory hosts file is an ini based file. The hosts file for the above example looks like this:

[all:vars]
ansible_connection = local

[spine]
n9k1
n9k2

[leaf]
n3k1
n9k3
n9k4

[wan]
delhi  mgmt_ip=10.1.10.1
mumbai  mgmt_ip=10.1.20.1
lucknow
pune
banglore




    Note: You will need the [all:vars] section for every inventory file you create since we are using a custom connection protocol. The other option is to use connection: local for each play. If you are integrating with server environments, you may need a combination of both.

Since you cloned the repo, just make sure you are in the nxos-ansible and issue the command:

cisco@onepk:~/nxos-ansible$ cat hosts

You should see what is shown in the text above. If you are only testing with a single switch, you can remove all other 9k# hosts from the inventory file. If you don't, you'll get errors below as you run through the examples.

    Note: You can disregard the wan section. That'll be used in an upcoming example.

    Note: the names in the hosts file should match what you have in DNS, the /etc/hosts/ file, or it can just be the mgmt IP Address of the switch.

The same play could have used hosts: spine instead of hosts: n9k1 in order to automate n9k1, n9k2 for all tasks in a given play or hosts: all to automate all hosts.



    Note: you may have noticed {{ inventory_hostname }} in the playbook. Double curly braces are used to reference variables and it's worth noting that inventory_hostname is an internal / built-in Ansible variable. This variable is equivalent to the hostname as defined in the hosts file as the hosts get iterated through to execute the set of tasks in the playbook. In this previous example, this means that n9k1 would get sent to the Ansible module during the first task run, n9k2 for the second run, and so on, etc.

    Note: names are not required for the modules. IP Addresses work just fine too if DNS or the hosts file isn't defined.

Executing Ansible Playbooks

By now, you should have a very high level understanding of the layout of an Ansible playbook. It's worth taking a look at how Ansible playbooks are executed. As stated above, the filename of the playbook is nexus-automation.yml. The name of the inventory hosts file is called hosts. For this example, ensure both are stored in the same working directory (/home/cisco/nxos-ansible/). As with most files being reviewed in this document, they will have already been created and downloaded when you cloned the repository, but they are still being covered for completeness.

To execute the playbook, the following terminal command is used:

cisco@onepk:~/nxos-ansible$ ansible-playbook -i hosts nexus-automation.yml

-i hosts tells the system which inventory hosts file to use.

After running the playbook, feel free to check the changes out on your switch.

In order to not have to continuously state where the hosts file is, you can set the ANSIBLE_HOSTS environment variable. Within your working directory, you can source the env-setup file.

cisco@onepk:~/nxos-ansible$ source env-setup                  

That covers some of the basics to get started with Ansible. The next section walks through how to use an Ansible core module called template that can help in creating network device templates and simplify how configuration files are created for network devices of all types. Following the templating section is when we'll cover using the Cisco specific Ansible modules.
Network Configuration Templating

    Creating the Configs Directory
    Creating a Template
    Intro to Ansible Variables
    Creating a New Playbook
    Dynamically Creating Config Files

The previous sections gave a high level overview of Ansible and reviewed an example playbook. In this section, we'll look at how to use Ansible to template build configurations for network devices by walking through another playbook and learning a bit more about Ansible.

First, we'll ensure the inventory hosts file looks like the following. You should have the group wan and each hostname and variable underneath as shown here:

[spine]
n9k1
n9k2

[leaf]
n9k3
n9k4

[wan]
delhi  mgmt_ip=10.1.10.1
mumbai  mgmt_ip=10.1.20.1
lucknow
pune
banglore




    This example will run through building IOS router configurations. This is to show that templating with Ansible has no direct correlation to the Nexus or NX-OS platform. Note: the only requirement for Nexus is when you are using the custom Cisco Ansible modules that'll be covered in the next section.

In this section, we'll be using the Ansible core module called template to automate the creation of the configuration files required for routers that exist at the 5 locations listed in the hosts file. First, we'll do two things before creating the configurations. (1) create a new directory that will store the final configs and (2) Create a very basic config template. Again, this will have already been done for you if you cloned the repository.
Creating the Configs Directory

This is straightforward. A new directory will be created that will store the final config files rendered for each device in the hosts file. We'll create a new directory called configs (mkdir configs) that exists in the working directory (/home/cisco/nxos-ansible/configs).

Creating a Template

Ansible integrates natively with Jinja2 templates, so that's what will be used here. Create a new file called routers.j2 in the working directory.

    Note: since this is already in your repository, feel free to open it up to view it. You can issue a cat templates/routers.j2 to check it out.

The contents should look like the following:

hostname {{ inventory_hostname }}

interface mgmt0
  ip address {{ mgmt_ip }}

ntp server {{ ntp_server }}
snmp-server {{ snmp_server }}

username cisco password cisco

    Note: this is an extremely basic config template. It is possible to get much more robust with how templates are created using Ansible with different variables per site, region, etc.

For more details on Jinja2, please reference the official Jinja2 docs.

As you may recall, {{ inventory_hostname }} is an internal Ansible variable, so the hostname of each device, will be equivalent to what is defined in the hosts file.

Intro to Ansible Variables

The other variables found in the Jinja template can be defined in a number of locations. For this example, we'll show defining them in three locations.

    Note: for more detail about variables and variables scope, please reference the Ansible variables docs.

First, variables can be defined in the hosts file as you may have already noticed.

[spine]
n9k1
n9k2

[leaf]
n9k3
n9k4

[wan]
delhi  mgmt_ip=10.1.10.1
mumbai  mgmt_ip=10.1.20.1
lucknow
pune
banglore

Second is in a file called wan.yml that needs to be created and stored in a group_vars directory. This group will match the group as defined in the hosts file.

The path to this file should be the ansible working directory group_vars/wan.yml, so for this complete example it would be /home/cisco/nxos-ansible/group_vars/wan.yml Any variables found in wan.yml can be accessed by any device in the WAN group within the hosts file.


wan.yml looks like the following:

---
ntp_server: 192.168.100.11
snmp_server: 192.168.100.12

The third location we'll store variables is in host specific variables files. Also within the working directory, there will be a new dir called host_vars. The dir should contain as many files that are required that contain variables that can only be used for and by specific devices. In this example, there will be three new yaml files that exist within the host_vars directory. They are sjc.yml, rtp.yml, and richardson.yml. These file names must match the name of the device as defined in the inventory hosts file.

    Variables can also be stored in the all.yml file that would exist in the group_vars directory.

These three files look like the following:

File: /home/cisco/nxos-ansible/host_vars/sjc.yml

---
mgmt_ip: 10.1.30.1

File: /home/cisco/nxos-ansible/host_vars/rtp.yml

---
mgmt_ip: 10.1.40.1

File: /home/cisco/nxos-ansible/host_vars/richardson.yml

---
mgmt_ip: 10.1.50.1

Creating a New Playbook

In the working directory, create a new playbook called config-builder.yml. It should have the following contents:

---

- name: template building
  hosts: wan
  tasks:

    # Create config files
    - template: src=templates/routers.j2 dest=configs/{{ inventory_hostname }}.cfg

This playbook will leverage the newly created hosts file, all.yml, three different host variables files (.yml), and the Jinja2 template.

