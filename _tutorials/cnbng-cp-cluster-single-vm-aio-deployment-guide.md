---
published: false
date: '2021-12-06 18:19 +0530'
title: cnBNG CP Cluster Single VM (AIO) Deployment Guide
---
## Introduction
AIO VMware Deployment means All-In-One deployment of cnBNG Control Plane in VMWare ESXi environment. This is a single VM deployment for cnBNG Control Plane in VMWare environment which deploys following in a single customized base Ubuntu VM:

- SMI cluster: Cloud Native Infra including K8s, docker, keepalived, calico, istio etc.
- CEE application: Common Execution Environment for telemetry, snmp, bulkstats, alert etc.
- cnBNG Control Plane application: Core BNG Control Plane application

The deployment is done in 3 steps, where the 3rd step uses automation provided by Inception Server to deploy cnBNG CP in All-In-One form. The main steps for deployment are:

1. Inception Server/SMI Cluster Deployer deployment using vCenter
1. SSH Key Generation for security purposes
1. cnBNG CP Deployment using Inception Server, which includes:
	- Customized Base OS Ubuntu VM Deployment (automated for ESXi)
	- SMI Cluster Deployment, which includes CN Infra (automated)
	- CEE and cnBNG CP Application deployment (automated)

## Prerequisite (optional, if DC already exists)
1. Make sure vCenter is installed (ver 6.7 is tested)
1. In the vCenter, right click and select New Data Center
	- Provide name of the data-center and click ok
1. Right click on the newly created datacenter and select New Cluster.
	- Provide name of the cluster and click ok (all other options remain default)
![vmware-dc1.png]({{site.baseurl}}/images/vmware-dc1.png)
1. Now add a host to the cluster. By selecting add host from right click menu on newly created cluster. Follow on screen instructions to add the host.

## Step 1: Inception CM Deployment
Refer to [Inception Server Deployment Guide](https://xrdocs.io/cnbng/tutorials/inception-server-deployment-guide/).

## Step 2: SSH Key Generation
This is for security reasons, where cnBNG CP VM can only be accessed by ssh key and not through password by default.

- SSH Login to Inception VM
- Generate SSH key using:

```
ssh-keygen -t rsa
```
- Note down ssh keys in a file from .ssh/id_rsa (private) and ./ssh/id_rsa.pub (public)
- Remove line breaks from private key and replace them with string "\n"
- Remove line breaks from public key if there are any.

## Step 3: cnBNG CP deployment using Inception CM
- Login to SMI Cluster Deployer or Cluster Manager on inception VM:

```
cloud-user@inception:~$ ssh admin@localhost -p 2022
admin@localhost's password: 

      Welcome to the Cisco SMI Cluster Deployer on inception
      Copyright © 2016-2020, Cisco Systems, Inc.
      All rights reserved.
    
admin connected from 172.18.0.1 using ssh on 79bd273aeec7
[inception] SMI Cluster Deployer# 
[inception] SMI Cluster Deployer#
```

- In the next few steps we will be setting up configuration in SMI Cluster Deployer. We will be using config mode and after adding all the configs we will be using commit.

```
[inception] SMI Cluster Deployer# config
Entering configuration mode terminal
[inception] SMI Cluster Deployer(config)# <<your configs to apply>>
[inception] SMI Cluster Deployer(config)# commit
[inception] SMI Cluster Deployer(config)# end
```

- Create environment configs. This config is for inception deployer to get access to vCenter.

```
environments vmware
 vcenter server         <<vcenter ip>>
 vcenter allow-self-signed-cert true
 vcenter user           <<your vcenter username>>
 vcenter password       <<your password>>
 vcenter datastore      <<server local datastore>>
 vcenter cluster        cnBNG_SMI_CL01
 vcenter datacenter     cnBNG_DC_01
 vcenter host           <<your host name>>
 vcenter nics "VM Network"
 exit
exit
```

- Define Netplan for cnBNG CP Ubuntu VM:

```
 network:
    version: 2
    ethernets:
        ens192:
            dhcp4: false
            addresses:
            - {{K8S_SSH_IP}}/24
            routes:
            - to: 0.0.0.0/0
              via: <<gateway>>
            nameservers:
                addresses:
                - <<dns1>>
                - <<dns2>>
                search:
                - <<domain>>
```

- Replace line breaks by "\n". Netplan will look like this:

```
 "\n     network:\n        version: 2\n        ethernets:\n            ens192:\n                dhcp4: false\n                addresses:\n                - {{K8S_SSH_IP}}/24\n                routes:\n                - to: 0.0.0.0/0\n                  via: <<gateway>>\n                nameservers:\n                    addresses:\n                    - <<dns1>>\n                    - <<dns2>>\n                    search:\n                    - <<domain>>\n\n\n"
```

- Create Cluster configs with SSH public and private keys copied in Step-2 and netplan from previous step:

```
clusters <<your cnBNG CP Cluster Name>>
 environment vmware
 addons ingress bind-ip-address <<your cnBNG CP VM IP>>
 addons ingress enabled
 addons istio enabled
 configuration master-virtual-ip <<your cnBNG CP VM IP>>
 configuration master-virtual-ip-interface ens192
 configuration pod-subnet    192.202.0.0/16
 configuration allow-insecure-registry true
 node-defaults initial-boot default-user <<your cnBNG CP VM user name>>
 node-defaults initial-boot default-user-ssh-public-key "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDDH4uQdrxTvgFg32szcxLUfwrxmI513ySoDkjznncofvkL/JV122JAsGHvL6T39jBRKytuPpLhvr9ZvwKlOZ5BeeUB+Zj8tuZdQRqIpvonhtcESc11oRM6w5dQjHUIaDdK/vBQQg5hSQ0CVzvVU2DS8E+csMbBoPzaUVi+qPjqfBEt9MzMysc1DnnXN1fFMDVzTAsb5mgt/cLjcfbo6qVgLt9irFJ/AfoQGltQi9M4vehsprYZngGL5U9+qt9qn4lbaxTt2HqNQJAuk701LVhZNXKCfHmgg6en3IYk7YZI1IhhAnYAqlQLuC7df9DqE6i0iB8eTuRh5I2KsPUrCLGv cloud-user@inception"
 node-defaults k8s ssh-username <<your cnBNG CP VM username>>
 node-defaults k8s ssh-connection-private-key "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAwx+LkHa8U74BYN9rM3MS1H8K8ZiOdd8kqA5I8553KH75C/yV\nddtiQLBh7y+k9/YwUSsrbj6S4b6/Wb8CpTmeQXnlAfmY/LbmXUEaiKb6J4bXBEnN\ndaETOsOXUIx1CGg3Sv7wUEIOYUkNAlc71VNg0vBPnLDGwaD82lFYvqj46nwRLfTM\nzMrHNQ551zdXxTA1c0wLG+ZoLf3C43H26OqlYC7fYqxSfwH6EBpbUIvTOL3obKa2\nGZ4Bi+VPfqrfap+JW2sU7dh6jUCQLpO9NS1YWTVygnx5oIOnp9yGJO2GSNSIYQJ2\nAKpUC7gu3X/Q6hOotIgfHk7kYeSNirD1KwixrwIDAQABAoIBADKgQKne5MYlil4E\nGeBjfwM7Yy+EEZJrrysbabor52bOave9NVo67aczHHXeusLLUYX92WrlOV7xCtzS\nPnF4HaOHaO+2PwdyvRp9BdFm4YjX53npXDGk9URN8zim+MaRo6cFtnxcZza+qW1u\nDMwwsfKI/178TtV2W6SZbpkpZkwQJ/cN1fljxuQYubOujMZErc1Q7TlIQYrG8XVz\naOTVkYBU+O2DOKkWp4AM9qe/0RIdjmz1ZZHRB83iF3/OaU3j2mhlhRQwTiS3jcGX\nkWoovz/GmFAcw+VSz81tz2UFmGNczkl8F5t4afYq31M7r+restSd6Mc7dcqWA7pR\nYflz7SECgYEA+qleQDAxe9cSRyGk0aCXV7NKQnvmIReSiRxibNne2FTnLaBH7iM7\nGe8XT23gFLCvM1qmpaLxjQZEP4A7WTaZuTEQAvVHYM+4m+1XDz4+ePaSCFwru8Q/\nLvE0h45JGfuHiXIWQNsyycSnDa+50BQ1a+EruzsmdHDvVnhO/QK0gzsCgYEAx0dg\nVbOjGzEvuGTfTI8qqRYz2XQjSKr6IHqRGY/h2hVcJo0Is+fr9n2QlebJ9oIOVnw7\npSlPgnMNcI9b74aCGuVnVUW/23TRkTvq6goNBa2V++d9EHPNo2gahc1tuJZzkp8I\nnZONBHzU8jwe9nBbK7uuz9i8cgSnMaWbS6Q8PB0CgYEAoXzoWdYyqyQ+hFEqjFs3\n5ap+lyKXeo5jO65rwtECfsEERyLR9JwCAY1FqUiSawIBfcZTQrcdg8ubwIVutuU0\nWFlBhYZcPATXXK2lvw5M1UWVg4lOK6QdSLLhMsv6UKD6CxTTPWl66P6m2Wxy+5lp\naV0h/Xf4KGBx8XWE/f/2J+0CgYEAlfJGMZZut4pGLwhv4WqkngBf2VMDLa3Bcejn\n/4T9W5zQ7w0WLFDpg1quDa1P8JWh9j+ancc81Zp+1WB5u/zJLzXIkChgmeAHxLGC\nLMKNU+VuwtJHj7ajWD6AHogZ9Ff49K2HzRH2fRb1IKROY/7dC0Y43ppmCaEosTm8\nZalZzZ0CgYAF6KnlPw1qbpVZYyVcPw38omeisMs/92ovf4oqzJpZ5zCW1iKKRpQR\nq4oDLca0zJS/AUmXFTIG//VHvsna4A2JrmNS4rI+bEgwSDrRKHmX8Idctdu1ifgS\nrRtv1s5RfTZ67eWVYE6rUn1+EFBZwAzZorOXL3CzbjZEXUCJeOH6Lw==\n-----END RSA PRIVATE KEY-----"

 node-defaults netplan template "    network:\n        version: 2\n        ethernets:\n            ens192:\n                dhcp4: false\n                addresses:\n                - {{K8S_SSH_IP}}/24\n                routes:\n                - to: 0.0.0.0/0\n                  via: <<gateway>>\n                nameservers:\n                    addresses:\n                    - <<dns1>>\n                    - <<dns2>>\n                    search:\n                    - <<domain>>\n\n\n"

 node-defaults os ntp enabled
 node-defaults os ntp servers <<ntp server 1>>
 exit
 node-defaults os ntp servers <<ntp server 2>>
 exit
```

In the above config, we have enabled ingress for Grafana access and configured two ntp servers.

- Create AIO Node config:

```
clusters <<your cnBNG CP Cluster Name>>
 nodes <<your cnBNG CP Cluster VM Name>>>
  k8s node-type master
  k8s ssh-ip <<your cnBNG CP VM IP>>
  k8s node-labels disktype ssd
  exit
  k8s node-labels smi.cisco.com/node-type oam
  exit
  initial-boot default-user <<your cnBNG CP Cluster VM username>>>
  initial-boot default-user-password <<your password1>>
  vmware datastore <<ESXi Host Datastore>>
  vmware host <<ESXi Host IP>>
  vmware sizing ram-mb 16384
  vmware sizing cpus 8
  vmware sizing disk-root-gb 150
! attach NIC based on the port group config
  vmware nics "VM Network"
  exit
  os ntp enabled
 exit
exit
```

- cnBNG software is available as a tarball and it can be hosted on local http server for offline deployment. In this step we configure the software repository locations for tarball. We setup software cnf for both cnBNG CP and CEE. URL and SHA256 depends on the version of the image and the url location, so these two could change for your deployment.

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

- Setup Ops Center configs inside cluster for cnBNG

```
clusters <<your cnBNG CP Cluster Name>>
 ops-centers bng bng
  repository-local        bng
  sync-default-repository true
  netconf-ip              <<your cnBNG CP VM IP>>
  netconf-port            3022
  ssh-ip                  <<your cnBNG CP VM IP>>
  ssh-port                2024
  ingress-hostname        <<your cnBNG CP VM IP>>.nip.io
  initial-boot-parameters use-volume-claims true
  initial-boot-parameters first-boot-password <<your password1>>
  initial-boot-parameters auto-deploy false
  initial-boot-parameters single-node true
 exit
 ops-centers cee global
  repository-local        cee
  sync-default-repository true
  netconf-ip              <<your cnBNG CP VM IP>>
  netconf-port            3024
  ssh-ip                  <<your cnBNG CP VM IP>>
  ssh-port                2023
  ingress-hostname        <<your cnBNG CP VM IP>>.nip.io
  initial-boot-parameters use-volume-claims true
  initial-boot-parameters first-boot-password <<your password1>>
  initial-boot-parameters auto-deploy true
  initial-boot-parameters single-node true
 exit
exit
```

- Deploy cnBNG CP cluster using cluster sync command.

```
[inception] SMI Cluster Deployer# clusters pod1 actions sync run   
This will run sync.  Are you sure? [no,yes] yes
message accepted
[inception] SMI Cluster Deployer# 
```

- Monitor sync using below command. It will take 45mins to sync and deploy the cluster. 

```
monitor sync-logs pod1

E.g.

[inception] SMI Cluster Deployer# monitor sync-logs pod1
. . .
. . .
TASK [cluster-manager : Deleting cluster manager chart directory] **************
Thursday 18 November 2021  14:38:49 +0000 (0:00:00.042)       0:08:42.683 ***** 
skipping: [cnbng-cp]

PLAY RECAP *********************************************************************
cnbng-cp                   : ok=483  changed=209  unreachable=0    failed=0    skipped=441  rescued=0    ignored=0   
localhost                  : ok=57   changed=8    unreachable=0    failed=0    skipped=20   rescued=0    ignored=0   

Thursday 18 November 2021  14:38:49 +0000 (0:00:00.064)       0:08:42.748 ***** 
=============================================================================== 
2021-11-18 14:38:49.716 DEBUG cluster_sync.podX: Cluster sync successful 
2021-11-18 14:38:49.717 DEBUG cluster_sync.podX: Ansible sync done 
2021-11-18 14:38:49.717 INFO cluster_sync.podX: _sync finished.  Opening lock 
```


  
