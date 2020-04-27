# **CP4I on IBM Cloud Bare Metal without Virtualisation**

Ben Cornwell, Architect, Hybrid Cloud Integration

24/3/2020

## Introduction

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
Openshift requires load balancers for bootstrap related traffic, for control plane traffic, and for ingress traffic i.e. access to the user applications.  This guide uses the IBM local load banalcer service and will create four instances:

| Name             | Type                | Servers            | Ports       |
| -----------------| --------------------|--------------------|-------------|
| Internal Proxy   | Private to private  | Workers            | 80, 443     |
| External Proxy   | Public to private   | Workers            | 80, 443     |
| Internal API     | Private to private  | Bootstrap, masters | 6443, 22623*|
| External API     | Public to private   | Bootstrap, workers | 6443        |

 \* This rule should be removed after installation is complete along with the bootstrap server.

 Complete the following steps for each of the load balancers in the above table
 - On the Classic Infrastructure page

### 7. Domain Name Registration
Openshift 4.x requires a registered domain name for a cluster.  The installation program requires a cluster name, and this is prepended to the domain name e.g. mycluster.mydomain.com. Then, Openshift requires subdomains to be added to the DNS zone for the cluster to funciton, some of which are needed by external clients and some by internal.  The external names need to resolve to external IPs, and the internal names to internal IPs.

This guide uses a public DNS service for the public DNS names (in my case, Amazon Route53 since my domain was already registered there), and a local nameserver (BIND) in the cluster for the private names.

Create the following A records in your zone, assuming your cluster name will be mycluster and your domain is mydomain.com:

| Name | IP |
|------| ---|
| api.mycluster.mydomain.com | IP of external API load balancer |
| *.apps.mycluster.mydomain.com | IP of external proxy load balancer |

Reverse records are not necessary for the public DNS.

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

- At the end of the file add the following for the forward zone:
```
zone "mycluster.mydomain.com" IN {
 type master;
 file "mycluster.mydomain.com.db";
 allow-update { none; };
};
```

- Reverse zones are named with the significant IP octets in reverse, so if your subnet is 10.1.2.x then the zone shoudl be 2.1.10.in-addr-arpa
```
zone "2.1.10.in-addr.arpa" IN {
  type master;
  file "2.1.10.db";
  allow-update { none; };
};
```
- NB In the following examples, 'name' refers to the first part of the hostname because the rest of the domain name is defined at the top of the zone file.
- Create the zone file in for the forward zone in `/var/named` called `mycluster.mydomain.com.db` with the following content including your own values. Note that IBM Cloud load balancers have multiple redundant IP addresses, so two records are included.

```$TTL 86400
@ IN SOA ns1.softlayer.com. root.mycluster.mydomain.com. (
                       2020030800        ; Serial
                       7200              ; Refresh
                       600               ; Retry
                       1728000           ; Expire
                       3600)             ; Minimum

@                      86400    IN NS    ns1.softlayer.com.
@                      86400    IN NS    ns2.softlayer.com.

*.apps                 900      IN A     <internal proxy LB address 1>
*.apps                 900      IN A     <internal proxy LB address 2>
api                    900      IN A     <internal API LB address 1>
api                    900      IN A     <internal API LB address 2>
api-int                900      IN A     <internal API LB address 1>
api-int                900      IN A     <internal API LB address 2>
etcd-0                 900      IN A     <master 1 address>
etcd-1                 900      IN A     <master 2 address>
etcd-2                 900      IN A     <master 3 address>
<bootstrap name>       900      IN A     <bootstrap address>
<master 1 name>        900      IN A     <master 1 address>
<master 2 name>        900      IN A     <master 2 address>
<master 3 name>        900      IN A     <master 3 address>
<worker 1 name>        900      IN A     <worker 1 address>
<worker 2 name>        900      IN A     <worker 2 address>
<worker 3 name>        900      IN A     <worker 3 address>
<VSI name>             900      IN A     <VSI IP>
_etcd-server-ssl._tcp  900      IN SRV   0 10 2380 etcd-0
_etcd-server-ssl._tcp  900      IN SRV   0 10 2380 etcd-1
_etcd-server-ssl._tcp  900      IN SRV   0 10 2380 etcd-2
```

- Then create a reverse file matching the reverse zone definition above e.g. 2.1.10.db:

```
$TTL 86400
@ IN SOA ns1.softlayer.com. root.mycluster.mydomain.com. (
                       2020030722        ; Serial
                       7200              ; Refresh
                       600               ; Retry
                       1728000           ; Expire
                       3600)             ; Minimum

@                      86400    IN NS    ns1.softlayer.com.
@                      86400    IN NS    ns2.softlayer.com.

<bootstrap last octet>  IN PTR <bootstrap name>.;
<master 1 last octet>   IN PTR <master 1 name>.;
<master 2 past octet>   IN PTR <master 2 name>.;
<master 3 last octet>   IN PTR <master 3 name>.;
<worker 1 last octet>   IN PTR <worker 1 name>.;
<worker 2 last octet>   IN PTR <worker 2 name>.;
<worker 3 last octet>   IN PTR <worker 3 name>.;
```

- Restart the DNS service with `systemctl named restart`
- Test name resolution from your test server, if you have one.

### 9. VSI Firewall
- Identify your public and private interfaces.  On an IBM Cloud VSI the public interface should be eth1 and the private eth0.
- Add the private interface to the 'internal' group `firewall-cmd --permanent --zone=internal --change-interface=eth0`
- Add the public interface to the 'external' group `firewall-cmd --permanent --zone=external --change-interface=eth1`
- Add the SMB service to the internal zone `firewall-cmd --permanent --zone=internal --add-service=samba`
- Add HTTP to the internal zone `firewall-cmd --permanent --zone=internal --add-service=http`
- Add DNS to the internal zone `firewall-cmd --permanent --zone=internal --add-service=dns`
- Start or restart the firewall `systemctl start firewalld` or `systemctl restart firewalld`
- Enable the firewall permanently `systemctl enable firewalld`

### 10. SMB Server
The VSI needs to host an SMB share to allow the bare metal server management system to mount it.  It is also possible to mount ISOs from your local workstation, but it is less convenient in most cases as it requires extra software to be installed.

To set up SMB on the VSI:

- Install the packages with `yum install samba samba-client samba-common`
- Create a group for SMB users `groupadd smbgrp`
- Create an SMB user called e.g. coreos with the command `useradd coreos -G smbgrp`
- Set an SMB password for the user with `smbpasswd -a coreos`
- Create a folder to hold the shared files e.g. `/share/coreos`
- Change the permissions of the shared folder: `chmod 777 /share/coreos`
- Add the following to /etc/samba/smb.conf:

```
[coreos]
	path = /share/coreos
	valid users = @smbgrp
	guest ok = no
	writable = yes
	browsable = no
```
- Under the `[global]` section in smb.conf, add the line `ntlm auth = yes`.  This is an insecure protocol, but it is required because the IPMI console will try to use it when it mounts the ISO as a virtual DVD.
- Start the SMB services with `systemctl start smb.service` and `systemctl start nmb.service`
- Enable the services to start on boot with `systemctl enable smb.service` and `systemctl enable nmb.service`

### 11. HTTP Server
This is required to allow the nodes to access their configuration files when they are installing.
- On the VSI, install apache `yum install httpd`
- Start the server 'systemctl start httpd'
- Enable the service permanently `systemctl enable httpd`

### 12. Install docker
Docker is not specifically required for the installation process, however these instructions use a utility called filetranspiler which is written in Python, and distributed as both Python source and a container image.  In the spirit of the modern containerised world it was decided to use the container version which will be described here.  However those familiar with Python may wish to use the source directly.  Consult the github repo at https://github.com/ashcrow/filetranspiler for instructions on using Python directly.  We will proceed with docker here.

- Install docker with `yum install docker`
- Start docker with `systemctl start docker`
- Enable docker on restart with `systemctl enable docker`

### 13. Set up the VPN
The web-based IPMI console for managing the bare metal servers is available via a management IP address which is on the same subnet as the servers themselves.  To access this from your local workstation a VPN connection is required.

- Under the Classic Infrastructure menu in the IBM Cloud console, select the Network drop-down on the right hand side, then IPSec VPN.
- In the top right, click 'Order VPN', and select your data center from the list.  Wait a few minutes for the service to be provisioned.
- Once the VPN is available, click on its name in the list.  On the following configuration page it is not necessary to enter anything in the Remote Peer Address field.
- The Customer Subnets list is a white-list of the IPs from which connection is allowed.  Enter a suitable CIDR.
- The Hosted Private Subnets section is a list of the subnets to which this VPN will allow access.  Click on 'Add Hosted Private Subnet', then in the dialog box find the subnet on which your servers are and click the Add link.
- Click on Update Device in the bottom right.

Now the endpoint is configured, a username and password will be required to connect.  VPN access is granted to users via the access management page.

- At the top of the screen click on the Manage drop-down and select Access (IAM).  On the next screen click on Users.
- Select your user from the list.
- On the user details page, scroll down to VPN password and enter a password.
- By default you have access to all subnets on which the devices to which you have access are situated.  The account owner has access to all the devices.

To connect to the VPN, an IPSec VPN client is required such as MotionPro which is a free download.  The software requires an endpoint address, a username and password.

- To find your endpoint address, go to https://cloud.ibm.com/docs/iaas-vpn?topic=iaas-vpn-available-vpn-endpoints and select the appropriate endpoint for your data centre.
- The username and password are those configured in the IAM screen earlier in this section.

### 14. Obtaining the installation files
Openshift 4.x uses Red Hat Core OS.  To install CoreOS on a machine, the machine must boot from an ISO with configuration parameters suppled to a GRUB-like bootloader.  The ISO will then download an ignition file and an OS image which it will will write to the disk.  The ignition file contains configuration for the server.  When the machine is rebooted CoreOS will boot and join the cluster as specified in the ignition file.

There are two version of the OS disk image - one for BIOS and one for UEFI.  These instructions will use the BIOS version but the process should be the same for UEFI.

The ignition files are created by a utility called `openshift-install` which along with the ISO and the OS images is downloadable from Red Hat.

These instructions will conduct the installation from the VSI although it is possible to do this from a local workstation if desired, however the VPN must be correctly configured to allow this.

Visit https://cloud.redhat.com/openshift/install/metal/user-provisioned to download the following:
- The Openshift installer (Linux version)
- Your pull secret
- The Openshift command line tool (oc) (Linux version)

Click on the Download RHCOS link to get to the download archive.  From this page, download the following, where <version> is the appropriate version:
- rhcos-<version>-x86_64-installer.x86_64.iso
- rhcos-<version>-x86_64-metal.x86_64.raw.gz

Place the files in the appropriate locations
- Copy the installer ISO `rhcos-<version>-x86_64-installer.x86_64.iso` to the SMB share e.g. `/share/coreos`
- Copy the disc image `rhcos-<version>-x86_64-metal.x86_64.raw.gz` to the HTML server directory `/var/www/html`

### 15. Configuring installation

- Put the openshift-installer executable on the VSI.
- Create a file called install-config.yaml, and add the following content, substituting the appropriate values.  NB the number of worker nodes in this file must be zero even though we are adding worker nodes later.  Make sure that the cidr of the cluster network does NOT overlap with the subnet of your bare metal machines.  The service network can be left as it is.  The SSH key is the public part of the one created in step 1.

```
apiVersion: v1
baseDomain: mydomain.com
compute:
- hyperthreading: Enabled   
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled   
  name: master
  replicas: 3
metadata:
  name: mycluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/9
    hostPrefix: 26
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: 'xxx'
sshKey: 'ssh-rsa xxx'
```

- Create an install directory e.g. `install01`.  This is the working directory for the openshift installer.  **Make a copy** of install-config.yaml and place it in this directory.  The openshift-install program consumes this file, so if the installation fails and needs to be run again, it is useful to have a copy.
- Create manifests `openshift-install create manifests --dir=install01`.  Ignore the warning about no compute nodes.
- Edit `install01/manifests/cluster-scheduler-02-config.yml` and change the parameter `mastersSchedulable` to `false`.
- Create ignition config files with `openshift-install create ignition-configs --dir=install01`

This will create bootstrap, master and worker ignition files that are suitable for use with DHCP.  However this cluster must use static IPs, so each node must have its own ignition file with that static IP written into it.  The static IP must be the same IP as assigned to the server by IBM Cloud.

Ignition files are not human readable, so it is best to use a utility for this.  As mentioned above, these instructions will use filetranspiler available from https://github.com/ashcrow/filetranspiler - but other tools may be available.  Filetranspiler works by taking files from a 'fakeroot' directory and including them into the ignition file.

- Make a directory e.g. `filetranspiler` and change to it.
- `git clone https://github.com/ashcrow/filetranspiler`
- Build the docker image `docker build . -t filetranspiler:latest`
- Make another directory to contain the fake roots e.g. `fakeroots`

The following tasks are repetitive and may be scripted, and a script to help is available in this repository at http://XXXX.  The steps in the script will be described below.  For *each machine* i.e. bootstrap, three masters and three workers, do the following steps:

- Create a standard interface configuration file for the static network in the  fakeroot directory
```
cat > fakeroots/<machine name>/etc/sysconfig/network-scripts/ifcfg-eno1 << EOF
DEVICE=eno1
BOOTPROTO=static
ONBOOT=yes
IPADDR=<ip address>
PREFIX=<subnet prefix>
NETMASK=<netmask>
GATEWAY=<VSI IP>
DNS1=<VSI IP>
DNS2=10.0.80.11
DOMAIN=mycluster.mydomain.com
DEFROUTE=yes
IPV6INIT=no
HOSTNAME=<node name>.mycluster.mydomain.com
EOF
```

- Run filetranspiler, substituting machine type for boostrap, master or worker:
```docker run --rm -ti --volume `pwd`:/srv:z filetranspiler -i install01/<machine type>.ign -f fakeroots/ocpmaster41 -o install01/<machine name>.ign```

After this there should be seven new ign files in the install01 directory.  
- Copy these files to /var/www/html

### 16. Initiate CoreOS Installation
Installing CoreOS is a two stage process.  Firstly, the system boots from the ISO.  At this point, basic installation parameters must be entered on the command line to enable networking.  The system then downloads an ign file and an OS image from the HTTP server, writes the image to the specified disk and configures the system according to the ign file with networking and the cluster details.  When the system reboots from the disk, the system will adopt the role defined in the ign file, either bootstrap, master or worker.  The master and worker nodes will contact the bootstrap node which will orchestrate the assembly of the cluster.

**NB** The process of booting the machine from the ISO and entering parameters requires using the IPMI management console from your workstation.  This console presents a virtual screen that shows the boot up screen and allows the user to type in parameters.  However, it is unfortunately NOT possible to copy and paste in or out of the virtual screen.  The configuration strings are long and mis-typing is easy.  Therefore it is strongly recommended to use an auto-typing tool.  On a Mac this is possible via Applescript, see https://www.sythe.org/threads/auto-typer-script-for-mac/ for an example.

- Connect the VPN via your IPsec client as described in section 13.

For each node - bootstrap, three masters and three workers (in that order), complete the following steps:

- Go to the Classic Infrastructure page on IBM Cloud and select Devices then Device List.  Select a device - starting with the bootstrap device.
- Select Remote Management from the left hand menu.  Under Management Details there is a password - select Show and then copy the password.
- From the Actions drop-down in the top right, select KVM Console.  In the resulting log in screen, enter 'root' for the user and paste the password.  The IPMI console should appear.
- Under Remote Control select Power Control.  Use the menu to power the server off if it is running.
- Under Virtual Media, select CD-ROM image.  There should be listed three devices with no disk emulation set.  If not, contact support to un-mount any unwanted images.
- Enter the details of the SMB share:  for host, enter the VSI IP; for the path enter the share name and filename of the ISO e.g. /coreos/rhcos-<version>-x86_64-installer.x86_64.iso; enter the username and password set for the SMB user in step 10.
- Click Save then Mount.  If the mount was successful, Device 1 in the list of devices should report that an image is mounted.
- Under Remote Control at the top of the screen, select iKVM/HTML5.  Then on the next screen click the iKVM/HTML5 button.
- A pop-up window will appear with a menu and a blank screen.  In the menu select Power Control then Set Power On.  The CoreOS installer should boot.
- A boot screen should appear.  This will vary depending on the specific type of server being used.  There should be an option to enter boot parameters - sometimes this is accessed by pressing TAB, and sometimes by pressing 'e'.  Either way, select the option.
- The general format for the boot parameter string is as follows
```
ip=machine IP::gateway:subnet:hostname:network adapter:none:nameserver coreos.inst.install_dev=install device coreos.inst.image_url=OS image URL coreos.inst.ignition_url=ignition file URL
```

- Therefore in this installation, for the boot parameters enter the following text **all on one line**
```
ip=<machine IP>::<VSI IP>:<subnet netmask>:<machine short name>:eno1:none:<VSI IP> coreos.inst.install_dev=sda coreos.inst.image_url=http://<VSI IP>/rhcos-<version>-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://<VSI IP>/<ign filename>
```
- Press the key to boot the machine as prompted - either 'e' or Ctrl-X

The machine should boot, then after configuring its network it should download the ign file and the OS image.  After this is complete it will write the image to disc and then reboot automaticaly.  In the traditional style for installing an OS, the virutal ISO must be ejected before the machine has rebooted, to allow it to boot from the hard disk.  Use the IPMI console to unmount the CD as described previously.

### 16. Monitoring Installation Progress
When a node comes online, it will contact the bootstrap node and connect itself to the cluster.  This may take some time.  The process can be monitored with the openshift-install command.

- Monitor the bootstrap process with `./openshift-install --dir=install01 wait-for bootstrap-complete --log-level=info`

At this point you can remove the bootstrap node from the load balancer.
- On the IBM Cloud Classic Infrastructure page click on Network on the left hand menu then on Load Balancers.

- When this has completed, run `./openshift-install --dir=install01 wait-for install-complete --log-level=info`

### 17. Post-install Tasks
