OpenStack
=========

This project provides a number of playbooks to get your hosts created
using OpenStack. You can find them in ``openstack/`` in the main
project directory.

Before we can build any servers using Ansible, we need to configure authentication.

Configuring OpenStack authentication 
------------------------------------

Because we can deploy to different OpenStack regions, we'll have to use a file that contains unique configurations for each region.

Example files are located in ``inventory/group_vars/dc1`` and ``inventory/group_vars/dc2``

The file looks like this:

.. code-block:: yaml

  os_auth_url:
  os_tenant_name:
  os_tenant_id:
  os_net_id: fd94baf0-b314-453e-9cd6-73c90fc53857
  consul_dc: dc1
  security_group: default

In the next sections, we'll explain how to obtain these settings.

Getting OpenStack tenant settings
----------------------------------------
``os_auth_url``, ``os_tenant_name``, and ``os_tenant_id`` is unique for each OpenStack datacenter. You can get these from the OpenStack web console:

1. Log Into the OpenStack web console and in the Manage Compute section, select "Access & Security". 

2. Select the "API Access" tab.

3. Click on the "Download the OpenStack RC File" button. We'll use this file to set up authentication.
4. Download the RC file for each Data Center you want to provision servers in. You may have to log into different OpenStack web consoles.

.. image:: _static/openstack_rc.png

Open the file that you just downloaded. We are interested in three of the environment variables that are exported:

.. code-block:: shell

  export OS_AUTH_URL=https://my.openstack.com:5000/v2.0
  export OS_TENANT_ID=my-long-unique-id
  export OS_TENANT_NAME="my-project"


Update the ``inventory/group_vars/dc1`` file with these values for the apropriate fields. If you have a second region, update ``inventory/group_vars/dc2`` with values from the second file.


OpenStack Username/Password
---------------------------

The playbooks get Username/Password information via environment variables:

.. envvar:: OS_USERNAME

   Your OpenStack username

.. envvar:: OS_PASSWORD

   Your OpenStack password


Before running any playbooks, run the following command to to pull in your username and password for Ansible to use, changing the file name and location to the location of your OpenStack RC file:

.. code-block:: shell

  source ~/Downloads/my-project.rc

.. note:: The default OpenStack RC file will prompt for your password in order to set OS_PASSWORD. 

Creating a Network
------------------

You will need to create a network in your OpenStack tenant for the
services in this project to run on. The commands below will create
your network and router, and allow traffic through on the ports you'll
need to use to communicate with the services.

Network
^^^^^^^

The network is the first thing you'll need to create in OpenStack. The
ID of this network (in our example
``fd94baf0-b314-453e-9cd6-73c90fc53857``) is what you'll need to add
as ``os_net_id`` in the ``group_vars`` files mentioned above.

.. note:: If you already have a network and routers in your OpenStack region, you can skip these steps and just use the ID of your existing network. Make sure that hosts on the existing network can resolve DNS names and pull data from IP addresses (like ``centos.org``).


.. code-block:: shell

   $ neutron net-create network1
   Created a new network:
   +-----------------+--------------------------------------+
   | Field           | Value                                |
   +-----------------+--------------------------------------+
   | admin_state_up  | True                                 |
   | id              | fd94baf0-b314-453e-9cd6-73c90fc53857 |
   | name            | network1                             |
   | router:external | False                                |
   | shared          | False                                |
   | status          | ACTIVE                               |
   | subnets         |                                      |
   | tenant_id       | ...                                  |
   +-----------------+--------------------------------------+

   $ neutron subnet-create network1 10.10.10.0/24 --name subnet1
   Created a new subnet:
   +-------------------+------------------------------------------------+
   | Field             | Value                                          |
   +-------------------+------------------------------------------------+
   | allocation_pools  | {"start": "10.10.10.2", "end": "10.10.10.254"} |
   | cidr              | 10.10.10.0/24                                  |
   | dns_nameservers   |                                                |
   | enable_dhcp       | True                                           |
   | gateway_ip        | 10.10.10.1                                     |
   | host_routes       |                                                |
   | id                | ...                                            |
   | ip_version        | 4                                              |
   | ipv6_address_mode |                                                |
   | ipv6_ra_mode      |                                                |
   | name              | subnet1                                        |
   | network_id        | fd94baf0-b314-453e-9cd6-73c90fc53857           |
   | tenant_id         | ...                                            |
   +-------------------+------------------------------------------------+

Router
^^^^^^

Once you've created your network, you'll also need a router with an
external gateway on an existing publicly accessible network. In the example below,
put the name of the publicly accessible network in place of <external network>. If
you followed the tutorial above, this can be ``network1``

.. code-block:: shell

   $ neutron router-create router1
   Created a new router:
   +-----------------------+--------------------------------------+
   | Field                 | Value                                |
   +-----------------------+--------------------------------------+
   | admin_state_up        | True                                 |
   | external_gateway_info |                                      |
   | id                    | c5a07e4d-09d2-434a-96b2-73c088c13dc5 |
   | name                  | router1                              |
   | routes                |                                      |
   | status                | ACTIVE                               |
   | tenant_id             | 7dc1ba3b443c4b34a202924a75bd81a3     |
   +-----------------------+--------------------------------------+

   $ neutron router-gateway-set router1 <external network>
   Set gateway for router router1

   $ neutron router-interface-add router1 subnet1
   Added interface ... to router router1.

To check that everything was created successfully, run ``neutron
router-show router1``,``network_id`` should be set.

Security Group
^^^^^^^^^^^^^^

You should add the following rules to your security group. These are
for the web and publicly facing interfaces to the various services in
your cluster:

.. table:: Security Group Rules

   ================ ======== =========
   Service          Protocol Ports    
   ================ ======== =========
   Ping             ICMP     -1       
   SSH              TCP      22
   HAproxy          TCP      80
   Mesos            TCP      5050
   Marathon         TCP      8080
   Consul           TCP      8500
   ================ ======== =========


We suggest creating a new security group instead of using the default group.

In the following example, we're going to create a security group called ``microservice`` that allows a few connections from the outside, but trusts all traffic between hosts in the security group. 

You'll have to create a security group for each OpenStack region you are going
to create nodes in:

.. code-block:: shell

   nova secgroup-create microservice "microservices-infrastructure"
   nova secgroup-add-rule microservice icmp -1 -1 0.0.0.0/0
   nova secgroup-add-rule microservice tcp 22 22 0.0.0.0/0
   nova secgroup-add-rule microservice tcp 80 80 0.0.0.0/0
   nova secgroup-add-rule microservice tcp 5050 5050 0.0.0.0/0
   nova secgroup-add-rule microservice tcp 8080 8080 0.0.0.0/0
   nova secgroup-add-rule microservice tcp 8500 8500 0.0.0.0/0
   nova secgroup-add-group-rule microservice microservice tcp 1 65535

Once you have created this group in each dc, update the ``security_group:``
entry in the ``inventory/group_vars/dc1`` and ``inventory/group_vars/dc2``
files with the name of this security group.

In order for the dcs to be able to talk to each other, you'll have to 
open us the security for each of their subnets and make sure they can 
connect to each other. 

In ``dc1`` you'll want to run:

.. code-block:: shell
  
   nova secgroup-add-rule microservice tcp 1 65535 <cidr for dc2 network>

Where the cidr will be formatted like ``172.18.19.0/23``.

And in ``dc2`` you'll want to run:

.. code-block:: shell

   nova secgroup-add-rule microservice tcp 1 65535 <cidr for dc1 network>

If you don't set up connectivity between DCs, Consul WAN dns lookups won't
work. Strictly speaking, only the consul WAN ports need to be open between data centers, but we are opening everything here for simplicity. 

Creating Instances
------------------

After setting up auth and your network, you can add your SSH key to
your tenant with ``openstack/provision-nova-key.yml``, spin up new
instances with ``openstack/provision-hosts.yml``, and destroy them
with ``openstack/destroy-hosts.yml``. These playbooks all use the host
variables defined in ``inventory/``

A SSH key is required to configure servers. ``openstack/provision-nova-key.yml`` will take the your ``${HOME}/.ssh/id_rsa`` and upload it to OpenStack as ``ansible_key``. SSH key vars can be changed via the ``inventory/group_vars/all/all.yml`` file.

.. code-block:: shell 

  ansible-playbook -i inventory/1-datacenter openstack/provision-nova-key.yml

Here's an example invocation:

.. code-block:: shell

  ansible-playbook -i inventory/1-datacenter openstack/provision-hosts.yml

If you already have a CentOS 7 image in your OpenStack environment, you don't need to create a new one. 


