---
published: true
date: '2023-02-28 10:47 +0530'
title: cnBNG CP (AIO) Single VM Deployment Guide - Any VM Environment
position: hidden
tags:
  - cnbng
  - cse
---

cnBNG Control Plane deployment in single VM in any NFVI environment is called as cnBNG CP AIO Manual Deployment. In this deployment cnBNG Control Plane is deployed in a single customized Ubuntu VM. This Ubuntu VM is pre-deployed using SMI base iso image and hence the deployment is called as semi automated or manual. Following are included in this deployment:
	- SMI cluster (Cisco CaaS)
	- CEE application (Application Infrastructure for Monitoring and Alerting)
	- cnBNG Control Plane application (Control Plane application for Cisco CUPS BNG)
    
This is to be noted that only SMI Ubuntu VM deployment in NFVI environment is manual, rest of the process to deploy SMI, CEE and cnBNG Control Plane is fully automated through SMI Deployer or Cluster Manager. 

For AIO deployment following steps are followed:
	1. Inception Server Cluster Manager deployment
	1. Base ISO Ubuntu VM deployment for cnBNG CP
	1. cnBNG CP base ISO Ubuntu OS customizations
	1. SSH Key generation
	1. SMI, CEE and cnBNG CP deployment using SMI Deployer

## Networking

![aio-networking1.png]({{site.baseurl}}/images/aio-networking1.png)

## Prerequisites: 
	- Inception Server (SMI Deployer)

## Step 1: Deploying Inception VM and Installing SMI Deployer

Refer to [Inception Server Deployment Guide](https://xrdocs.io/cnbng/tutorials/inception-server-deployment-guide/).

## Step 2: SMI Ubuntu VM Deployment

SMI Ubuntu VM can be deployed using any standard VM deployment procedure in a given NFVI environement. This procedure is fairly straight forward and simple. To give an idea on how the deployment of VM works following are the manual steps to deploy the VM in VMWare vCenter. Procedure to deploy the VM may differ based on the chosen NFVI environment. 

	1. Download the SMI Base ISO file and copy the file to the VM Datastore
	1. In the vCenter, select "Create a New Virtual Machine"
	1. Specify name of the VM and select the Datacenter
	1. Next select the host for the VM
	1. Select the datastore 
	1. Select compatibility (ESXi 6.7 or later)
	1. Select guest OS: Guest Family- Linux, Guest OS Version- Ubuntu Linux (64-bit)
	1. Customize Hardware:
		2. vCPU: 8, Memory: 16GB, New Hard disk: 100Gb
		2. Under Network: select management network ("VM Network" in most cases
		2. Click New CD/DVD Drive and do the following:
			3. Select Datastore ISO file option to boot from the SMI Base .iso file. Browse to the location of the .iso file on the datastore set in Step 1.
			3. In the Device Status field, select Connect at power on checkbox.
	1. After the VM boots up: login to the VM (user: cloud_user, password: Cisco_123). You will be prompted to change the password immediately
	1. Now setup Networking by editing /etc/netplan/50-cloud-init.yaml file. Here is a sample file config:
```
	network:
	    ethernets:
	        eno1:
	            dhcp4: true
	        enp0s3:
	            dhcp4: true
	        ens160:
	            dhcp4: false
	            addresses:
	               - 192.168.107.166/25
	            gateway4: 192.168.107.129
	            nameservers:
	                search: [cisco.com]
	                addresses: [72.163.128.140]
	        ens3:
	            dhcp4: true
	        eth0:
	            dhcp4: true
	    version: 2
```
**Note**: Sometimes interface is not shown as ens160, in that case it is a good idea to search for the interface using ifconfig -a command. Generally lower ens number is the first NIC attached to the VM, and higher number is the last.
{: .notice--info}


