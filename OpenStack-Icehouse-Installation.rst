####
OpenStack Icehouse Installation - Multi Node
####

Welcome to OpenStack Icehouse installation manual !


:Version: 1.0
:Authors: Edgar Magana
:License: Apache License Version 2.0
:Keywords: OpenStack, Icehouse, Neutron, Nova, CentOS 6.5, Glance, Horizon


===============================

**Authors:**

Copyright (C) `Edgar Magana <https://www.linkedin.com/profile/view?id=21754469&trk=nav_responsive_tab_profile>`_

Based on the work done by:

+ This document is based on the `OpenStack Official Documentation <http://docs.openstack.org/icehouse/install-guide/install/apt/content/index.html>`_ for Icehouse.
+ Chaima Ghribi & Marouen Mechtri `OpenStack-Icehouse-Installation <https://github.com/ChaimaGhribi/OpenStack-Icehouse-Installation>`_


================================

.. contents::
   

Basic Architecture and Network Configuration
============================================

In this installation guide, we cover the step-by-step process of installing Openstack Icehouse on CentOS 6.5.  We consider a multi-node architecture with Openstack Networking (Neutron) that requires two node types:

+ **Controller/Networking Node** that runs Keystone, Glance, Nova, Horizon and Neutron management services as well as OpenvSwitch and DHCP agents for connecting virtual machines to external networks.

+ **Compute Node** that runs the virtual machine instances in OpenStack. 

We have deployed a couple of compute nodes (see the Figure below) but you can simply add more compute nodes to our multi-node installation, if needed.

.. image:: https://github.com/emagana/OpenStack-Icehouse-Install-Guide/blob/master/images/openstack-icehouse-ref-architecture.png

So, let’s prepare the nodes for OpenStack installation!

Configure Networking on Controller Node
---------------------------------------

All operations are performed by root

* Edit network settings to configure interface eth0::

    vi /etc/sysconfig/network-scripts/ifcfg-eth0
    (modify the following values)

    # The management network interface
        ...
        ONBOOT=yes
        NM_CONTROLLED=no
        ...

* Restart network::

    service network restart


* Set the hostname::

    vim /etc/sysconfig/network
    HOSTNAME=controller


* Edit /etc/hosts::

    vim /etc/hosts

    #controller
    172.16.232.139       controller

    # compute1
    172.16.232.140       compute1


Configure Networking on Compute Nodes
---------------------------------------

All operations are performed by root

* Edit network settings to configure interface eth0::

    vi /etc/sysconfig/network-scripts/ifcfg-eth0
    (modify the following values)

    # The management network interface
        ...
        ONBOOT=yes
        NM_CONTROLLED=no
        ...

* Restart network::

    service network restart


* Set the hostname::

    vim /etc/sysconfig/network
    HOSTNAME=compute


* Edit /etc/hosts::

    vim /etc/hosts

    #controller
    172.16.232.139       controller

    # compute1
    172.16.232.140       compute1
    ...                  compute2


Verify connectivity
-------------------

We recommend that you verify network connectivity to the internet and among the nodes before proceeding further.

    
* From the controller node::

    # ping a site on the internet:
    ping www.openstack.org

    # ping the management interface on the compute node:
    ping compute1

* From the compute node::

    # ping a site on the internet:
    ping www.openstack.org

    # ping the management interface on the controller node:
    ping controller

    
Install 
=======


Controller Node
---------------

Here we've installed the basic services (keystone, glance, nova, neutron and horizon) and also the supporting services
such as MySql database, message broker (RabbitMQ), and NTP. 

	
Install the supporting services
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install VIM and NTP service (Network Time Protocol)::

    yum install vim ntp -y
    service ntpd start
    chkconfig ntpd on


* Install MySQL::

    yum install mysql mysql-server MySQL-python -y
    service mysqld start
    chkconfig mysqld on
    mysql_install_db
    mysql_secure_installation
    (set-up a root password for mysql)


* Under the [mysqld] section, set the following keys to enable InnoDB, UTF-8 character set, and UTF-8 collation by default::

    vim /etc/mysql/my.cnf

    [mysqld]
    bind-address = controller
    default-storage-engine = innodb
    innodb_file_per_table
    collation-server = utf8_general_ci
    init-connect = 'SET NAMES utf8'
    character-set-server = utf8

* Restart the MySQL service::

    service mysql restart


* Install Icehouse Repos::

    yum install yum-plugin-priorities -y
    yum install http://repos.fedorapeople.org/repos/openstack/openstack-icehouse/rdo-release-icehouse-3.noarch.rpm -y
    yum install http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm -y
    yum install openstack-utils -y

* Install RabbitMQ (Message Queue)::

    yum install rabbitmq-server
    service rabbitmq-server start
    chkconfig rabbitmq-server on



Install the Identity Service (Keystone)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* Install keystone packages::

    yum install openstack-keystone python-keystoneclient -y

* Create a MySQL database for keystone::

    mysql -u root -p

    CREATE DATABASE keystone;
    GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';

    exit;

* Edit /etc/keystone/keystone.conf::

     vim /etc/keystone/keystone.conf
  
    [database]
    connection = mysql://keystone:password@controller/keystone
    
    [DEFAULT]
    admin_token=ADMIN
    log_dir=/var/log/keystone
  

* Restart the identity service then synchronize the database::

    service openstack-keystone start
    chkconfig openstack-keystone on
    keystone-manage db_sync

* Check synchronization::
        
    mysql -ukeystone -ppassword
    use keystone;
    show TABLES;


* Define users, tenants, and roles::

    export OS_SERVICE_TOKEN=ADMIN
    export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
    
    #Create an administrative user
    keystone user-create --name=admin --pass=admin_pass --email=admin@domain.com
    keystone role-create --name=admin
    keystone tenant-create --name=admin --description="Admin Tenant"
    keystone user-role-add --user=admin --tenant=admin --role=admin
    keystone user-role-add --user=admin --role=_member_ --tenant=admin
    
    #Create a normal user
    keystone user-create --name=demo --pass=demo_pass --email=demo@domain.com
    keystone tenant-create --name=demo --description="Demo Tenant"
    keystone user-role-add --user=demo --role=_member_ --tenant=demo
    
    #Create a service tenant
    keystone tenant-create --name=service --description="Service Tenant"


* Define services and API endpoints::
    
    keystone service-create --name=keystone --type=identity --description="OpenStack Identity"
    
    keystone endpoint-create \
    --service-id=$(keystone service-list | awk '/ identity / {print $2}') \
    --publicurl=http://controller:5000/v2.0 \
    --internalurl=http://controller:5000/v2.0 \
    --adminurl=http://controller:35357/v2.0


* Create a simple credential file::

    vi admin_creds
    # Paste the following:
    export OS_USERNAME=admin
    export OS_PASSWORD=admin_pass
    export OS_TENANT_NAME=admin
    export OS_AUTH_URL=http://controller:35357/v2.0

* Create the signing keys and certificates and restrict access to the generated data::

    keystone-manage pki_setup --keystone-user keystone --keystone-group keystone
    chown -R keystone:keystone /etc/keystone/ssl
    chmod -R o-rwx /etc/keystone/ssl
        
* Test Keystone::
    
    #clear the values in the OS_SERVICE_TOKEN and OS_SERVICE_ENDPOINT environment variables        
     unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT

    # Load credential admin file
    source admin_creds

    keystone user-list
    keystone user-role-list --user admin --tenant admin


Install the image Service (Glance)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install Glance packages::

    yum install openstack-glance python-glanceclient -y


* Create a MySQL database for Glance::

    mysql -u root -p

    CREATE DATABASE glance;
    GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';

    exit;

* Configure service user and role::

    keystone user-create --name=glance --pass=service_pass --email=glance@domain.com
    keystone user-role-add --user=glance --tenant=service --role=admin

* Register the service and create the endpoint::

    keystone service-create --name=glance --type=image --description="OpenStack Image Service"
    keystone endpoint-create \
    --service-id=$(keystone service-list | awk '/ image / {print $2}') \
    --publicurl=http://controller:9292 \
    --internalurl=http://controller:9292 \
    --adminurl=http://controller:9292

* Update /etc/glance/glance-api.conf::

    vim /etc/glance/glance-api.conf

    [database]
    connection = mysql://glance:password@controller/glance

    [DEFAULT]
    rpc_backend = rabbit
    rabbit_host = controller

    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_host = controller
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = glance
    admin_password = service_pass

    [paste_deploy]
    flavor = keystone


* Update /etc/glance/glance-registry.conf::

    vim /etc/glance/glance-registry.conf

    [database]
    connection = mysql://glance:password@controller/glance

    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_host = controller
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = glance
    admin_password = service_pass

    [paste_deploy]
    flavor = keystone


* Restart the glance-api and glance-registry services::

    service openstack-glance-api start; service openstack-glance-registry start
    chkconfig openstack-glance-api on; chkconfig openstack-glance-registry on


* Synchronize the glance database::

    glance-manage db_sync

* Test Glance, upload the cirros cloud image::

    source creds
    glance image-create --name "cirros-0.3.2-x86_64" --is-public true \
    --container-format bare --disk-format qcow2 \
    --location http://cdn.download.cirros-cloud.net/0.3.2/cirros-0.3.2-x86_64-disk.img

* List Images::

    glance image-list


Install the compute Service (Nova)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install nova packages::

    yum install openstack-nova-api openstack-nova-cert openstack-nova-conductor \
    openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler python-novaclient -y


* Create a Mysql database for Nova::

    mysql -u root -p

    CREATE DATABASE nova;
    GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';

    exit;

* Configure service user and role::

    keystone user-create --name=nova --pass=service_pass --email=nova@domain.com
    keystone user-role-add --user=nova --tenant=service --role=admin

* Register the service and create the endpoint::

    keystone service-create --name=nova --type=compute --description="OpenStack Compute"
    keystone endpoint-create \
    --service-id=$(keystone service-list | awk '/ compute / {print $2}') \
    --publicurl=http://controller:8774/v2/%\(tenant_id\)s \
    --internalurl=http://controller:8774/v2/%\(tenant_id\)s \
    --adminurl=http://controller:8774/v2/%\(tenant_id\)s


* Edit the /etc/nova/nova.conf::

    vim /etc/nova/nova.conf

    [database]
    connection = mysql://nova:password@controller/nova

    [DEFAULT]
    rpc_backend = rabbit
    rabbit_host = controller
    my_ip = controller
    vncserver_listen = controller
    vncserver_proxyclient_address = controller
    auth_strategy = keystone

    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_host = controller
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = nova
    admin_password = service_pass


* Synchronize your database::

    nova-manage db sync

* Restart nova-* services::

    service openstack-nova-api start; service openstack-nova-cert start
    service openstack-nova-consoleauth start; service openstack-nova-scheduler start
    service openstack-nova-conductor start; service openstack-nova-novncproxy start
    chkconfig openstack-nova-api on; chkconfig openstack-nova-cert on
    chkconfig openstack-nova-consoleauth on; chkconfig openstack-nova-scheduler on
    chkconfig openstack-nova-conductor on; chkconfig openstack-nova-novncproxy on


* Check Nova is running. The :-) icons indicate that everything is ok!::

    nova-manage service list

* To verify your configuration, list available images::

    nova image-list


Install the network Service (Neutron)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install the Neutron server and the OpenVSwitch packages::

    yum install openstack-neutron openstack-neutron-ml2 python-neutronclient openstack-neutron-openvswitch -y

* Edit /etc/sysctl.conf to contain the following::

    vi /etc/sysctl.conf
    net.ipv4.ip_forward=1
    net.ipv4.conf.all.rp_filter=0
    net.ipv4.conf.default.rp_filter=0

* Implement the changes::

    sysctl -p

* Create a MySql database for Neutron::

    mysql -u root -p

    CREATE DATABASE neutron;
    GRANT ALL PRIVILEGES ON neutron.* TO neutron@'localhost' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON neutron.* TO neutron@'%' IDENTIFIED BY 'password';

    exit;

* Configure service user and role::

    keystone user-create --name=neutron --pass=service_pass --email=neutron@domain.com
    keystone user-role-add --user=neutron --tenant=service --role=admin

* Register the service and create the endpoint::

    keystone service-create --name=neutron --type=network --description="OpenStack Networking"

    keystone endpoint-create \
    --service-id=$(keystone service-list | awk '/ network / {print $2}') \
    --publicurl=http://controller:9696 \
    --internalurl=http://controller:9696 \
    --adminurl=http://controller:9696


* Update /etc/neutron/neutron.conf::

    vim /etc/neutron/neutron.conf

    [database]
    connection = mysql://neutron:password@controller/neutron

    [DEFAULT]
    core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
    service_plugins = neutron.services.l3_router.l3_router_plugin.L3RouterPlugin
    allow_overlapping_ips = True

    auth_strategy = keystone
    rpc_backend = neutron.openstack.common.rpc.impl_kombu
    rabbit_host = controller

    notify_nova_on_port_status_changes = True
    notify_nova_on_port_data_changes = True
    nova_url = http://controller:8774/v2
    nova_admin_username = nova
    # Replace the SERVICE_TENANT_ID with the output of this command (keystone tenant-list | awk '/ service / { print $2 }')
    nova_admin_tenant_id = SERVICE_TENANT_ID
    nova_admin_password = service_pass
    nova_admin_auth_url = http://controller:35357/v2.0

    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_host = controller
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass


* Configure the Modular Layer 2 (ML2) plug-in::

    vim /etc/neutron/plugins/ml2/ml2_conf.ini

    [ml2]
    type_drivers = gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch

    [ml2_type_gre]
    tunnel_id_ranges = 1:1000

    [securitygroup]
    enable_security_group = False

* Edit the /etc/neutron/dhcp_agent.ini::

    vim /etc/neutron/dhcp_agent.ini

    [DEFAULT]
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    use_namespaces = True
    dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf

* Create the /etc/neutron/dnsmasq-neutron.conf file::

    vim /etc/neutron/dnsmasq-neutron.conf

    dhcp-option-force=26,1454

* Kill any existing dnsmasq processes::

    pkill dnsmasq

* Edit the /etc/neutron/metadata_agent.ini::

    vim /etc/neutron/metadata_agent.ini

    [DEFAULT]
    auth_url = http://controller:5000/v2.0
    auth_region = regionOne

    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass
    nova_metadata_ip = controller
    metadata_proxy_shared_secret = helloOpenStack

* Configure Compute to use Networking::

    vim /etc/nova/nova.conf

    [DEFAULT]
    network_api_class=nova.network.neutronv2.api.API
    neutron_url=http://controller:9696
    neutron_auth_strategy=keystone
    neutron_admin_tenant_name=service
    neutron_admin_username=neutron
    neutron_admin_password=service_pass
    neutron_admin_auth_url=http://controller:35357/v2.0
    libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
    linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
    firewall_driver=nova.virt.firewall.NoopFirewallDriver
    security_group_api=neutron
    service_metadata_proxy = True
    metadata_proxy_shared_secret = helloOpenStack

* Create a symbolic link needed by the networking service initialization::

    ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini

* Populate the database::

    neutron-db-manage --config-file /etc/neutron/neutron.conf \
    --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade icehouse neutron

* Restart the Compute services::

    service openstack-nova-api restart
    service openstack-nova-scheduler restart
    service openstack-nova-conductor restart

* Restart the Networking service::

    service neutron-server start
    chkconfig neutron-server on

* Start the OVS service and configure it to start when the system boots::

    service openvswitch start
    chkconfig openvswitch on

* Add the integration bridge::

    ovs-vsctl add-br br-int

* Assign the right config file for OVS::

    cp /etc/init.d/neutron-openvswitch-agent /etc/init.d/neutron-openvswitch-agent.orig
    sed -i 's,plugins/openvswitch/ovs_neutron_plugin.ini,plugin.ini,g' /etc/init.d/neutron-openvswitch-agent

* Start the Networking services and configure them to start when the system boots::

    service neutron-openvswitch-agent start
    service neutron-dhcp-agent start
    service neutron-metadata-agent start
    chkconfig neutron-openvswitch-agent on
    chkconfig neutron-dhcp-agent on
    chkconfig neutron-metadata-agent on


Install the dashboard Service (Horizon)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install the required packages::

    yum install httpd memcached python-memcached mod_wsgi openstack-dashboard -y


* Edit /etc/openstack-dashboard/local_settings::

    vim /etc/openstack-dashboard/local_settings
    ALLOWED_HOSTS = ['*']
    OPENSTACK_HOST = "controller"
    CACHES = {
        'default': {
        'BACKEND' : 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION' : '127.0.0.1:11211'
        }
    }

* Ensure that the SELinux policy of the system is configured to allow network connections to the HTTP server::

    setsebool -P httpd_can_network_connect on

* Flush IPTables::

    iptables -F

* Reload HTTP and memcached::

    service httpd start; service memcached start
    chkconfig httpd on
    chkconfig memcached on

* Note::

    If you have this error: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1.
    Set the 'ServerName' directive  globally to suppress this message”

    Solution: Edit /etc/httpd/conf/httpd.conf

    vim /etc/httpd/conf/httpd.conf
    Add the following new line end of file:
    ServerName localhost

* Reload Apache and memcached::

    service httpd start; service memcached start


* Check OpenStack Dashboard at http://controller/horizon. login admin/admin_pass
