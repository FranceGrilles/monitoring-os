# Monitoring OpenStack

This repository provides a set of Nagios probes for monitoring an OpenStack Cloud.


## Requirement

To work properly, these OpenStack Nagios probes have the following requirements:
* Python 2.7
* python-openstacksdk

They have been successfully tested with OpenStack Mitaka and Newton.

## Installation

The installation is quite simple. The probes have to be placed into the
Nagios probe directory and made usable by the Nagios user:
```
# cp plugins/check_os_* /usr/lib64/nagios/plugins/
# chown -R nagios /usr/lib64/nagios/plugins/check_os_*
```

## Configuration

### OS probes configuration
For each monitored OpenStack Cloud, a specific configuration file (ini format) needs to be created. It may be placed in the Nagios home directory. This configuration file should only be readable by the Nagios user:
```
# cat /var/spool/nagios/.creds/my_site.conf
[keystone_authtoken]
username = demo
password = secure_password
project_name = demo
user_domain_name = default
project_domain_name = default
auth_uri = https://keystone.example.org:5000/v3
cacert = /etc/pki/tls/certs/CA.pem
# ls -l /var/spool/nagios/.creds/my_site.conf
-rw------- 1 nagios nagios 224 Mar  1 08:51 /var/spool/nagios/.creds/my_site.conf
```

Please note that:
* The cacert parameter is not mandatory.
* glance, cinder, neutron and nova API parameters are automatically retrieved from Keystone endpoints catalog

Before configuring Nagios to use these probes, it is wise to check that they are working properly:
```
# sudo -u nagios /usr/lib64/nagios/plugins/check_os_keystone /var/spool/nagios/.creds/my_site.conf
OK - Keystone API successfully tested.
```

### Nagios configuration

Once the probes are correctly installed and configured, the Nagios configuration can be modified to use them. The glance, cinder, neutron and nova probes require that Keystone is working. Therefore they should depend on the check_keystone_api probe.

First, add the following command definition to Nagios:
```
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

```

In a standard CentOS 7 configuration, $USER1$ may be set to /usr/lib64/nagios/plugins and $USER2$ to /var/spool/nagios/.creds

Then, create a service definition for each OpenStack service to monitor:
```
define service{
        name                    openstack-keystone-api
        use                     service-template,graph
        host_name               keystone.example.org
        service_description     OpenStack Keystone API
        check_command           check_keystone_api!my_site.conf
        }

define service{
        name                    openstack-glance-api
        use                     service-template,graph
        host_name               glance.example.org
        service_description     OpenStack Glance API
        check_command           check_glance_api!my_site.conf
        }

define service{
        name                    openstack-cinder-api
        use                     service-template,graph
        host_name               cinder.example.org
        service_description     OpenStack Cinder API
        check_command           check_cinder_api!my_site.conf
        }

define service{
        name                    openstack-neutron-api
        use                     service-template,graph
        host_name               neutron.example.org
        service_description     OpenStack Neutron API
        check_command           check_neutron_api!my_site.conf
        }

define service{
        name                    openstack-nova-api
        use                     service-template,graph
        host_name               nova.example.org
        service_description     OpenStack Nova API
        check_command           check_nova_api!my_site.conf
        }
```

The glance, cinder, neutron and nova api probes depend on keystone. If Keystone is not working, these probes can be disabled using this configuration:

```
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
```

