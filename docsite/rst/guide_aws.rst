Amazon Web Services Guide
=========================

.. contents::
   :depth: 2

.. _aws_intro:

Introduction
````````````

.. note:: This section of the documentation is under construction.  We are in the process of adding more examples about all of the EC2 modules
   and how they work together.  There's also an ec2 example in the language_features directory of `the ansible-examples github repository <http://github.com/ansible/ansible-examples/>`_ that you may wish to consult.  Once complete, there will also be new examples of ec2 in ansible-examples.

Ansible contains a number of core modules for interacting with Amazon Web Services (AWS).  These also work with Eucalyptus, which is an AWS compatible private cloud solution.  There are other supported cloud types, but this documentation chapter is about AWS API clouds.  The purpose of this
section is to explain how to put Ansible modules together (and use inventory scripts) to use Ansible in AWS context.

Requirements for the AWS modules are minimal.  All of the modules require and are tested against boto 2.5 or higher. You'll need this Python module installed on the execution host. If you are using Red Hat Enterprise Linux or CentOS, install boto from `EPEL <http://fedoraproject.org/wiki/EPEL>`_:

.. code-block:: bash

    $ yum install python-boto

You can also install it via pip if you want.

The following steps will often execute outside the host loop, so it makes sense to add localhost to inventory.  Ansible
may not require this step in the future::

    [local]
    localhost

And in your playbook steps we'll typically be using the following pattern for provisioning steps::

    - hosts: localhost
      connection: local
      gather_facts: False

.. _aws_provisioning:

Provisioning
````````````

The ec2 module provides the ability to provision instances within EC2.  Typically the provisioning task will be performed against your Ansible master server as a local_action statement.  

.. note::

   Authentication with the AWS-related modules is handled by either 
   specifying your access and secret key as ENV variables or passing
   them as module arguments. 

.. note::

   To talk to specific endpoints, the environmental variable EC2_URL
   can be set.  This is useful if using a private cloud like Eucalyptus, 
   exporting the variable as EC2_URL=https://myhost:8773/services/Eucalyptus.
   This can be set using the 'environment' keyword in Ansible if you like.

The ec2 module provides the ability to provision EC2 instance(s).  Typically the provisioning task will be performed against your Ansible master server as a local_action. There are two ways to provision instances:

* **Non-idempotent provisioning** (default). With non-idempotent provisioning, each time the provision command is run, a new instance is provisioned regardless of any instances that are alredy running.
* **Idempotent provisioning** (enabled with the idempotent_attribute option). By using the idempotent_attribute, the provisioning will ensure that at least count=N instances are running.  If N instance are already running, no new instances will be provisioned.

Non-idempotent provisioning
++++++++++++++

Here is an example of provisioning a number of instances in ad-hoc mode mode:

.. code-block:: bash

    # ansible localhost -m ec2 -a "image=ami-6e649707 instance_type=m1.large keypair=mykey group=webservers wait=yes" -c local

In a play, this might look like (assuming the parameters are held as vars)::

    tasks:
    - name: Provision a set of instances
      local_action: ec2 
          keypair={{mykeypair}} 
          group={{security_group}} 
          instance_type={{instance_type}} 
          image={{image}} 
          wait=true 
          count={{number}}
      register: ec2
        
By registering the return its then possible to dynamically create a host group consisting of these new instances.  This facilitates performing configuration actions on the hosts immediately in a subsequent play::

        
By registering the return its then possible to dynamically create a host group consisting of these new instances.  This facilitates performing configuration actions on the hosts immediately in a subsequent task::

    - name: Add all instance public IPs to host group
      local_action: add_host hostname={{ item.public_ip }} groupname=ec2hosts
      with_items: ec2.instances

With the host group now created, a second play in your provision playbook might now have some configuration steps::

    - name: Configuration play
      hosts: ec2hosts
      user: ec2-user
      gather_facts: true

      tasks:
      - name: Check NTP service
        action: service name=ntpd state=started

Rather than include configuration inline, you may also choose to just do it as a task include or a role.

The method above ties the configuration of a host with the provisioning step.  This isn't always ideal and leads us onto the next section.

Idempotent provisioning
++++++++++++++
Idempotent provisioning provides a simple mechanism for maintaing a specified number of instances running in a particular host group.

Using the ec2 inventory plugin ([documented in the API chapter](http://ansible.cc/docs/api.html#external-inventory-scripts>)) it is possible to group hosts by security group, machine image (AMI) or instance tags. Instance tags in particular provide a flexible way of marking instances as belonging to a particular host group.

The following example shows how one can idempotently provision a group of 5 hosts tagged as webservers::

    - local_action: 
	module: ec2 
	keypair: mykey 
	group: webservers
	instance_type: m1.large 
	image: ami-6e649707 
	wait: yes 
	count: 5 
	instance_tags: '{"name":"webserver"}'
	idempotency_attribute: instance_tags

If this play is run when 3 EC2 instances with the tag `'{"name":"webserver"}'` are already running, then only two more will be provisioned in order to bring the total up to 5. If five such instances are already running, then no new instances will be provisioned. If you wanted to refer to this group at some point in the future, then make sure the EC2 inventory plugin is enabled and select the hosts using::

    - hosts: tag_Name_webservers
      gather_facts: false
      sudo: true

      tasks:
      ...

Note that the value of the `idempotency_attribute` option can also be `image`, `group` (security group), `group_id` (security group id) or `client-token`. The `client-token` is set using the `id` option.

:: _aws_advanced:
Advanced Usage
``````````````

:: _aws_host_inventory:

Host Inventory
++++++++++++++

Once your nodes are spun up, you'll probably want to talk to them again.  The best way to handle his is to use the ec2 inventory plugin.

Even for larger environments, you might have nodes spun up from Cloud Formations or other tooling.  You don't have to use Ansible to spin up guests.  Once these are created and you wish to configure them, the EC2 API can be used to return system grouping with the help of the EC2 inventory script. This script can be used to group resources by their security group or tags. Tagging is highly recommended in EC2 and can provide an easy way to sort between host groups and roles. The inventory script is documented `in the API chapter <http://www.ansibleworks.com/docs/api.html#external-inventory-scripts>`_.

You may wish to schedule a regular refresh of the inventory cache to accommodate for frequent changes in resources:

.. code-block:: bash
   
    # ./ec2.py --refresh-cache

Put this into a crontab as appropriate to make calls from your Ansible master server to the EC2 API endpoints and gather host information.  The aim is to keep the view of hosts as up-to-date as possible, so schedule accordingly. Playbook calls could then also be scheduled to act on the refreshed hosts inventory after each refresh.  This approach means that machine images can remain "raw", containing no payload and OS-only.  Configuration of the workload is handled entirely by Ansible.  

:: _aws_pull:

Pull Configuration
++++++++++++++++++

For some the delay between refreshing host information and acting on that host information (i.e. running Ansible tasks against the hosts) may be too long. This may be the case in such scenarios where EC2 AutoScaling is being used to scale the number of instances as a result of a particular event. Such an event may require that hosts come online and are configured as soon as possible (even a 1 minute delay may be undesirable).  Its possible to pre-bake machine images which contain the necessary ansible-pull script and components to pull and run a playbook via git. The machine images could be configured to run ansible-pull upon boot as part of the bootstrapping procedure. 

More information on pull-mode playbooks can be found `here <http://www.ansibleworks.com/docs/playbooks2.html#pull-mode-playbooks>`_.

(Various developments around Ansible are also going to make this easier in the near future.  Stay tuned!)

:: _aws_autoscale:

AWX Autoscaling
+++++++++++++++

AnsibleWorks's "AWX" product also contains a very nice feature for auto-scaling use cases.  In this mode, a simple curl script can call
a defined URL and the server will "dial out" to the requester and configure an instance that is spinning up.  This can be a great way
to reconfigure ephmeral nodes.  See the AWX documentation for more details.  Click on the AWX link in the sidebar for details.

A benefit of using the callback in AWX over pull mode is that job results are still centrally recorded and less information has to be shared
with remote hosts.

:: _aws_use_cases:

Use Cases
`````````

This section covers some usage examples built around a specific use case.

:: _aws_cloudformation_example:

Example 1
+++++++++

    Example 1: I'm using CloudFormation to deploy a specific infrastructure stack.  I'd like to manage configuration of the instances with Ansible.

Provision instances with your tool of choice and consider using the inventory plugin to group hosts based on particular tags or security group. Consider tagging instances you wish to managed with Ansible with a suitably unique key=value tag.

.. note:: Ansible also has a cloudformation module you may wish to explore.

:: _aws_autoscale_example:

Example 2
+++++++++

    Example 2: I'm using AutoScaling to dynamically scale up and scale down the number of instances. This means the number of hosts is constantly fluctuating but I'm letting EC2 automatically handle the provisioning of these instances.  I don't want to fully bake a machine image, I'd like to use Ansible to configure the hosts.

There are several approaches to this use case.  The first is to use the inventory plugin to regularly refresh host information and then target hosts based on the latest inventory data.  The second is to use ansible-pull triggered by a user-data script (specified in the launch configuration) which would then mean that each instance would fetch Ansible and the latest playbook from a git repository and run locally to configure itself. You could also use the AWX callback feature.

:: _aws_builds:

Example 3
+++++++++

    Example 3: I don't want to use Ansible to manage my instances but I'd like to consider using Ansible to build my fully-baked machine images.

There's nothing to stop you doing this. If you like working with Ansible's playbook format then writing a playbook to create an image; create an image file with dd, give it a filesystem and then install packages and finally chroot into it for further configuration.  Ansible has the 'chroot' plugin for this purpose, just add the following to your inventory file::

    /chroot/path ansible_connection=chroot

And in your playbook::

    hosts: /chroot/path

.. note:: more examples of this are pending.   You may also be interested in the ec2_ami module for taking AMIs of running instances.

:: _aws_pending:

Pending Information
```````````````````

In the future look here for more topics.


.. seealso::

   :doc:`modules`
       All the documentation for Ansible modules
   :doc:`playbooks`
       An introduction to playbooks
   :doc:`playbooks_delegation`
       Delegation, useful for working with loud balancers, clouds, and locally executed steps.
   `User Mailing List <http://groups.google.com/group/ansible-devel>`_
       Have a question?  Stop by the google group!
   `irc.freenode.net <http://irc.freenode.net>`_
       #ansible IRC chat channel

