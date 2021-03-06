## Installing Sahara on MOC OpenStack
*These instructions are deprecated. Sahara installation is now controlled by Puppet. 
Check out pull requests [74](https://github.com/CCI-MOC/kilo-puppet/pull/74) and 
[78](https://github.com/CCI-MOC/kilo-puppet/pull/78) to find out what to add to Hiera file.*

[Based on these instructions](https://access.redhat.com/documentation/en/red-hat-openstack-platform/8/installation-reference/chapter-11-install-the-data-processing-service)

Assume that all commands are run as root. Hopefully, a script will come soon to automate this process.  
     
The essential Sahara instructions are steps 1-13. Beyond that deals with typical extra setup needed 
based on what is expected in the typical MOC staging environment.
 1. **Install the necesssary packages**  
```shell
yum install -y openstack-sahara-api openstack-sahara-engine
```
 1. **Create the database** (replacing PASSWORD with desired password):  
```shell
mysql -e "CREATE DATABASE IF NOT EXISTS sahara; GRANT ALL ON sahara.* TO 'sahara'@'%' IDENTIFIED BY 'PASSWORD'; GRANT ALL ON sahara.* TO 'sahara'@'localhost' IDENTIFIED BY 'PASSWORD';"  
```
 1. Add the line `max_allowed_packet=256M` to /etc/my.cnf in the `[mysqld]` section then 
 restart MySQL with `systemctl restart mysqld`  
 1. **Add database info to sahara.conf** (replacing PASSWORD with the value you set):  
```shell
 openstack-config --set /etc/sahara/sahara.conf database connection mysql://sahara:PASSWORD@localhost/sahara
```
 1. **Configure database schema**  
```shell
sahara-db-manage --config-file /etc/sahara/sahara.conf upgrade head
```
 1. **Set up the necessary users, services, and endpoints** in keystone; 
 be sure to `source /root/keystonerc_admin` before continuing:  
```shell
keystone user-create --name sahara --pass PASSWORD
keystone user-role-add --user sahara --role admin --tenant services
keystone service-create --name sahara --type data-processing --description "OpenStack Data Processing"
keystone endpoint-create --service sahara --publicurl 'http://HOSTNAME:8386/v1.1/%(tenant_id)s' --adminurl 'http://HOSTNAME:8386/v1.1/%(tenant_id)s' --internalurl 'http://HOSTNAME:8386/v1.1/%(tenant_id)s' --region 'MOC_Kaizen'`  
```
 *Note that in steps 6-10, the command `openstack-config --set [FILENAME] [SECTION] [PARAMETER] [VALUE]` 
 is equivalent to editing the file at FILENAME and adding/changing the value of PARAMETER in SECTION to VALUE.*
 1. **Add keystone info to sahara.conf** (replacing HOSTNAME with the proper hostname):
```shell
openstack-config --set /etc/sahara/sahara.conf keystone_authtoken auth_uri http://HOSTNAME:5000/v2.0/  
openstack-config --set /etc/sahara/sahara.conf keystone_authtoken identity_uri https://HOSTNAME:35357 # make sure that this one is HTTPS
openstack-config --set /etc/sahara/sahara.conf keystone_authtoken admin_tenant_name services  
openstack-config --set /etc/sahara/sahara.conf keystone_authtoken admin_user sahara
openstack-config --set /etc/sahara/sahara.conf keystone_authtoken admin_password PASSWORD # (this is the password from step 5)
```
 1. **Configure RabbitMQ in sahara.conf**
```shell
openstack-config --set /etc/sahara/sahara.conf oslo_messaging_rabbit rabbit_use_ssl False
openstack-config --set /etc/sahara/sahara.conf oslo_messaging_rabbit rabbit_userid $(openstack-config --get /etc/nova/nova.conf oslo_messaging_rabbit rabbit_userid)
openstack-config --set /etc/sahara/sahara.conf oslo_messaging_rabbit rabbit_password $(openstack-config --get /etc/nova/nova.conf oslo_messaging_rabbit rabbit_password)
openstack-config --set /etc/sahara/sahara.conf oslo_messaging_rabbit rabbit_virtual_host /
```
 1. **Enable the use of neutron in sahara.conf** 
```shell
openstack-config --set /etc/sahara/sahara.conf DEFAULT use_neutron true
```
 1. **Enable logging in sahara.conf**  
```shell
openstack-config --set /etc/sahara/sahara.conf DEFAULT log_file /var/log/sahara/sahara.log
```
 1. **Enable plugins sahara.conf** (not limited to just these three)  
```shell
openstack-config --set /etc/sahara/sahara.conf DEFAULT plugins vanilla,hdp,cdh
```
 1. **Make Heat not have timeout issues** 
```shell
openstack-config --set /etc/sahara/sahara.conf DEFAULT heat_enable_wait_condition false
```
 1. **Configure iptables** : Add the line `-A INPUT -p tcp -m multiport --dports 8386 -j ACCEPT` to the file 
 `/etc/sysconfig/iptables`, and then restart the iptables service with 
```shell 
 systemctl restart iptables.service
```
 1. **Start the sahara services, and enable them to start at boot**:  
```shell
systemctl start openstack-sahara-api
systemctl start openstack-sahara-engine  
systemctl enable openstack-sahara-api
systemctl enable openstack-sahara-engine
```
 The following steps deal with the configuration of Heat. If your Openstack installation is based on the MOC puppet scripts 
 for staging environments, Heat probably isn't fully configured yet. 

 If at any time during steps 15-21 you experience unexpected behavior, switch over to 
 [these instructions](https://access.redhat.com/documentation/en/red-hat-openstack-platform/8/installation-reference/chapter-9-install-the-orchestration-service)

 Additionally, note that OpenStack versions Liberty and prior can function without heat, using the deprecated 
 "direct" orchestration service. 
 To bypass heat, set `infrastructure_engine=direct` in `[DEFAULT]` section of `sahara.conf` 
 1. `yum list *heat*` and **make sure that Heat was installed**
 1. **Check if Heat has database** with 
```shell 
 mysql -e "SHOW DATABASES LIKE 'heat'"
```
 1. **Check if Heat has SQL connection** with 
```shell 
 cat /etc/heat/heat.conf | grep mysql
```
 1. **Check if Heat has a user** with 
```shell 
 keystone user-list
```
 1. **Add Heat Role** : Run `keystone user-role-add --user heat --role admin --tenant services` 
 and hopefully you will find out that the user already has the role.  
 1. **Check if Heat has services** with `keystone service-list` for "heat" and "heat-cfn" services  
 1. **Make sure Heat has endpoints in keystone** with `keystone endpoint-list | grep PORT` with PORT being 8000 and 8004  
 **From step 22 onward, you are no longer checking existing configuration, but instead adding new functionalities and configurations.**
 1. `heat-manage db_sync` to **build Heat database**.  
 1. **Find Keystone admin token** with `cat /etc/keystone/keystone.conf | grep admin_token`  
 1. **Create Heat domain**
```shell
openstack --os-token ADMIN_TOKEN --os-url=https://HOSTNAME:5000/v3 --os-identity-api-version=3 domain create heat --description "Owns users and projects created by heat"
```
 ... this returns a domain ID which you need for next steps.  
 1. **Create admin user of heat domain** : 
```shell 
 openstack --os-token ADMIN_TOKEN --os-url=https://HOSTNAME:5000/v3 --os-identity-api-version=3 user create heat_domain_admin --password PASSWORD --domain DOMAIN_ID --description "Manage users and projects created by heat"
``` 
 ...this return a user ID which you need for the next step.  
 1. **Grant admin role** : 
```shell 
 openstack --os-token ADMIN_TOKEN --os-url=https://HOSTNAME:5000/v3 --os-identity-api-version=3 role add --user USER_ID --domain DOMAIN_ID admin`
```
 1. **Add the info you created to** `heat.conf` 
```shell 
 openstack-config --set /etc/heat/heat.conf DEFAULT stack_domain_admin_password PASSWORD stack_domain_admin USERNAME stack_user_domain_name DOMAIN
```
 1. **Restart services** with 
```shell 
 systemctl restart openstack-heat-api openstack-heat-engine
```
