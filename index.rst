Introduction
============

The following document lays out the design and topology of the virtualization cluster of Rubin Observatory locations, base, summit, and Tucson.


Motivation
==========

New technologies and developments are been discovered at an outstanding pace. Virtualization became a standard for the industry many years ago, but it has been replaced by new technologies, like micro-services and containerize applications, however, there are still some services that need to be run in virtual machine.

The following are some of the critical services for Rubin Observatory that are required to run in virtual machines:

  - Several Windows servers such as Domain Controllers, Email, etc.
  - The platform controlling the HVAC system of the Summit
  - The Fire detection system of the Summit
  - The UPS platform of the summit
  - Cisco services such as the Wireless Lan Controller, Cisco Unified Communications Manager, etc.

Virtualization also provides an extra layer of availability for services running on containers.


Requirements
============

Clusters must meet the following criteria and recommendations:

  - Front-end Graphical User Interface, for easy VMs creation, manipulation, edition and deletion.
  - Connection to a centralized authentication backend, such as LDAP, OpenLDAP or FreeIPA.
  - The cluster size has to be an odd number.
  - Reliable support or at least wide community support.
  - A common storage pool that all cluster nodes can access at all time.
  - The storage pool has to be able to tolerate catastrophic events, such us disk failures, servers maintenance, failures, and network outages.
  - Each hypervisor must be able to enter maintenance, allowing all services and Virtual Machines to remain ongoing on a different hypervisor of the cluster.
  - If a node is abruptly taken offline, the Virtual Machines running in such node must be able and capable of live auto-migration.
  - If a node irrevocably shuts down - complete hardware failure - the node must be easily replaced with a new one, with no data corruption.


OpenSource vs Licenced Solution
===============================

There are several alternatives in the market to deploy a Virtualization Server, and another few for Virtualization Clusters. For this document, we'll be grouping them into 2 big groups OpenSource - no licence or support cost, with support from the OpenSource community - and the Licenced ones - licence and support cost.

OpenSource
----------

For the OpenSource analysis, we'll be using QEMU. QEMU - short for Quick EMUlator - is a generic open source machine emulator and virtualizer. When used as a virtualizer, is commonly used with KVM - Kernel-based Machine - which can virtualize x86, server and embedded PowerPC, 64-bit POWER, S390, 32-bit and 64-bit ARM, and MIPS guests.

In terms of requirements:

  - QEMU does have a GUI called Virtual Manager, that allows you to manage VMs running across the cluster.
    The problem is the access: in order to get to it, you need to either export display through an ssh session or access through remote desktop to the main Hypervisor. Also, simultanueos user access isn't doable, but it can be accomplish through third party applications, like VNC.
  - QEMU does not allow remote authentification directly into the Virtual Manager, but it can be done through third party applications.
  - QEMU allows live migration - with a common storage pool - but does not have any live-automigration feature. This means, that if a node shuts down or fails, the VM will fail as well, and it will not be loaded in another server. Also, if the node is lost for good, the VM will have to be recreated in KVM, but the OS and the data will remain, cause the VM's Virtual Drive is in the common storage pool.
  - Since live migration is an option, a node can be taken off for maintanance or replacement, without outages or downtime required.
  - The fact that is OpenSource has it advantages and disadvantages. On the Pro side, you have an entire community constantly checking for bugs and also supporting you in case your instance fails, as well as a lot of information online. The Con, OpenSource software is vulnerable to glitches and bugs, and eventhough there is a whole community to support, since you are not paying for seervice, no one is obligated to respond or help.
  - If used without a common storage pool, and only local server storage, it rises a big risk, cause not only live migration is not an option, but if the server and/or local storage fails, the Virtual Machine is lost for good, representing a dangerous Single Point of Failure (SPF).

VMWare
------

For the licenced solution, we are going to analyze VMWare's solution: vSphere + vCenter. vSphere is a compute virtualization platform, which along with vCenter, it allows to control multiple Virtualization Servers - that are running vSphere. VMWare offers all amount of services and features, which goes hand by hand with the costs.

In terms of requirements:
  - It offers a front-end Web User Interface (WUI) through the vCenter Server. This allows to manage, migrate, create, modify or delete VMs, as long with several other services.
  - Offers connection to remote Centralized Authentication Platforms.
  - It has a feature called vMotion, which allows live-automigration, meaning that if a node is scheduled for maintenance or powered off, the VM will automatically be redeployed in another node.
  - It comes with a 3 year support, which includes 24/7 online helpdesk and software updates.
  - vCenter requires of a Common Storage Pool to enable the live-automigration feature.

Design
======

It is expected that, since for VMWare you are paying for a service, they must provide support and guarantee a relatively bug-free environment, and eventhough QEMU is capable of doing almost everything that vSphere can, it is not recommended for an industrial usability; specially since the purpose of this cluster is to support and run essential/core services, we must be able to guarantee that neither the servers or storage represents a SPF. Taking under consideration both the benefits and disadvantages of both schemes, the proposed desgined is over VMWare.

The following diagram shows the proposed architecture that will sustain the Virtualization Platform in vCenter:

  .. figure:: /_static/vmware_cluster_design.png
     :name: vmware_cluster_design

     VMWare Cluster Design

  - The OS (ESXi or vSphere) is mounted in a RAID1, construct over 2x1.6TB SSD, with a failure toleration of 1 SSD and a total pool storage of 1.6TB.
  - 2x RAID5, construct over 4x2TB SSD , with a failure toleration of 1 SSD per RAID and a storage pool of 6TB (each RAID).
  - The RAIDs - from now on, VB (Virtual Block) - are hardware build and directly mapped to the OS.
  - Over the local storage - the RAID1, common to the OS - a CentOS Virtual Machine is created.
  - To each Virtual Machine - one per server - the local VB are mapped into them.
  - The VMs are interconnected and a Gluster File System is created, composed by 6 Blocks - gluster storage metric, been 1 Block a volume storage, such as one HDD/SDD, VB, VD and LVM. Only one Block is allowed per one of said units.
  - The newly created GlusterFS (Gluster File System), with a 3 Replica configuration, would have a Storage Pool of 12TB.
  - The Gluster Storage Pool is then mapped into vCenter Server, and serves as a common pool, for all servers, to allocate the VMs.


Failover and WCS
================

WCS stands for Worst Case Scenario. The suggested design takes under consideration the following scenarios:


System Disk Failure
-------------------

In the first scenario, we consider the loss of one of the two System Disks. By System Disks we are reffering to the drives from were the OS is mounted on. Also, in this case, were one of the Gluster's VM will be allocated:


  .. figure:: /_static/system_disk_failure.png
     :name: system_disk_failure

     Losing one System Disk


Since the System Disks are arrange in a RAID1, the system will continue to go one and a hot-swap can be performed to replace the fail drive. No downtime necessarry.


Data Disks Failure
------------------

The Data Disks consists on were the Virtual Machines "virtual drives" are allocate. Virtual Drive is a logical drive created by the virtualization agent, in which reserves a space disk on the common storage, but from the VM perspective, is a regular drive.

  .. figure:: /_static/data_disk_failure.png
     :name: data_disk_failure.png

     Losing one Data Disk

Similar to the RAID1, the RAID5 configuration used in the Data Disks tolerate 1 Drive per RAID arrangement. This means, that we can loose 2 drives per server - only one per RAID - and keep working as usual, allowing to perform hot-swap as well and no downtime.


1 VM or Server loss
-------------------

The base OS runs a modified Linux Kernel, that allows to run many tasks and services, but they are pretty limited for external services - externals to vSphere -, which is why a Virtual Machine is mounted on top of the System Disks, to be able to run services - such as gluster - and be able to monitor it as well.

  .. figure:: /_static/server_failure.png
     :name: server_failure.png

     Server Failure

What this means, is that is basically the same - only in this setup - to loose a server than to loose a VM. Since the replication of gluster is set to 3 - keeps one copy per node -, in case one Server/VM is powered off or distroyed, the gluster storage would still have quorum and the data would remain uncorrupted. The repair procedure does not contemplate timeout or downtime, but since there is going to be a remaping and data duplication when the Server/VM comes back online, stressfull operation with high I/O - such VMs creation - are not recommended to be performed.


Network Outage
--------------

In order to explain what would happen during a Network Outage, the "Network Layer" was add to the diagram:

  .. figure:: /_static/vmware_cluster_with_network.png
     :name: vmware_cluster_with_network.png

     VMWare Cluster with Network Connection

Each link - numbers 1, 2 and 3 over the left of the servers - are composed by 2 connections: A primary and a failover. Keep in mind this are not PC or VPC (Port-Channel or Virtual Port-Channel), but a failover, meaning that if the primary link is lost, the failover kicks in.

If both links - primary and secondary - are lost, we face a similar scenario than a Server Loss with a sutil difference: when the network connection to a server is lost, a new instance of the Virtual Machines that were running in that server, will be replicated into one of the others, but the one that was running will remain running. This will produce that the local data - from the gluster VM - and the data from the other nodes - the other gluster VMs - will form a discrepancy. Fortunately, Gluster operates in a quorum base, which means that if two out of three nodes have the same data, the data is overwritten in the one that differs, so when the Server comes back online, a syncronization process will start and the Virtual Machines that were migrated, will be destroyed in the recently recover node. This mechanism is provided by the vCenter VMWare platform called vMotion, that ensures that the Virtual Machines are always running and auto-migrate them if any of the mentioned events happened.


2 VMs or Servers loss
---------------------

As mentioned before, Gluster is based on a quorum algorithm, which means that if two out of three nodes are unreachable or down but still reachable whitin each other, there is a high chance of data corruption.

The failsafe mechanisms that gluster uses here, are based on: "I cannot reach one of my agents and I'm getting timeouts to the network" in the 2 isolated nodes, and in the one still connected "I don't have a quorum, due to only one of the three nodes is available", then what happens is the gluster storage pool will fall in a state called "Read Only" to prevent data corruption.

vMotion is going to attempt to migrate the Virtual Machines from the fallen servers to the live one, but gluster won't allow it in order to prevent data corruption or a phenomena called "Split-Brain". Split-Brain happens when the metadata from one node differs from another, and it takes an arbitrary node to act as arbiter.

In this fatalistic scenario, if the at least one of the servers can be placed back online, the gluster storage will start again and the live-migration will begin; but if neither of the two servers are recoverable, the only option is redeem the data, reconstruct the gluster and drop the data on top of the new gluster. This will cause an outage and downtime.


.. sectnum::
