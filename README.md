# CP4I on IBM Cloud Bare Metal without Virtualisation

IBM Cloud Pak for Integration relies on Red Hat Openshift Container Platform (OCP) for its platform.  OCP features the ability to create the required infrastructure automatically on certain cloud platforms; this is called Installer Provisioned Infrastructure or IPI.  The alternative is User Provisioned Infrastructure or UPI.  IPI is not currently available on IBM Cloud so UPI must be used.  This document is therefore in three parts:

1. Creating infrastructure on IBM Cloud
2. Installing OCP
3. Installing CP4I

IBM Cloud does not provide virtual servers with Red Hat CoreOS, which is required for OCP 4.x, so bare metal servers must be used.

## Infrastructure Overview

An OCP cluster contains master nodes which host the control plane services, and worker nodes which host the workload.  UPI mandates at least three master nodes, and three worker nodes is recommended for high availability although the cluster can still operate with fewer.

### VLAN
Bare metal servers are not part of the Virtual Private Cloud offering.  Instead they are connected to a VLAN which has an associated subnet.

### Load Balancers                                            
Requests to the master nodes are distributed between the nodes via a load balancer.  The same is true for the worker nodes.  Internal facing and external facing load balancers are required for both the api and etcd traffic to master nodes, and the application traffic to the workers.

### DNS
The cluster needs its own DNS nameserver with forward zones so that the nodes can resolve each other, and reverse zones so that the nodes can get their own hostname from their IP.

### Domain name
OCP 4.x requires a domain name to be registered, because a wildcard record needs to be created to point to the proxy load balancer, and this is only possible with DNS.

### NAT
The nodes in the cluster should have internet access, so a NAT gateway is required.

### SMB file server
OCP master nodes must run Red Hat CoreOS ; worker nodes can run RHEL or RHCOS.  In order to install an OS remotely an ISO must be made available on an SMB share.

### Web Server
During the installation process, CoreOS needs to download the full OS image and configuration files, and it must do this via HTTP.  Therefore an HTTP server needs to be created to host these files.

### VPN
To install CoreOS on the nodes, the remote management console must be used to mount the ISO as a virtual CD.  The management consoles are on a private subnet, so to access this a VPN must be set up.  IBM Cloud provides this facility that can be used with freely available clients.

## IBM Cloud
IBM Cloud virtual servers are not available with CoreOS pre-installed.  Custom OS images can be created but they need to be based on recognised OS.  Finally, virtualisation is not supported on virtual servers.  At time of writing then, bare metal is the only option.

### Networking
Bare metal servers are placed on a VLAN during creation which is associated with a subnet.  During post-install configuration CoreOS automatically selects the first network interface in its list.  If servers have both public and private interfaces it may select the private interface, leaving the node unable to connect to the internet.  Having a public interface would introduce security issues, so it is preferable to select private-only networking and configure a NAT gateway (see later).

A VLAN must  be created for the cluster before any servers are created.  Associating servers with VLANs must be done when the servers are provisioned as it cannot be done later.

These instructions do not create a firewall on the VLAN.

### Servers
#### Bare Metal
Bare metal servers are used here for bootstrap, master and worker nodes.  IBM Cloud does not provide an image for bare metal servers, so a means to boot the installation ISO must be configured.  It is possible to use PXE, however this requires the use of DHCP and whilst it is possible to have DHCP enabled via a support request, it is not supported.  Therefore the ISO must be mounted to each server as a virtual boot device, and a good way to do this is by hosting the ISO on an SMB server.  When provisioning, 'No OS' must be selected.

There are many server types available.  Some of the available options have configuration issues that prevent the CoreOS installer from booting.  These instructions use the following types which have been verified:

- Intel Xeon E3-1270 v6 4 core for master and boostrap nodes (please note the v6 - the other single processor options do not work)
- Intel Xeon 4110 16 core for worker nodes (the version **without** UEFI boot or GPU)

These servers use the SuperMicro IPMI console for management, remote operation and the mounting of ISOs as virtual devices.  The boot order may not be set to virtual CD by default, this can easily be changed by contacting support.  CoreOS does support UEFI boot or BIOS, however these instructions use and have been tested with BIOS.  Some servers not labelled as UEFI still have the option to boot from UEFI, and these must be set to boot in BIOS mode first to use the BIOS boot image as specified here.

By default, servers are configured with highly available network adapters using bonded networking.  It is possible to configure CoreOS to boot with bonded networking, but for simplicity these instructions do NOT use bonded networking, so the option labelled 'port redundancy' must be de-selected at provisioning time.

#### Virtual Servers
This guide will create a CentOS virtual server with both public and private network interfaces to host the following supporting systems:

- SMB server for the virtual ISO
- DNS server via BIND
- HTTP server via Apache for the CoreOS image and configuration files during installation
- NAT gateway for the nodes to access the internet

It can also be used as a bastion node to ssh into the bare metal servers once running.  The IPSec VPN should allow network access from your workstation but I had trouble with this.

It may also be helpful to create a test server with private only interface on the same subnet as the bare metal servers to test these systems.

## Detailed instructions

The following sources contributed to this guide:

- Red Hat documentation: https://docs.openshift.com/container-platform/4.2/installing/installing_bare_metal/installing-bare-metal.html#cluster-entitlements_installing-bare-metal
- Red Hat blog: https://www.openshift.com/blog/openshift-4-bare-metal-install-quickstart
- SMB configuration: https://www.howtoforge.com/samba-server-installation-and-configuration-on-centos-7
- IBM Cloud VPN access: https://cloud.ibm.com/docs/iaas-vpn?topic=iaas-vpn-getting-started
- IBM Cloud bare metal servers: https://cloud.ibm.com/docs/bare-metal?topic=bare-metal-getting-started
- Internet gateway NAT guide: https://linuxtechlab.com/turning-centosrhel-6-7-machine-router/
- Filetranspiler: https://github.com/ashcrow/filetranspiler
- BIND: https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-centos-7

Most of these instructions start from the Classic Infrastructure menu. From the IBM Cloud dashboard, click the hamburger menu and select Classic Infrastructure.  The menu on the left will be the starting point for these operations unless otherwise specified.

### 1. Create an SSH key (if required)
Use ssh-keygen on Mac or Linux to create an ssh key, the public part of which will be uploaded later to provide access to the servers.

### 2. Create a VLAN
Under Network, select IP Management then VLANs.  At the top of the screen select Order VLAN.  Select Private, then choose a data center near you - this data center will be referred to as 'your' data center for the rest of this process.  Supply a name for the VLAN such as 'ocp'.  At this point, it should be possible to create further VLANs in different data centers, and enable VLAN spanning to allow communication between them.  This guide will be updated pending testing of this feature.

Make a note of the subnet CIDR.

### 3. Create a Virtual Server (VSI) or two
Under Devices, select Device List.  On the top right, select Order Devices then select Virtual Server.  
- Choose your data center, and select 2 CPUs and 4Gb RAM.
- If you do not already have an SSH key stored in IBM Cloud, select 'add ssh key' and upload the key created earlier.  This key will now be available for subsequent servers.
- For the domain, choose something simple.  It does not have to be the same as the domain name registered for the cluster.
- Choose CentOS to follow these instructions.
- Add a second hard disk of size 250Gb.
- Set private security group to 'allow_all' and public security group to 'allow_ssh'.
- For the private VLAN, select the VLAN created earlier.  There should only be one subnet associated with it, so it will auto select.  This machine will be referred to as 'the VSI'.
- A second VSI can optionally be created on the private subnet (no public interface) with 1 CPU for the purposes of testing the network configuration.  CentOS 7 is suggested for the OS and instructions will be given for CentOS 7.  This machine will be referred to as the 'test server'.

### 4. Create the Bare Metal Servers
Return to the device list and select 'Order Device', this time selecting Bare Metal.  Seven machines will need to be created:  one bootstrap, three masters and in this scenario three workers.  

As mentioned above, at least one type of server did not boot with the CoreOS image.  The following server types were tested and found to work.  The sizings  are reasonable, but considerably more or less may be required depending on usage - see CP4I product requirements for the required capabilities at https://www.ibm.com/support/knowledgecenter/en/SSGT7J_20.1/install/sysreqs.html

For each server:

- Supply a name and domain according to your convention.  The domain does not need to be the same as your registered domain, it can be a local one.  If Quantity is more than one, numbers on the end of the hostname will be incremented, so if the hostname is worker01 and three are selected, the other two servers will automatically be created as worker02 and worker03.
- Choose Monthly billing, as some server types are not available with hourly.
- Choose your data center
- Select from 'Configurable servers'

| Role       | Category          | Server Type             | RAM  | Notes |
| -----------|-------------------|-------------------------|------| |
| Bootstrap  | Single Processor  | Intel Xeon E3-1270 v6   | 16Gb |  |
| Master     | Single Processor  | Intel Xeon E3-1270 v6   | 16Gb | |
| Worker     | Dual Processor    | Intel Xeon 4110         | 96Gb | **Not** GPU or UEFI |

- Choose your SSH key uploaded earlier from the drop-down.
- Choose 'No OS' for the operating system
- The standard HD option will be fine
- For network interface, select Private, and None for port redundancy
- Select your VLAN for the Private VLAN
- Accept the service agreement with the check box on the right and select 'Create'  Since these servers are not fast provisioning, they will take several hours to create.  This is a good time to have lunch.

You should end up with seven servers named something like the following:

- bootstrap01
- master01
- master02
- master03
- worker01
- worker02
- worker03

### 5. Set up the NAT Router on the VSI

NB at this point it is worth doing a few optional things to make life easier, because you will be logging into it a lot.

- Set up an entry in your /etc/hosts file to point to the VSI's public IP called something like ocp
- Copy your public key to the VSI to enable passwordless login.

You can now log in with `ssh root@ocp`.  To configure the router, use the following steps:

- Establish the names of public and private interfaces with `ip addr`, we will assume eth1 is public and eth0 is private.
- Run `sysctl -w net.ipv4.ip_forward=1z` to enable IP forwarding
- Add the line `net.ipv4.ip_forward = 1` to the file `/etc/sysctl.conf` to keep it enabled across reboots.
- `systemctl start firewalld`
- `firewall-cmd –permanent –direct –passthrough ipv4 -t nat -I POSTROUTING -o eth1 -j MASQUERADE -s XXX` where XXX is your subnet CIDR e.g. 10.123.231.0/26
- `systemctl enable firewalld`

This gateway can now be tested from the test server as follows:

- On the test server, edit the file `/etc/sysconfig/network-scripts/ifcfg-eth0` and add the line `GATEWAY=XXX` where XXX is the IP address of your VSI.
- Try and access the internet e.g. `ping www.bbc.co.uk`.

### 6. Load Balancers
Openshift requires four load balancers:  

### 7. Domain Name Registration
Register a domain name

### 8. Cluster DNS Services
IBM Cloud offers a DNS service, however it does not allow reverse records for private IPs, and OCP nodes need to do reverse lookups on their own IPs to establish their hostnames.  Other DNS solutions could be used, however these instructions use BIND set up on the VSI.

- On the VSI, install bind with `yum install bind bind-utuls`
- Edit /etc/named.conf and create an access control list by adding the following to the top of the file, where XXX is your private subnet CIDR:
```
acl openshift {
        XXX;
};
```
- In the `options` section, add or change the following lines:
```
listen-on port 53 { 10.113.180.174; };
allow-query     { localhost; openshift; };
```

At this point, you will need to know your your registered domain
