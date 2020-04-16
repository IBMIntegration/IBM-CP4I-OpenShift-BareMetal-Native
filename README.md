# CP4I on IBM Cloud Bare Metal without Virtualisation

IBM Cloud Pak for Integration relies on Red Hat Openshift Container Platform (OCP) for its platform.  OCP features the ability to create the required infrastructure automatically on certain cloud platforms; this is called Installer Provisioned Infrastructure or IPI.  The alternative is User Provisioned Infrastructure or UPI.  IPI is not currently available on IBM Cloud so UPI must be used.  This document is therefore in three parts:

1. Creating infrastructure on IBM Cloud
2. Installing OCP
3. Installing CP4I

IBM Cloud does not provide virtual servers with Red Hat CoreOS, which is required for OCP 4.x, so bare metal servers must be used.

## Infrastructure Overview

An OCP cluster contains master nodes which host the control plane services, and worker nodes which host the workload.  UPI mandates at least three master nodes, and three worker nodes is recommended for high availability although the cluster can still operate with fewer.

### Topology
#### VLAN
Bare metal servers are not part of the Virtual Private Cloud offering.  Instead they are connected to a VLAN which has a private subnet.  Care must be taken when provisioning components to make sure the correct VLAN is selected as it cannot be changed later.  Some components default to 'auto-assigned' VLAN which may not be the same as the

#### Load Balancers                                            
Requests to the master nodes are distributed between the nodes via a load balancer.  The same is true for the worker nodes.  Internal facing and external facing load balancers are required for both the api and etcd traffic to master nodes, and the application traffic to the workers.

#### DNS
The cluster needs its own DNS nameserver with forward zones so that the nodes can resolve each other, and reverse zones so that the nodes can get their own hostname from their IP.

### Domain name
OCP 4.x requires a domain name to be registered.  This does not have to be registered in IBM Cloud.

#### NAT
The nodes in the cluster should have internet access, so a NAT gateway is required.

#### Boot server
OCP master nodes must run Red Hat CoreOS ; worker nodes can run RHEL or RHCOS.  IBM Cloud does not provide an image for bare metal servers, so a means to boot the installation ISO must be configured.  It is possible to use PXE, however this requires the use of DHCP and whilst it is possible to have DHCP enabled, it is not supported.  Therefore the ISO must be mounted to each server as a virtual boot device, and a good way to do this is by hosting the ISO on an SMB server which must be provisioned.

#### VPN
To install CoreOS on the nodes, the remote management console must be used to mount the ISO as a virtual CD.  The management consoles are on a private subnet, so to access this a VPN must be set up.  IBM Cloud provides this facility that can be used with freely available clients.
