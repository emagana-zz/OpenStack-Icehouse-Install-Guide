OpenStack-Icehouse-Install-Guide
================================

In this installation guide, we cover the step-by-step process of installing Openstack Icehouse on CentOS 6.5.  We consider a multi-node architecture with Openstack Networking (Neutron) that requires two node types:

+ **Controller/Networking Node** that runs Keystone, Glance, Nova, Horizon and Neutron management services as well as OpenvSwitch and DHCP agents for connecting virtual machines to external networks.

+ **Compute Node** that runs the virtual machine instances in OpenStack.

Note:
This guide is based on the work done by Chaima Ghribi and Marouen Mechtri presented here [OpenStack-Icehouse-Installation](https://github.com/ChaimaGhribi/OpenStack-Icehouse-Installation).
