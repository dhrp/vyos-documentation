:lastproofread: 2021-07-12

.. _cloud-init:

###############
VyOS cloud-init
###############

Cloud and virtualized instances of VyOS are initialized using the
industry-standard cloud-init. Via cloud-init, the system performs tasks such as
injecting SSH keys and configuring the network. In addition, the user can supply
a custom configuration at the time of instance launch.

**************
Config Sources
**************

VyOS support three types of config sources.

* Metadata - Metadata is sourced by the cloud platform or hypervisor.
  In some clouds, there is implemented as an HTTP endpoint at
  ```http://169.254.169.254```.
* Network configuration - This config source informs the system about the
  network settings like IP addresses, routes, DNS. Available only in several
  cloud and virtualization platforms.
* User-data - User-data is specified by the user. This config source offers the
  ability to insert any CLI configuration commands into the configuration before
  the first boot.

*********
User-data
*********

Major cloud providers offer a means of providing user-data at the time of
instance launch. It can be provided as plain text or as base64-encoded text,
depending on cloud provider. Also, it can be compressed using gzip, which makes
sense with a long configuration commands list, because of the hard limit to
~16384 bytes for the whole user-data.

The easiest way to configure the system via user-data is the Cloud-config syntax
described below.

********************
Cloud-config modules
********************

In VyOS, by default, enabled only two modules:

* ``write_files`` - this module allows to insert any files into the filesystem
  before the first boot, for example, pre-generated encryption keys,
  certificates, or even a whole ``config.boot`` file.
* ``vyos_userdata`` - the module accepts a list of CLI configuration commands in
  a ``vyos_config_commands`` section, which gives an easy way to configure the
  system during deployment.

************************
cloud-config file format
************************

A cloud-config document is written in YAML. The file must begin
with ``#cloud-config`` line. The only supported top-level keys are
``vyos_config_commands`` and ``write_files``. The use of these keys is described
in the following two sections.


************************
Initial Configuration
************************


The key used to designate a VyOS configuration is ``vyos_config_commands``. What 
follows is VyOS configuration using the "set-style" syntax. Both "set" and "delete" 
commands are supported.

Commands requirements:

* one command per line
* if command ends in a value, it must be inside single quotes
* a single-quote symbol is not allowed inside command or value

The commands list produced by the ``show configuration commands`` command on a
VyOS router should comply with all the requirements, so it is easy to get a 
proper commands list by copying it from another router.

The configuration specified in the cloud-config document overwrites default
configuration values and values configured via Metadata.

Here is an example cloud-config that appends configuration at the time of first boot.

.. code-block:: yaml

   #cloud-config
   vyos_config_commands:
     - set system host-name 'vyos-prod-ashburn'
     - set system ntp server 1.pool.ntp.org
     - set system ntp server 2.pool.ntp.org
     - delete interfaces ethernet eth1 address 'dhcp'
     - set interfaces ethernet eth1 address '192.0.2.247/24'
     - set protocols static route 198.51.100.0/24 next-hop '192.0.2.1'

-------------------------
System Defaults/Fallbacks
-------------------------

These are the VyOS defaults and fallbacks.

* SSH is configured on port 22
* ``vyos``/``vyos`` credentials if no others specified by data source
* DHCP on first Ethernet interface if no network configuration is provided

All of these can be overridden using the configuration in user-data.


*********************************
Command Execution at Initial Boot
*********************************

VyOS supports the execution of operational commands and linux commands at
initial boot. This is accomplished using ``write_files`` to certain
files in the /opt/vyatta/etc/config/scripts directory. Commands specified
in opt/vyatta/etc/config/scripts/vyos-preconfig-bootup.script are executed
prior to configuration. The 
/opt/vyatta/etc/config/scripts/vyos-postconfig-bootup.script file contains
commands to be executed after configuration. In both cases, commands are
executed as the root user.

Note that the /opt/vyatta/etc/config is used instead of the /config/scripts
directory referenced in the :ref:`command-scripting` section of the 
documentation because the /config/script directory isn't mounted when the 
``write_files`` module executes.

The following example shows how to execute commands after the initial 
configuration.

.. code-block:: yaml

   #cloud-config
   write_files:
     - path: /opt/vyatta/etc/config/scripts/vyos-postconfig-bootup.script
       owner: root:vyattacfg
       permissions: '0775'
       content: |
         #!/bin/vbash
         source /opt/vyatta/etc/functions/script-template
         filename=/tmp/bgp_status_`date +"%Y_%m_%d_%I_%M_%p"`.log
         run show ip bgp summary >> $filename


If you need to gather information from linux commands to configure VyOS, you can
execute commands and then configure VyOS in the same script.

The following example sets the hostname based on the instance identifier
obtained from the EC2 metadata service.

.. code-block:: yaml


   #cloud-config
   write_files:
     - path: /opt/vyatta/etc/config/scripts/vyos-postconfig-bootup.script
       owner: root:vyattacfg
       permissions: '0775'
       content: |
         #!/bin/vbash
         source /opt/vyatta/etc/functions/script-template
         hostname=`curl -s http://169.254.169.254/latest/meta-data/instance-id`
         configure
         set system host-name $hostname
         commit
         exit

***************
Troubleshooting
***************

If you encounter problems, verify that the cloud-config document contains
valid YAML. Online resources such as https://yamlvalidator.com/ provide
a simple tool for validating YAML.

cloud-init logs to /var/log/cloud-init.log. This file can be helpful in
determining why the configuration varies from what you expect. You can fetch the
most important data filtering output for ``vyos`` keyword:

.. code-block:: none

    sudo grep vyos /var/log/cloud-init.log

