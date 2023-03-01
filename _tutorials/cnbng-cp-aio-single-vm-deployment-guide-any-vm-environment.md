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
1. SMI, CEE and cnBNG CP deployment using SMI Deployer

## Networking

![aio-networking1.png]({{site.baseurl}}/images/aio-networking1.png)

## Prerequisites: 
- Inception Server (SMI Deployer)

## Step 1: Deploying Inception VM and Installing SMI Deployer

Refer to [Inception Server Deployment Guide](https://xrdocs.io/cnbng/tutorials/inception-server-deployment-guide/).

## Step 2: SMI Ubuntu VM Deployment (Manual)

SMI Ubuntu VM can be deployed using any standard VM deployment procedure in a given NFVI environement. This procedure is fairly straight forward and simple. To give an idea on how the deployment of VM works following are the manual steps to deploy the VM in VMWare vCenter. Procedure to deploy the VM may differ based on the chosen NFVI environment. 

1. Download the SMI Base ISO file and copy the file to the VM Datastore
1. In the vCenter, select "Create a New Virtual Machine"
1. Specify name of the VM and select the Datacenter
1. Next select the host for the VM
1. Select the datastore 
1. Select compatibility (ESXi 6.7 or later)
1. Select guest OS: Guest Family- Linux, Guest OS Version- Ubuntu Linux (64-bit)
1. Customize Hardware:
	1. vCPU: 8, Memory: 16GB, New Hard disk: 100Gb
	1. Under Network: select management network ("VM Network" in most cases
	1. Click New CD/DVD Drive and do the following:
		1. Select Datastore ISO file option to boot from the SMI Base .iso file. Browse to the location of the .iso file on the datastore set in Step 1.
		1. In the Device Status field, select Connect at power on checkbox.
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

## Step 3: Base ISO Ubuntu OS customization

1. SSH login to cnBNG CP AIO Ubuntu VM which was deployed in Step-2
1. Now change the hostname of the VM to: <your-cnbng-cp-cluster>-aio, using:
	```
		sudo hostnamectl set-hostname <your-cnbng-cp-cluster>-aio
	e.g.
		sudo hostnamectl set-hostname cnbng-cp-lab1-aio
	```
1. Logout of the VM and login again to see hostname changes are reflected
1. Make the hostname persistent even after reload by adding "preserve_hostname: true" to /etc/cloud/cloud.cfg file if not added already or change the setting to true from false if already present
1. (optional) Replace default hostname for VM with the one you set into /etc/hosts file
1. Verify that the hostname is persistent even after reboot of the VM
1. SSH Key Generation
	1. SSH Login to Inception VM
	1. Generate SSH key using: 
    ```
		ssh-keygen -t rsa
    ```
	1. Run ssh-copy-id command, which will copy ssh keys to Base ISO Ubuntu Image for cnBNG CP AIO e.g.
    ```
		ssh-copy-id cloud-user@192.168.107.166
    ```
	1. Verify that the login using keys is working by logging to cnBNG CP AIO VM
    ```
		ssh cloud-user@192.168.107.166
    ```

## Step 4: SMI, CEE and cnBNG CP deployment using SMI Deployer

1. Login to inception VM
1. Note down ssh keys in a file from .ssh/id_rsa (private) and ./ssh/id_rsa.pub (public)
1. Remove line breaks from private key and replace them with string "\n"
1. Login to SMI Cluster Deployer or Cluster Manager on inception VM:
   ```
	ssh admin@localhost -p 2022
   ```
1. Create environment configs:
   ```
	environments manual
	   manual
	exit
   ```
1. Create Cluster configs:
<div class="highlighter-rouge">
<pre class="highlight">
<code>
  clusters cnbng
   environment manual
 addons ingress bind-ip-address 192.168.107.166
 addons ingress enabled
 addons istio enabled
 configuration master-virtual-ip 192.168.107.166
 configuration master-virtual-ip-interface ens160
 configuration pod-subnet    192.202.0.0/16
 configuration allow-insecure-registry true
 node-defaults initial-boot default-user cisco
 node-defaults initial-boot default-user-ssh-public-key <mark>"ssh public key from step-2"</mark>
 node-defaults k8s ssh-username cloud-user
 node-defaults k8s ssh-connection-private-key "-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEAtuZ56pW6nMf0e8ZMAjsKnj89YIpEfVyZ31GkrqObUua70egN\nOqDdWF8ToUHQrPnwDP/E4BplhtdiP2n/Dq8e9Xf8GRHX6KNXC6dhl5c6MKaBoh5A\nRDj8aVEfRfoDwpp5L0clg01EP0IIqkTIrSqScAheNfiKSptE5OleDS4I4mxfZsDR\nKnhAOtjgU/aHNGFDt1rYJ+QifAcp97ouyiNfsUsJwifLwBN3/EisbTH3rD0Z5fUO\n1UYmOLFW3vEI8voXAHk4BnNad6QGACCfENmCVZW3RBovr5wSdc22XXDTwp1gzJN3\nqlDcgQ6JfvE8i2JyL44Fb8KiZVvDgongXj9RGwIDAQABAoIBAB4Tjm7WCmbntrt3\n413miZt2OMicVCDtTlxb16HkQ5GBYddlum8urtduYxL8eK1JOIFauexESve+iWh2\nLLwkbgndnjYdKg0WdyTydGjyNF51sxGOufC+EjvbXDIsp9ujfVQZ9gA+f3+Lg1NE\nll9rhcMojR2A7nTQTab6/T1bmZhqBKcsX0DEFiv2o64/QeXSijzXMTLH1jASJKC7\ns+xyu3AmwE3LrecQah8xe+zZSefFWcPCQHNRWflVGUnKWFk58qbdv2X/MSzLvkYu\nU2A0RJX/U4ZOIWM1J8RUPv4MKFEgJ7LA1OT3UtOC+S6Bemlx6h/VWhoJnNhcwvm9\nd5vJoDECgYEA2cjmN0U/eeGlFZDLGRtQ4HVoUkQFMosHk1KQhEj3NZzIZnI9/VFf\njBkfFlP0bFajKZgCwwctb6EcJaRY6JsnKUuTHT5ZikVddFVU1hJkU/GwwHOX9cCM\nFuJi+yxc1G27pE82xWaUuiVRJ8wbQOkPMWrrvaR8eyUa6X8CRk3+JBkCgYEA1v6I\nhRtrxtiZ32hVr33X08vg2VDazgXAdOrBRimxkjjBPlHWekDZePkxKT+fTIzlLffQ\nIemn/gFJo4+YhuBzmv+k5I6Bo6I1M/VSoAFXKShwghg7KSfAQ5BH8oIr9PG4XEl4\nWs5cGb1NrCgqmQISLS1Cn5q6H+NDvdpgckryJVMCgYAQyBRFSga8I5EO+ltMEfjH\ncwSY4jjsTh5FUeVk7CJwdSZUDpWMQYr1RrJIjCuXdY2ZFOeRk6oCog2DMQjQ07PO\n0M4DQNyxdOrgnfqtjDlC5qrSCZY6D5473TH3XNHCZLpCzP/Rcjgfp+R7BpVLCSps\nimqj8FrPOmq6d1j7heMBcQKBgE+ltj/Rm8jrv32DcpL0BPwCwMbhbF38xYLK4VUz\n5wProLOMr+9UjPyDHNJSLpq2a8Tu1J1rqX+xTG2aqf/1sP5QDO9bV+2eDyWzkauT\nM44c3Cll/qzNfC3Lisvtq4kv74PI+Bxz7Kzgc6D+tGFA4ij4ZoEoWiGsGRGBkE9n\nMnPfAoGBAK9q6auE3HXkZ+VVD5LH2td7oze+7o1vrs2O/175oTzf/drcY5S59lK4\nudOjDIXNA/Lot080YONX7lbe2Eept21YektS9xpZ1Qld8NGoPFYsrYjqGYbojFbN\nYJMSEpOpbber+Mca7NenEusL5hK87sQCOP6OJ2RwjkGUVlraVpgQ\n-----END RSA PRIVATE KEY-----"
 
  node-defaults netplan template "    network:\n        version: 2\n        ethernets:\n            ens160:\n                dhcp4: false\n                addresses:\n                - {{K8S_SSH_IP}}/24\n                routes:\n                - to: 0.0.0.0/0\n                  via: 192.168.107.129\n                nameservers:\n                    addresses:\n                    - 64.104.128.236\n                    - 64.102.6.247\n                    search:\n                    - cisco.com\n\n\n"
 
 node-defaults os ntp enabled
 node-defaults os ntp servers <mark>clock.cisco.com</mark>
 exit
  ```
In the above config, change ntp server to the one available in the lab. Also netplan should be as per the netplan configured in the VM. 
{: .notice--info}
  
1. Create AIO Node config:
  ```
clusters cnbng
 nodes cp
  k8s node-type master
  k8s ssh-ip 192.168.107.166
  k8s node-labels disktype ssd
  exit
  k8s node-labels smi.cisco.com/node-type oam
  exit
  initial-boot default-user cloud-user
  initial-boot default-user-password <<your-password>>
 exit
exit
  ```
1. Setup software repository for CEE and cnBNG CP
```
software cnf bng
 url             http://192.168.107.148/images/CP/cp_30sep21/bng/bng.2021.04.m0.i74.tar
 allow-dev-image true
 sha256          e36b5ff86f35508539a8c8c8614ea227e67f97cf94830a8cee44fe0d2234dc1c
 description     bng-products
exit
software cnf cee
 url         http://192.168.107.148/images/CP/cp_30sep21/cee-2020.02.6.i04.tar
 sha256      b5040e9ad711ef743168bf382317a89e47138163765642c432eb5b80d7881f57
 description cee-products
exit
```
1. Setup Ops Center configs inside cluster for cnBNG
  ```
ops-centers bng bng
  repository-local        bng
  sync-default-repository true
  netconf-ip              192.168.107.166
  netconf-port            3022
  ssh-ip                  192.168.107.166
  ssh-port                2024
  ingress-hostname        192.168.107.166.nip.io
  initial-boot-parameters use-volume-claims true
  initial-boot-parameters first-boot-password <<your password>>
  initial-boot-parameters auto-deploy false
  initial-boot-parameters single-node true
 exit
 ops-centers cee global
  repository-local        cee
  sync-default-repository true
  netconf-ip              192.168.107.166
  netconf-port            3024
  ssh-ip                  192.168.107.166
  ssh-port                2023
  ingress-hostname        192.168.107.166.nip.io
  initial-boot-parameters use-volume-claims true
  initial-boot-parameters first-boot-password <<your password>>
  initial-boot-parameters auto-deploy true
  initial-boot-parameters single-node true
 exit
  ```
1. Deploy cnBNG CP cluster using cluster sync command:
  ```
clusters cnbng-cp-lab1 actions sync run debug true 
```	
Monitor sync using monitor command:
  ```
monitor sync-logs cnbng-cp-lab1
  ```
  


