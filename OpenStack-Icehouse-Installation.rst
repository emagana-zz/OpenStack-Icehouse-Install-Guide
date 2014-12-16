####
OpenStack Icehouse Installation - Multi Node
####

Welcome to OpenStack Icehouse installation manual !

This document is based on `the OpenStack Official Documentation <http://docs.openstack.org/icehouse/install-guide/install/apt/content/index.html>`_ for Icehouse. 

:Version: 1.0
:Authors: Edgar Magana
:License: Apache License Version 2.0
:Keywords: OpenStack, Icehouse, Neutron, Nova, CentOS 6.5, Glance, Horizon


===============================

**Authors:**

Copyright (C) `Edgar Magana <https://www.linkedin.com/profile/view?id=21754469&trk=nav_responsive_tab_profile>`_

Based on the work done by:

+ **Chaima Ghribi & Marouen Mechtri [OpenStack-Icehouse-Installation](https://github.com/ChaimaGhribi/OpenStack-Icehouse-Installation).


================================

.. contents::
   

Basic Architecture and Network Configuration
==========================================

In this installation guide, we cover the step-by-step process of installing Openstack Icehouse on Ubuntu 14.04.  We consider a multi-node architecture with Openstack Networking (Neutron) that requires three node types: 

+ **Controller/Networking Node** that runs Keystone, Glance, Nova, Horizon and Neutron management services as well as OpenvSwitch and DHCP agents for connecting virtual machines to external networks.

+ **Compute Node** that runs the virtual machine instances in OpenStack. 

We have deployed a single compute node (see the Figure below) but you can simply add more compute nodes to our multi-node installation, if needed.  


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


Configure Networking on Compute Node
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

Here we've installed the basic services (keystone, glance, nova,neutron and horizon) and also the supporting services 
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


* Install Icehouse Repos

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

* Create the signing keys and certificates and restrict access to the generated data: 

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

