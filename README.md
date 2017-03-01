# Monitoring OpenStack

This repository provides a set of Nagios probes for monitoring an OpenStack Cloud.


## Requirement

To work properly, these OpenStack Nagios probes have the following requirement:
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
For each monitored OpenStack Cloud, a specific configuration file (ini format) needs to be created. It may be placed in the Nagios home directory. This configuration fileshould only be readable by the Nagios user:
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

The cacert parameter is not mandatory.


Before to configure Nagios to use this probes, it is wise to check that they are working properly:
```
# sudo -u nagios /usr/lib64/nagios/plugins/check_os_keystone /var/spool/nagios/.creds/my_site.conf
OK - Keystone API successfully tested.
```

### Nagios configuration

Once the probes are correctly installed and configured, the Nagios configuration can be modified to use them. The following command is defined:
```
define command{
        command_name            check_keystone_api
        command_line            $USER1$/check_os_keystone $USER2$/$ARG1$
                                ; $ARG1$ = site-config-file
        }
```

In a standard CentOS 7 configuration, $USER1$ may be set to /usr/lib64/nagios/plugins/ and $USER2$ to /var/spool/nagios/.creds

This service is created for each keystone service to monitor:
```
define service{
        use                     openstack-keystone-template
        host_name               keystone.example.org
        check_command           check_keystone_api!my_site.conf
        }
```

