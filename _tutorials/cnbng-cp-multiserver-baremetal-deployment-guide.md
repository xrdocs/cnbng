---
published: true
date: '2022-01-20 15:48 +0530'
title: cnBNG CP Multiserver (Baremetal) Deployment Guide
author: Gurpreet Dhaliwal
excerpt: >-
  Learn how to deploy cnBNG CP HA setup on baremetal UCS servers. Baremetal
  deployment is also called as CNDP deployment, it allows cnBNG CP to be
  deployed without need of running hypervisor. CNDP deployment offers 20%
  performance improvement over traditional hypervisor/VM based deployment.
position: top
---

{% include toc %}

In this tutorial we will learn how to deploy high capacity cnBNG CP on baremetal UCS Servers. Deployment of cnBNG CP on Baremetal Server is also known as CNDP (Cloud Native Development Platform) Deployment. The deployment of cnBNG CP is fully automated through Inception Deployer or Cluster Manager. 

We will deploy Multi server cnBNG CP in following steps:
	1. CNDP Preparation
	1. SSH Key Generation
	1. cnBNG CP Cluster Configuration
	1. cnBNG CP Deployment

## Logical Deployment Topology

![cndp-logical.png]({{site.baseurl}}/images/cndp-logical.png)

## Physical Deployment Topology

![cndp-physical.png]({{site.baseurl}}/images/cndp-physical.png)

## Prerequisites:

SMI Deployer Should be Preinstalled: We can either use Inception itself as SMI Deployer or deploy a separate cluster manager using Inception. Refer to Inception Server deployment guide or Cluster Manager Deployment Guide in tutorials.

***Note***: SMI Deployer requires access to CIMC for CNDP Cluster Deployment
{: .notice--info}

## Step 1: CNDP Preparation

- We have following hosts to deploy cnBNG CP. In this tutorial all UCS servers have same UCS access credentials. 

<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th></th>
    <th>CIMC IP</th>
  </tr>
  <tr>
    <td>Server-1</td>
    <td>10.81.103.30</td>
  </tr>
  <tr>
    <td>Server-2</td>
    <td>10.81.103.31</td>
  </tr>
  <tr>
    <td>Server-3</td>
    <td>10.81.103.32</td>
  </tr>
  <tr>
    <td>Server-4</td>
    <td>10.81.103.33</td>
  </tr>
</table>

- We will use following networks on each UCS server:

<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th>Network</th>
    <th>Bond I/f</th>
    <th>NIC/Port</th>
    <th>VLAN</th>
    <th>Host VLAN Interface</th>
     </tr>
  <tr>
    <td>Host Management</td>
    <td>bd0</td>
    <td>MLOM</td>
    <td>3103</td>
    <td>bd0.mgmt.3103</td>
  </tr>
  <tr>
    <td>k8s-api-net</td>
    <td>bd0</td>
    <td>MLOM</td>
    <td>308</td>
    <td>bd0.k8s.308</td>
  </tr>
  <tr>
    <td>svc-net1</td>
    <td>bd1</td>
    <td>PCIE0,PCIE1</td>
    <td>313</td>
    <td>bd1.sci.3103</td>
  </tr>
  <tr>
    <td>svc-net2</td>
    <td>bd1</td>
    <td>PCIE0,PCIE1</td>
    <td>314</td>
    <td>bd1.radius.3103</td>
  </tr>
</table>


- In this tutorial we will use a subnet for CIMC (10.81.103.0/24) and an Internal subnet(208.208.208.0/24) for k8s operations. We will then attach two service networks- svc-net1 and svc-net2 (213.213.213.0/24 and 214.214.214.0/24 respectively). svc-net1 will be used for cnBNG CP-UP communication whereas svc-net2 will be used for cnBNG CP-Radius communication. We will also create a management network for Host OS management in same network as CIMC (10.81.103.0/24).

<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th>UCS Server</th>
    <th>IP Address 1 (Host Management)</th>
    <th>IP Address 2 (k8s-api-net)</th>
    <th>IP Address 3 (svc-net1)</th>
    <th>IP Address 4 (svc-net2)</th>
     </tr>
  <tr>
    <td>Server-1</td>
    <td>10.81.103.59</td>
    <td>208.208.208.11</td>
    <td>213.213.213.11</td>
    <td>214.214.214.11</td>
  </tr>
  <tr>
    <td>Server-2</td>
    <td>10.81.103.60</td>
    <td>208.208.208.12</td>
    <td>NA</td>
    <td>NA</td>
  </tr>
  <tr>
    <td>Server-3</td>
    <td>10.81.103.61</td>
    <td>208.208.208.13</td>
    <td>NA</td>
    <td>NA</td>
  </tr>
  <tr>
    <td>Server-1</td>
    <td>10.81.103.62</td>
    <td>208.208.208.14</td>
    <td>213.213.213.12</td>
    <td>214.214.214.12</td>
  </tr>
</table>

- We would also need Virtual IPs which will be used for VRRP between similar kind nodes to provide HA:

<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th>VIP Type/Type</th>
    <th>IP Address</th>
    <th>Remarks</th>
  </tr>
  <tr>
    <td>Master VIP1</td>
    <td>10.81.103.113</td>
    <td>Used for management access: k8s-master, ops-center, grafana etc.</td>
  </tr>
  <tr>
    <td>Master VIP2</td>
    <td>208.208.208.101</td>
    <td>k8s-api-net access</td>
  </tr>
  <tr>
    <td>Protocol VIP1</td>
    <td>208.208.208.51</td>
    <td>k8s-api-net vip for protocol labelled servers</td>
  </tr>
  <tr>
    <td>Protocol VIP2</td>
    <td>213.213.213.51</td>
    <td>svc-net-1 VIP, will be used for peering with cnBNG UP</td>
  </tr>
 <tr>
    <td>Proto VIP3</td>
    <td>214.214.214.51</td>
    <td>svc-net-2 VIP, will be used for Radius communication</td>
  </tr>
</table>

## Step 2: SSH Key Generation

This is for security reasons, where cnBNG CP VM can only be accessed by ssh key and not through password by default.

1. SSH Login to Inception VM
1. Generate SSH key using: 
    ```ssh-keygen -t rsa```
1. Note down ssh keys in a file from .ssh/id_rsa (private) and ./ssh/id_rsa.pub (public)
1. Remove line breaks from private key and replace them with string "\n"
1. Remove line breaks from public key if there are any.

## Step 3: Cluster Configuration

-  We will first define the software repos for cnBNG CP and CEE. cnBNG software is available as a tarball and it can be hosted on local http server for offline deployment. In this step we configure the software repository locations for tarball. We setup software cnf for both cnBNG CP and CEE. URL and SHA256 depends on the version of the image and the url location, so these two could change for your deployment.

```
software cnf bng
 url             http://192.168.107.148/images/CP/cp_30sep21/bng/bng.2021.04.m0.i74.tar
 sha256          e36b5ff86f35508539a8c8c8614ea227e67f97cf94830a8cee44fe0d2234dc1c
 description     bng-products
exit
software cnf cee
 url         http://192.168.107.148/images/CP/cp_30sep21/cee-2020.02.6.i04.tar
 sha256      b5040e9ad711ef743168bf382317a89e47138163765642c432eb5b80d7881f57
 description cee-products
exit
```

- Next we define two host profiles, host profile is used to for BIOS settings on UCS e.g. Hyperthreading Enable. We can host these profiles at the same location on HTTP server as software cnf tarballs in previous step.

```
software host-profile bng-ht-enable
 url    http://10.81.103.105/cnbng-cndp/ht.tgz
 sha256 aa7e240f2b785a8c8d6b7cd6f79fe162584dc01b7e9d32a068be7f6e5055f664
exit
software host-profile bng-ht-sysctl-enable
 url    http://10.81.103.105/cnbng-cndp/ht_sysctl.tgz
 sha256 cf468ac05de073fa51a5811a5eeb517fe003a55c394e5382686a573a472bf998
exit
```

- Next we define environment and Cluster configuration based on our deployment schemel. The configuration is self explanatory. However do note two kinds of VIPs define here for master. 

```
environments bare-metal
 ucs-server
exit
feature-gates test true
clusters cnbng-cp-cluster1
 environment bare-metal
 addons ingress bind-ip-address 10.81.103.113
 addons ingress bind-ip-address-internal 208.208.208.101
 addons ingress enabled
 addons kubernetes-dashboard enabled
 addons istio enabled
 addons cpu-partitioner enabled
 addons cpu-partitioner tier small
 configuration master-virtual-ip        208.208.208.101
 configuration master-virtual-ip-cidr   24
 configuration master-virtual-ip-interface bd0.k8s.308
 configuration additional-master-virtual-ip 10.81.103.113
 configuration additional-master-virtual-ip-cidr 24
 configuration additional-master-virtual-ip-interface bd0.mgmt.3103
 configuration virtual-ip-vrrp-router-id 60
 configuration enable-pod-security-policy false
 configuration pod-subnet               192.208.0.0/16
 configuration allow-insecure-registry  true
 configuration restrict-logging         false
 node-defaults ssh-username username1
 node-defaults k8s ssh-connection-private-key "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAwx+LkHa8U74BYN9rM3MS1H8K8ZiOdd8kqA5I8553KH75C/yV\nddtiQLBh7y+k9/YwUSsrbj6S4b6/Wb8CpTmeQXnlAfmY/LbmXUEaiKb6J4bXBEnN\ndaETOsOXUIx1CGg3Sv7wUEIOYUkNAlc71VNg0vBPnLDGwaD82lFYvqj46nwRLfTM\nzMrHNQ551zdXxTA1c0wLG+ZoLf3C43H26OqlYC7fYqxSfwH6EBpbUIvTOL3obKa2\nGZ4Bi+VPfqrfap+JW2sU7dh6jUCQLpO9NS1YWTVygnx5oIOnp9yGJO2GSNSIYQJ2\nAKpUC7gu3X/Q6hOotIgfHk7kYeSNirD1KwixrwIDAQABAoIBADKgQKne5MYlil4E\nGeBjfwM7Yy+EEZJrrysbabor52bOave9NVo67aczHHXeusLLUYX92WrlOV7xCtzS\nPnF4HaOHaO+2PwdyvRp9BdFm4YjX53npXDGk9URN8zim+MaRo6cFtnxcZza+qW1u\nDMwwsfKI/178TtV2W6SZbpkpZkwQJ/cN1fljxuQYubOujMZErc1Q7TlIQYrG8XVz\naOTVkYBU+O2DOKkWp4AM9qe/0RIdjmz1ZZHRB83iF3/OaU3j2mhlhRQwTiS3jcGX\nkWoovz/GmFAcw+VSz81tz2UFmGNczkl8F5t4afYq31M7r+restSd6Mc7dcqWA7pR\nYflz7SECgYEA+qleQDAxe9cSRyGk0aCXV7NKQnvmIReSiRxibNne2FTnLaBH7iM7\nGe8XT23gFLCvM1qmpaLxjQZEP4A7WTaZuTEQAvVHYM+4m+1XDz4+ePaSCFwru8Q/\nLvE0h45JGfuHiXIWQNsyycSnDa+50BQ1a+EruzsmdHDvVnhO/QK0gzsCgYEAx0dg\nVbOjGzEvuGTfTI8qqRYz2XQjSKr6IHqRGY/h2hVcJo0Is+fr9n2QlebJ9oIOVnw7\npSlPgnMNcI9b74aCGuVnVUW/23TRkTvq6goNBa2V++d9EHPNo2gahc1tuJZzkp8I\nnZONBHzU8jwe9nBbK7uuz9i8cgSnMaWbS6Q8PB0CgYEAoXzoWdYyqyQ+hFEqjFs3\n5ap+lyKXeo5jO65rwtECfsEERyLR9JwCAY1FqUiSawIBfcZTQrcdg8ubwIVutuU0\nWFlBhYZcPATXXK2lvw5M1UWVg4lOK6QdSLLhMsv6UKD6CxTTPWl66P6m2Wxy+5lp\naV0h/Xf4KGBx8XWE/f/2J+0CgYEAlfJGMZZut4pGLwhv4WqkngBf2VMDLa3Bcejn\n/4T9W5zQ7w0WLFDpg1quDa1P8JWh9j+ancc81Zp+1WB5u/zJLzXIkChgmeAHxLGC\nLMKNU+VuwtJHj7ajWD6AHogZ9Ff49K2HzRH2fRb1IKROY/7dC0Y43ppmCaEosTm8\nZalZzZ0CgYAF6KnlPw1qbpVZYyVcPw38omeisMs/92ovf4oqzJpZ5zCW1iKKRpQR\nq4oDLca0zJS/AUmXFTIG//VHvsna4A2JrmNS4rI+bEgwSDrRKHmX8Idctdu1ifgS\nrRtv1s5RfTZ67eWVYE6rUn1+EFBZwAzZorOXL3CzbjZEXUCJeOH6Lw==\n-----END RSA PRIVATE KEY-----"
 node-defaults initial-boot default-user username1
 node-defaults initial-boot default-user-ssh-public-key "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDDH4uQdrxTvgFg32szcxLUfwrxmI513ySoDkjznncofvkL/JV122JAsGHvL6T39jBRKytuPpLhvr9ZvwKlOZ5BeeUB+Zj8tuZdQRqIpvonhtcESc11oRM6w5dQjHUIaDdK/vBQQg5hSQ0CVzvVU2DS8E+csMbBoPzaUVi+qPjqfBEt9MzMysc1DnnXN1fFMDVzTAsb5mgt/cLjcfbo6qVgLt9irFJ/AfoQGltQi9M4vehsprYZngGL5U9+qt9qn4lbaxTt2HqNQJAuk701LVhZNXKCfHmgg6en3IYk7YZI1IhhAnYAqlQLuC7df9DqE6i0iB8eTuRh5I2KsPUrCLGv cloud-user@inception"
 node-defaults initial-boot default-user-password password1
```

- Define netplan to create bond interfaces and VLANs

```
node-defaults initial-boot netplan ethernets eno1
  dhcp4 false
  dhcp6 false
 exit
 node-defaults initial-boot netplan ethernets eno2
  dhcp4 false
  dhcp6 false
 exit
!! eno5 and eno6 are MLOM ports
 node-defaults initial-boot netplan ethernets eno5
  dhcp4 false
  dhcp6 false
 exit
 node-defaults initial-boot netplan ethernets eno6
  dhcp4 false
  dhcp6 false
 exit
!! enp216s0f1 and enp94s0f0 are PCIE0 and PCIE1 ports respectively 
 node-defaults initial-boot netplan ethernets enp216s0f0
  dhcp4 false
  dhcp6 false
 exit
 node-defaults initial-boot netplan ethernets enp216s0f1
  dhcp4 false
  dhcp6 false
 exit
 node-defaults initial-boot netplan ethernets enp94s0f0
  dhcp4 false
  dhcp6 false
 exit
 node-defaults initial-boot netplan ethernets enp94s0f1
  dhcp4 false
  dhcp6 false
 exit
 node-defaults initial-boot netplan bonds bd0
  dhcp4      false
  dhcp6      false
  optional   true
  interfaces [ eno5 eno6 ]   
  parameters mode      active-backup
  parameters mii-monitor-interval 100
  parameters fail-over-mac-policy active
 exit
 node-defaults initial-boot netplan bonds bd1
  dhcp4      false
  dhcp6      false
  optional   true
  interfaces [ enp216s0f1 enp94s0f0 ]  
  parameters mode      active-backup
  parameters mii-monitor-interval 100
  parameters fail-over-mac-policy active
 exit
!! VLANs are defined here, change it as per your environment
 node-defaults initial-boot netplan vlans bd0.k8s.308
  dhcp4 false
  dhcp6 false
  id    308
  link  bd0
 exit
```

- Next define the CIMC access details and parameters. Make sure NTP server is reachable from all servers and resides in common management network.

```
node-defaults ucs-server cimc user admin
 node-defaults ucs-server cimc password CIMC_Password
 node-defaults ucs-server cimc storage-adaptor create-virtual-drive true
 node-defaults ucs-server cimc remote-management sol enabled
 node-defaults ucs-server cimc remote-management sol baud-rate 115200
 node-defaults ucs-server cimc remote-management sol comport com0
 node-defaults ucs-server cimc remote-management sol ssh-port 2400
 node-defaults ucs-server cimc networking ntp enabled
 node-defaults ucs-server cimc networking ntp servers 72.163.32.44
exit
```
 
 - We now define NTP and node defaults
 
 ```
 node-defaults os ntp enabled
 node-defaults os ntp servers clock.cisco.com
 exit
 node-defaults os ntp servers ntp.esl.cisco.com
 exit
 node-type-defaults master
  initial-boot netplan vlans bd0.mgmt.3103
   nameservers search [ cisco.com  ]
   nameservers addresses [ 64.102.6.247 ]
   id   3103
   link bd0
  exit
 exit
 node-type-defaults worker
  initial-boot netplan vlans bd0.mgmt.3103
   nameservers search [ cisco.com ]
   nameservers addresses [ 64.102.6.247 ]
   id   3103
   link bd0
  exit
 exit
```

- Let’s now define the nodes for cluster creation. The definition is as per the plan we created above in Step-1.

```
nodes server-1
  host-profile bng-ht-sysctl-enable
  k8s node-type master
  k8s ssh-ip 208.208.208.11
  k8s node-ip 208.208.208.11
  k8s node-labels smi.cisco.com/node-type oam 
  exit
  k8s node-labels smi.cisco.com/proto-type protocol
  exit
  ucs-server cimc ip-address 10.81.103.30
  initial-boot netplan vlans bd0.k8s.308
   addresses [ 208.208.208.11/24 ]
  exit
  initial-boot netplan vlans bd0.mgmt.3103
   addresses [ 10.81.103.59/24 ]
   gateway4  10.81.103.1
  exit
  initial-boot netplan vlans bd1.n4.313
   dhcp4     false
   dhcp6     false
   addresses [ 213.213.213.11/24 ]
   id        313
   link      bd1
  exit
  initial-boot netplan vlans bd1.radius.314
   dhcp4     false
   dhcp6     false
   addresses [ 214.214.214.11/24 ]
   id        314
   link      bd1
  exit
  os tuned enabled
 exit
 nodes server-2
  host-profile bng-ht-enable
  k8s node-type master
  k8s ssh-ip 208.208.208.12
  k8s node-ip 208.208.208.12
  k8s node-labels smi.cisco.com/node-type oam
  exit
  k8s node-labels smi.cisco.com/sess-type cdl-node
  exit
  k8s node-labels smi.cisco.com/svc-type service
  exit
  ucs-server cimc ip-address 10.81.103.31
  initial-boot netplan vlans bd0.k8s.308
   addresses [ 208.208.208.12/24 ]
  exit
  initial-boot netplan vlans bd0.mgmt.3103
   addresses [ 10.81.103.60/24 ]
   gateway4  10.81.103.1
  exit
  os tuned enabled
 exit
 nodes server-3
  host-profile bng-ht-enable
  k8s node-type master
  k8s ssh-ip 208.208.208.13
  k8s node-ip 208.208.208.13
  k8s node-labels smi.cisco.com/node-type oam
  exit
  k8s node-labels smi.cisco.com/sess-type cdl-node
  exit
  k8s node-labels smi.cisco.com/svc-type service
  exit
  ucs-server cimc ip-address 10.81.103.32
  initial-boot netplan vlans bd0.k8s.308
   addresses [ 208.208.208.13/24 ]
  exit
  initial-boot netplan vlans bd0.mgmt.3103
   addresses [ 10.81.103.61/24 ]
   gateway4  10.81.103.1
  exit
  os tuned enabled
 exit
 nodes server-4
  host-profile bng-ht-sysctl-enable
  k8s node-type worker
  k8s ssh-ip 208.208.208.14
  k8s node-ip 208.208.208.14
  k8s node-labels smi.cisco.com/proto-type protocol
  exit
  ucs-server cimc ip-address 10.81.103.33
  initial-boot netplan vlans bd0.k8s.308
   addresses [ 208.208.208.14/24 ]
  exit
  initial-boot netplan vlans bd0.mgmt.3103
   addresses [ 10.81.103.62/24 ]
   gateway4  10.81.103.1
  exit
  initial-boot netplan vlans bd1.n4.313
   dhcp4     false
   dhcp6     false
   addresses [ 213.213.213.12/24 ]
   id        313
   link      bd1
  exit
  initial-boot netplan vlans bd1.radius.314
   dhcp4     false
   dhcp6     false
   addresses [ 214.214.214.12/24 ]
   id        314
   link      bd1
  exit
  os tuned enabled
 exit
```

-  We now define the virtual IPs 

```
virtual-ips udpvip
  check-ports    [ 28000 ]
  vrrp-interface bd1.n4.313
  vrrp-router-id 222
  check-interface bd0.k8s.308
  exit
  check-interface bd1.n4.313
  exit
  check-interface bd1.radius.314
  exit
  ipv4-addresses 208.208.208.51
   mask      24
   broadcast 208.208.208.255
   device    bd0.k8s.308
  exit
  ipv4-addresses 213.213.213.51
   mask      24
   broadcast 213.213.213.255
   device    bd1.n4.313
  exit
  ipv4-addresses 214.214.214.51
   mask      24
   broadcast 214.214.214.255
   device    bd1.radius.314
  exit
  hosts server-1
   priority 100
  exit
  hosts server-4
   priority 100
  exit
 exit
```

- We now define the Ops Center configurations

```
clusters cnbng-cp-cluster1
  ops-centers bng bng
    repository-local        bng
    sync-default-repository true
    netconf-ip              10.81.103.113
    netconf-port            2022
    ssh-ip                  10.81.103.113
    ssh-port                2024
    ingress-hostname        10.81.103.113.nip.io
    initial-boot-parameters use-volume-claims true
    initial-boot-parameters first-boot-password password1
    initial-boot-parameters auto-deploy false
    initial-boot-parameters single-node true
  exit
  ops-centers cee global
  	repository-local        cee
    sync-default-repository true
    netconf-ip              10.81.103.113
    netconf-port            3024
    ssh-ip                  10.81.103.113
    ssh-port                3023
    ingress-hostname        10.81.103.113.nip.io
    initial-boot-parameters use-volume-claims true
    initial-boot-parameters first-boot-password password1
    initial-boot-parameters auto-deploy true
    initial-boot-parameters single-node true
  exit
exit
```

## Step 4: cnBNG CP Deployment

- After committing configuration which we build in Step-4 on SMI Deployer. We can start the cluster sync.

```
[inception] SMI Cluster Deployer# clusters cnbng-cp-cluster1 actions sync run   
This will run sync.  Are you sure? [no,yes] yes
message accepted
[inception] SMI Cluster Deployer# 
```
- Monitor sync using below command. It will take about 90mins to sync and deploy the cluster. 

```
monitor sync-logs cnbng-cp-cluster1

E.g.

[inception] SMI Cluster Deployer# monitor sync-logs cnbng-cp-cluster1
. . .
. . .

Thursday 18 November 2021  14:38:49 +0000 (0:00:00.064)       0:08:42.748 ***** 
=============================================================================== 
2021-11-18 14:38:49.716 DEBUG cluster_sync.cnbng-cp-cluster1: Cluster sync successful 
2021-11-18 14:38:49.717 DEBUG cluster_sync.cnbng-cp-cluster1: Ansible sync done 
2021-11-18 14:38:49.717 INFO cluster_sync.cnbng-cp-cluster1: _sync finished.  Opening lock 
```

## Verifications

After the cluster is deployed it can be verified using below commands

- Connect to K8s Master node using SSH with Master VIP1 IP

```
ssh cisco@10.81.103.113
```

- After connecting verify that all PODs are in Running state

```
kubectl get pods -A
```

- Verify nodes in cnBNG Cluster using

```
kubectl get nodes

e.g.
cisco@cnbng-cp-cluster1-server-1:~$ kubectl get nodes
NAME                          STATUS   ROLES                  AGE   VERSION
cnbng-cp-cluster1-server-1   Ready    control-plane,master   11d   v1.21.0
cnbng-cp-cluster1-server-2   Ready    control-plane,master   11d   v1.21.0
cnbng-cp-cluster1-server-3   Ready    control-plane,master   11d   v1.21.0
cnbng-cp-cluster1-server-4   Ready    <none>                 11d   v1.21.0
```

- We can connect to Grafana at the grafana ingress

```
kubectl get ingress -A

e.g.
cloud-user@svi-cnbng-cndp-tb4-server-1:~$ kubectl get ingress -A
NAMESPACE     NAME                                       CLASS    HOSTS                                                        ADDRESS                                        PORTS     AGE
bng-bng       cli-ingress-bng-bng-ops-center             <none>   cli.bng-bng-ops-center.10.81.103.113.nip.io                  208.208.208.11,208.208.208.12,208.208.208.13   80, 443   3d22h
bng-bng       documentation-ingress                      <none>   documentation.bng-bng-ops-center.10.81.103.113.nip.io        208.208.208.11,208.208.208.12,208.208.208.13   80, 443   3d22h
bng-bng       oam-files-ingress-bng-bng-oam-pod          <none>   oam-files.bng-bng-oam-pod.10.81.103.113.nip.io               208.208.208.11,208.208.208.12,208.208.208.13   80, 443   2d21h
bng-bng       restconf-ingress-bng-bng-ops-center        <none>   restconf.bng-bng-ops-center.10.81.103.113.nip.io             208.208.208.11,208.208.208.12,208.208.208.13   80, 443   3d22h
cee-global    cee-global-product-documentation-ingress   <none>   docs.cee-global-product-documentation.10.81.103.113.nip.io   208.208.208.11,208.208.208.12,208.208.208.13   80, 443   33d
cee-global    cli-ingress-cee-global-ops-center          <none>   cli.cee-global-ops-center.10.81.103.113.nip.io               208.208.208.11,208.208.208.12,208.208.208.13   80, 443   33d
cee-global    documentation-ingress                      <none>   documentation.cee-global-ops-center.10.81.103.113.nip.io     208.208.208.11,208.208.208.12,208.208.208.13   80, 443   33d
cee-global    grafana-ingress                            <none>   grafana.10.81.103.113.nip.io                                 208.208.208.11,208.208.208.12,208.208.208.13   80, 443   33d
cee-global    prometheus-hi-res                          <none>   prometheus-hi-res.10.81.103.113.nip.io                       208.208.208.11,208.208.208.12,208.208.208.13   80, 443   33d
cee-global    restconf-ingress-cee-global-ops-center     <none>   restconf.cee-global-ops-center.10.81.103.113.nip.io          208.208.208.11,208.208.208.12,208.208.208.13   80, 443   33d
cee-global    show-tac-manager-ingress                   <none>   show-tac-manager.10.81.103.113.nip.io                        208.208.208.11,208.208.208.12,208.208.208.13   80, 443   33d
kube-system   kubernetes-dashboard                       <none>   kubernetes-dashboard.10.81.103.113.nip.io                    208.208.208.11,208.208.208.12,208.208.208.13   80, 443   33d
registry      charts-ingress                             <none>   charts.208.208.208.101.nip.io                                208.208.208.11,208.208.208.12,208.208.208.13   80, 443   33d
registry      registry-ingress                           <none>   docker.208.208.208.101.nip.io                                208.208.208.11,208.208.208.12,208.208.208.13   80, 443   33d
```

- SSH to cnBNG CP Ops Center at port 2024
  
```
cloud-user@inception:~$ ssh admin@10.81.103.113 -p 2024
admin@10.81.103.113's password: 

      Welcome to the bng CLI on pod2/bng
      Copyright © 2016-2020, Cisco Systems, Inc.
      All rights reserved.
    
User admin last logged in 2021-11-19T04:12:32.912093+00:00, to ops-center-bng-bng-ops-center-77fb6479fc-dtvt2, from 10.81.103.113 using cli-ssh
admin connected from 10.81.103.113 using ssh on ops-center-bng-bng-ops-center-77fb6479fc-dtvt2
[pod1/bng] bng# 
[pod1/bng] bng# 
```
  
- We can also test Netconf Interface availability of cnBNG Ops Center using ssh
  
```
cloud-user@inception:~$ ssh admin@10.81.103.113 -p 3022 -s netconf    
Warning: Permanently added '[10.81.103.113]:3022' (RSA) to the list of known hosts.
admin@10.81.103.101's password: 
<?xml version="1.0" encoding="UTF-8"?>
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<capabilities>
<capability>urn:ietf:params:netconf:base:1.0</capability>
<capability>urn:ietf:params:netconf:base:1.1</capability>
<capability>urn:ietf:params:netconf:capability:confirmed-commit:1.1</capability>
<capability>urn:ietf:params:netconf:capability:confirmed-commit:1.0</capability>
<capability>urn:ietf:params:netconf:capability:candidate:1.0</capability>
<capability>urn:ietf:params:netconf:capability:rollback-on-error:1.0</capability>
<capability>urn:ietf:params:netconf:capability:url:1.0?scheme=ftp,sftp,file</capability>
<capability>urn:ietf:params:netconf:capability:validate:1.0</capability>
<capability>urn:ietf:params:netconf:capability:validate:1.1</capability>
<capability>urn:ietf:params:netconf:capability:xpath:1.0</capability>
<capability>urn:ietf:params:netconf:capability:notification:1.0</capability>
<capability>urn:ietf:params:netconf:capability:interleave:1.0</capability>
<capability>urn:ietf:params:netconf:capability:partial-lock:1.0</capability>
<capability>urn:ietf:params:netconf:capability:with-defaults:1.0?basic-mode=explicit&amp;also-supported=report-all-tagged,report-all</capability>
<capability>urn:ietf:params:netconf:capability:yang-library:1.0?revision=2019-01-04&amp;module-set-id=a6803bdbd5c766b47137ad86700fff0a</capability>
<capability>urn:ietf:params:netconf:capability:yang-library:1.1?revision=2019-01-04&amp;content-id=a6803bdbd5c766b47137ad86700fff0a</capability>
<capability>http://tail-f.com/ns/netconf/actions/1.0</capability>
<capability>http://cisco.com/cisco-bng-ipam?module=cisco-bng-ipam&amp;revision=2020-01-24</capability>
<capability>http://cisco.com/cisco-cn-ipam?module=cisco-cn-ipam&amp;revision=2021-02-21</capability>
<capability>http://cisco.com/cisco-exec-ipam?module=cisco-exec-ipam&amp;revision=2021-06-01</capability>
<capability>http://cisco.com/cisco-mobile-nf-tls?module=cisco-mobile-nf-tls&amp;revision=2020-06-24</capability>
<capability>http://cisco.com/cisco-smi-etcd?module=cisco-smi-etcd&amp;revision=2021-09-15</capability>
<capability>http://tail-f.com/cisco-mobile-common?module=tailf-mobile-common&amp;revision=2019-04-25</capability>
<capability>http://tail-f.com/cisco-mobile-product?module=tailf-cisco-mobile-product&amp;revision=2018-06-06</capability>
<capability>http://tail-f.com/ns/aaa/1.1?module=tailf-aaa&amp;revision=2018-09-12</capability>
<capability>http://tail-f.com/ns/common/query?module=tailf-common-query&amp;revision=2017-12-15</capability>
<capability>http://tail-f.com/ns/confd-progress?module=tailf-confd-progress&amp;revision=2020-06-29</capability>
<capability>http://tail-f.com/ns/kicker?module=tailf-kicker&amp;revision=2020-11-26</capability>
<capability>http://tail-f.com/ns/netconf/query?module=tailf-netconf-query&amp;revision=2017-01-06</capability>
<capability>http://tail-f.com/ns/webui?module=tailf-webui&amp;revision=2013-03-07</capability>
<capability>http://tail-f.com/yang/acm?module=tailf-acm&amp;revision=2013-03-07</capability>
<capability>http://tail-f.com/yang/common?module=tailf-common&amp;revision=2020-11-26</capability>
<capability>http://tail-f.com/yang/common-monitoring?module=tailf-common-monitoring&amp;revision=2019-04-09</capability>
<capability>http://tail-f.com/yang/confd-monitoring?module=tailf-confd-monitoring&amp;revision=2019-10-30</capability>
<capability>http://tail-f.com/yang/last-login?module=tailf-last-login&amp;revision=2019-11-21</capability>
<capability>http://tail-f.com/yang/netconf-monitoring?module=tailf-netconf-monitoring&amp;revision=2019-03-28</capability>
<capability>http://tail-f.com/yang/xsd-types?module=tailf-xsd-types&amp;revision=2017-11-20</capability>
<capability>urn:ietf:params:xml:ns:netconf:base:1.0?module=ietf-netconf&amp;revision=2011-06-01&amp;features=confirmed-commit,candidate,rollback-on-error,validate,xpath,url</capability>
<capability>urn:ietf:params:xml:ns:netconf:partial-lock:1.0?module=ietf-netconf-partial-lock&amp;revision=2009-10-19</capability>
<capability>urn:ietf:params:xml:ns:yang:iana-crypt-hash?module=iana-crypt-hash&amp;revision=2014-08-06&amp;features=crypt-hash-sha-512,crypt-hash-sha-256,crypt-hash-md5</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-inet-types?module=ietf-inet-types&amp;revision=2013-07-15</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-netconf-acm?module=ietf-netconf-acm&amp;revision=2018-02-14</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring?module=ietf-netconf-monitoring&amp;revision=2010-10-04</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-netconf-notifications?module=ietf-netconf-notifications&amp;revision=2012-02-06</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-netconf-with-defaults?module=ietf-netconf-with-defaults&amp;revision=2011-06-01</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-restconf-monitoring?module=ietf-restconf-monitoring&amp;revision=2017-01-26</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-x509-cert-to-name?module=ietf-x509-cert-to-name&amp;revision=2014-12-10</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-yang-metadata?module=ietf-yang-metadata&amp;revision=2016-08-05</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-yang-types?module=ietf-yang-types&amp;revision=2013-07-15</capability>
</capabilities>
<session-id>171</session-id></hello>]]>]]>
```


***Credits***: Special thanks to Sripada Rao for providing configurations from SVI.
