####
OVS-Reference-Configuration - OpenStack Icehouse
####

Reference OVS configuration for multi-node deployment with one interface!


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


OVS Configuration on Controller Node
---------------------------------------

After running "ovs-vsctl show" on the Controller node, the output is::

Bridge br-tun
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port br-tun
            Interface br-tun
                type: internal
        Port "gre-ac10e88d"
            Interface "gre-ac10e88d"
                type: gre
                options: {in_key=flow, local_ip="172.16.232.139", out_key=flow, remote_ip="172.16.232.141"}
    Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
    Bridge br-int
        fail_mode: secure
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "tapd0f82038-75"
            tag: 1
            Interface "tapd0f82038-75"
                type: internal
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "2.1.3"


OVS Configuration on Compute Node
---------------------------------------

After running "ovs-vsctl show" on the Compute node, the output is::

Bridge br-int
        fail_mode: secure
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "qvof2bf0a33-a6"
            tag: 1
            Interface "qvof2bf0a33-a6"
        Port br-int
            Interface br-int
                type: internal
    Bridge br-tun
        Port "gre-ac10e88b"
            Interface "gre-ac10e88b"
                type: gre
                options: {in_key=flow, local_ip="172.16.232.141", out_key=flow, remote_ip="172.16.232.139"}
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
    ovs_version: "2.1.3"

