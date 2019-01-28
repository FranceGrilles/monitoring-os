====================
Monitoring OpenStack
====================

This repository provides a set of Nagios probes for monitoring an `OpenStack Cloud <https://www.openstack.org>`_. These probes can be categorized in two parts. The first permits to check the API and is composed of the following probes:

* check_os_cinder
* check_os_glance
* check_os_keystone
* check_os_neutron
* check_os_nova

The other part is for checking an OpenStack scenario. This probe launch a VM, create and attach storage to the VM, remove and delete the storage, attach and remove a floating IP and delete the VM. It is composed of the following probe:

* check_os_scenario

The development of these probes is supported by `France Grilles <https://www.france-grilles.fr>`_.


Requirement
===========

The OpenStack API probes require:

* Python 2.7
* python-six

The OpenStack scenario probe requires:

* Python 2.7
* python-keystoneauth1
* python-novaclient
* python-cinderclient

The probes have been successfully tested up to the OpenStack Rocky version.


Installation
============

The installation is quite simple. The probes have to be placed into the
Nagios probe directory and made usable by the Nagios user:

.. code-block:: bash

   # cp plugins/check_os_* /usr/lib64/nagios/plugins/
   # chown -R nagios /usr/lib64/nagios/plugins/check_os_*


Configuration
=============

OS probes configuration
-----------------------

For each monitored OpenStack Cloud, a specific configuration file (ini format) needs to be created. It may be placed in the Nagios home directory. This configuration file should only be readable by the Nagios user:

.. code-block:: bash

   # cat /var/spool/nagios/.creds/my_site.conf
   [keystone_authtoken]
   username = demo
   password = secure_password
   project_name = demo
   user_domain_name = default
   project_domain_name = default
   auth_uri = https://keystone.example.org:5000/v3
   cacert = /etc/pki/tls/certs/CA.pem
   insecure = False

   [openstack_scenario]
   flavor_id = 2
   network_id = 7eb64fb2-0e0e-11e7-8f72-c358c3b0ad79
   image_id = 860feb56-0e0e-11e7-a410-43ffadc83bc1
   external_network_id = ec386ec8-1b21-11e8-8685-7721a9674c4a

   # ls -l /var/spool/nagios/.creds/my_site.conf
   -rw------- 1 nagios nagios 224 Mar  1 08:51 /var/spool/nagios/.creds/my_site.conf

The **network_id** parameter defines the id of the network available to the *demo* project and the **image_id** parameter the id of the image to use when creating the VM.

The **external_network_id** parameter is not mandatory and should set if the floating ip allocation should be tested. In this case, the parameter should contain the id of the network providing the floating ip pool.

Please note that:

* The cacert parameter is not mandatory.
* glance, cinder, neutron and nova API parameters are automatically retrieved from Keystone endpoints catalog
* If you want to by-pass the SSL verification, set the **insecure** option to *True*.

Before configuring Nagios to use these probes, it is wise to check that they are working properly:

.. code-block:: bash

   # sudo -u nagios /usr/lib64/nagios/plugins/check_os_keystone /var/spool/nagios/.creds/my_site.conf
   OK - Keystone API successfully tested.


Nagios configuration
--------------------

Once the probes are correctly installed and configured, the Nagios configuration can be modified to use them. The glance, cinder, neutron and nova probes require that Keystone is working. Therefore they should depend on the check_keystone_api probe.

First, add the following command definition to Nagios:

.. code-block::

   define command{
           command_name            check_keystone_api
           command_line            $USER1$/check_os_keystone $USER2$/$ARG1$
                                   ; $ARG1$ = site-config-file
           }
   
   define command{
           command_name            check_glance_api
           command_line            $USER1$/check_os_glance $USER2$/$ARG1$
                                   ; $ARG1$ = site-config-file
           }
   
   define command{
           command_name            check_cinder_api
           command_line            $USER1$/check_os_cinder $USER2$/$ARG1$
                                   ; $ARG1$ = site-config-file
           }
   
   define command{
           command_name            check_neutron_api
           command_line            $USER1$/check_os_neutron $USER2$/$ARG1$
                                   ; $ARG1$ = site-config-file
           }
   
   define command{
           command_name            check_nova_api
           command_line            $USER1$/check_os_nova $USER2$/$ARG1$
                                   ; $ARG1$ = site-config-file
           }
   
   define command{
           command_name            check_openstack_scenario
           command_line            $USER1$/check_os_scenario $USER2$/$ARG1$
                                   ; $ARG1$ = site-config-file
           }


In a standard CentOS 7 configuration, $USER1$ may be set to /usr/lib64/nagios/plugins and $USER2$ to /var/spool/nagios/.creds

Then, create a service definition for each OpenStack service to monitor:

.. code-block:: bash

   define service{
           name                    openstack-keystone-api
           use                     service-template,graph
           host_name               openstack.example.org
           service_description     OpenStack Keystone API
           check_command           check_keystone_api!my_site.conf
           }
   
   define service{
           name                    openstack-glance-api
           use                     service-template,graph
           host_name               openstack.example.org
           service_description     OpenStack Glance API
           check_command           check_glance_api!my_site.conf
           }
   
   define service{
           name                    openstack-cinder-api
           use                     service-template,graph
           host_name               openstack.example.org
           service_description     OpenStack Cinder API
           check_command           check_cinder_api!my_site.conf
           }
   
   define service{
           name                    openstack-neutron-api
           use                     service-template,graph
           host_name               openstack.example.org
           service_description     OpenStack Neutron API
           check_command           check_neutron_api!my_site.conf
           }
   
   define service{
           name                    openstack-nova-api
           use                     service-template,graph
           host_name               openstack.example.org
           service_description     OpenStack Nova API
           check_command           check_nova_api!my_site.conf
           }
   
   define service{
           name                    openstack-openstack-scenario
           use                     service-template,graph
           host_name               openstack.example.org
           service_description     OpenStack Scenario
           check_command           check_openstack_scenario!my_site.conf
           }

The glance, cinder, neutron and nova api probes depend on keystone. If Keystone is not working, these probes can be disabled using this configuration:

.. code-block:: bash

   define servicedependency{
            host_name               glance.example.org
            service_description     OpenStack Glance API
            dependent_host_name     keystone.example.org
            dependent_service_description   OpenStack Keystone API
            execution_failure_criteria      c,u
            notification_failure_criteria   n
            }
   
   define servicedependency{
            host_name               cinder.example.org
            service_description     OpenStack Cinder API
            dependent_host_name     keystone.example.org
            dependent_service_description   OpenStack Keystone API
            execution_failure_criteria      c,u
            notification_failure_criteria   n
            }
   
   define servicedependency{
            host_name               neutron.example.org
            service_description     OpenStack Neutron API
            dependent_host_name     keystone.example.org
            dependent_service_description   OpenStack Keystone API
            execution_failure_criteria      c,u
            notification_failure_criteria   n
            }
   
   define servicedependency{
            host_name               nova.example.org
            service_description     OpenStack Nova API
            dependent_host_name     keystone.example.org
            dependent_service_description   OpenStack Nova API
            execution_failure_criteria      c,u
            notification_failure_criteria   n
            }


License
=======

The source code of the probes is released under the Apache License, Version 2.0.


Hacking
=======

The source code is hosted on the `France Grilles Github project <https://github.com/francegrilles/monitoring-os>`_.

Issues are managed through the `Github ticketing system <https://github.com/francegrilles/monitoring-os/issues>`_.

If you want to provide code fixes or enhancement, please follow the `PEP 8
style guidelines <https://www.python.org/dev/peps/pep-0008>`_.

