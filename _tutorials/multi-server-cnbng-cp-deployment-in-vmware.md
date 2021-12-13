---
published: true
date: '2021-12-13 08:39 +0530'
title: Multi-server cnBNG CP Deployment in VMWare
author: Gurpreet Dhaliwal
position: hidden
---
In this tutorial we will learn how to deploy high capacity cnBNG CP in multi server VMWare ESXi environment. The deployment of cnBNG CP is fully automated through Inception Deployer or Cluster Manager. 

We will deploy Multi server/node cnBNG CP in following steps:
1. VMWare ESXi Networking Preparation
1. Inception Server Deployment
1. cnBNG CP Cluster Configuration
1. cnBNG CP Deployment

## Prerequisite (optional, if DC already exists)
1. Make sure vCenter is installed (ver 6.7 is tested)
1. In the vCenter, right click and select New Data Center
	- Provide name of the data-center and click ok
1. Right click on the newly created datacenter and select New Cluster.
	- Provide name of the cluster and click ok (all other options remain default)
![vmware-dc1.png]({{site.baseurl}}/images/vmware-dc1.png)
1. Now add a host to the cluster. By selecting add host from right click menu on newly created cluster. Follow on screen instructions to add the host.

## Deployment Logical Topology

![cnbng-cp-multi-server1.png]({{site.baseurl}}/images/cnbng-cp-multi-server1.png)

## Step-1: VMWare ESXi Networking Preparation

We have following UCS hosts to deploy cnBNG CP

<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th> </th>
    <th>Host IP</th>
    <th>Datastore</th>
    <th>VMs Deployed</th>
  </tr>
  <tr>
    <td>Server-1</td>
    <td>10.81.103.64</td>
    <td>datastore01</td>
    <td>SMI Deployer, Master-1, ETCD-1, OAM-1</td>
  </tr>
  <tr>
    <td>Server-2</td>
    <td>10.81.103.65</td>
    <td>datastore02</td>
    <td>Master-2, ETCD-2, OAM-2</td>
  </tr>
  <tr>
    <td>Server-3</td>
    <td>10.81.103.66</td>
    <td>datastore03</td>
    <td>Master-3, ETCD-3, OAM-3</td>
  </tr>
  <tr>
    <td>Server-4</td>
    <td>10.81.103.141</td>
    <td>datastore141</td>
    <td>proto1, service1</td>
  </tr>
  <tr>
    <td>Server-5</td>
    <td>10.81.103.68</td>
    <td>datastore68</td>
    <td>proto2, session1</td>
  </tr>
  <tr>
    <td>Server-6</td>
    <td>10.81.103.63</td>
    <td>datastore1</td>
    <td>service2, session2</td>
  </tr>
</table>

We need following networks on ESXi hosts. Create following networks in VMWare ESXi by creating Port Groups, vSwitch and mapping vSwitches to correspinding vNICs.


<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th>Network</th>
    <th>Port Group</th>
    <th>vSwitch</th>
    <th>vNIC</th>
  </tr>
  <tr>
    <td>svc-net1</td>
    <td>PCIE0_VLANx</td>
    <td>vSwitch_PCIE0</td>
    <td>vnic corresponding to PCIE0 port</td>
  </tr>
  <tr>
    <td>svc-net1</td>
    <td>PCIE1_VLANx</td>
    <td>vSwitch_PCIE1</td>
    <td>vnic corresponding to PCIE1 port</td>
  </tr>
  <tr>
    <td>svc-net2</td>
    <td>PCIE0_VLANy</td>
    <td>vSwitch_PCIE0</td>
    <td>vnic corresponding to PCIE0 port</td>
  </tr>
  <tr>
    <td>svc-net2</td>
    <td>PCIE1_VLANy</td>
    <td>vSwitch_PCIE1</td>
    <td>vnic corresponding to PCIE1 port</td>
  </tr>
  <tr>
    <td>k8s-api-net</td>
    <td>PCIE0_VLANz</td>
    <td>vSwitch_PCIE0</td>
    <td>vnic corresponding to PCIE0 port</td>
  </tr>
  <tr>
    <td>k8s-api-net</td>
    <td>PCIE1_VLANz</td>
    <td>vSwitch_PCIE1</td>
    <td>vnic corresponding to PCIE1 port</td>
  </tr>
  <tr>
    <td>Management</td>
    <td>VM Network</td>
    <td>vSwitch_LOM</td>
    <td>vnic corresponding to LOM for management</td>
  </tr>
</table>

**Note**: VLANx, VLANy and VLANz could be based on the setup. In this tutorial we will be using z=312, y=311,x=310
{: .notice--info}

We will be deploying total of 16 VMs, since this setup is production quality this ensures proper Resiliency in the cluster and labelling of nodes to help place PODs for optimal performance. Below table list the NIC attachment for each VM type.

<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th>VM Name/Type</th>
    <th>NIC1</th>
    <th>NIC2</th>
    <th>NIC3</th>
  </tr>
  <tr>
    <td>Inception/SMI Deployer</td>
    <td>PCIE0_VLANz</td>
    <td>VM Network</td>
    <td></td>
  </tr>
  <tr>
    <td>master1</td>
    <td>PCIE0_VLANz</td>
    <td>VM Network</td>
    <td></td>
  </tr>
  <tr>
    <td>master2</td>
    <td>PCIE0_VLANz</td>
    <td>VM Network</td>
    <td></td>
  </tr>
  <tr>
    <td>master3</td>
    <td>PCIE0_VLANz</td>
    <td>VM Network</td>
    <td></td>
  </tr>
  <tr>
    <td>etcd1</td>
    <td>PCIE0_VLANz</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>etcd2</td>
    <td>PCIE0_VLANz</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>etcd3</td>
    <td>PCIE0_VLANz</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>oam1</td>
    <td>PCIE1_VLANz</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>oam2</td>
    <td>PCIE1_VLANz</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>oam3</td>
    <td>PCIE1_VLANz</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>proto1</td>
    <td>PCIE1_VLANz</td>
    <td>PCIE1_VLANx</td>
    <td>PCIE1_VLANy</td>
  </tr>
  <tr>
    <td>proto2</td>
    <td>PCIE1_VLANz</td>
    <td>PCIE1_VLANx</td>
    <td>PCIE1_VLANy</td>
  </tr>
  <tr>
    <td>service1</td>
    <td>PCIE0_VLANz</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>service2</td>
    <td>PCIE0_VLANz</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>session1</td>
    <td>PCIE0_VLANz</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>session2</td>
    <td>PCIE1_VLANz</td>
    <td></td>
    <td></td>
  </tr>
</table>

**Note**: VLANx, VLANy and VLANz could be based on the setup. In this tutorial we will be using z=312, y=311,x=310
{: .notice--info}

Let us now look at the IP addressing we will be using for this deployment. In this tutorial we will use one External Management network (10.81.103.0/24) and an Internal network (212.212.212.0/24) for k8s operations. We will also attach two service networks: 11.0.0.0/24 and 12.0.0.0/24. svc-net1 (11.0.0.0/24) will be used for cnBNG CP and UP communication whereas svc-net2 (12.0.0.0/24) will be used for cnBNG CP and Radius communication.

<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th>VM Name/Type</th>
    <th>NIC1 IP</th>
    <th>NIC2 IP</th>
    <th>NIC3 IP</th>
  </tr>
  <tr>
    <td>Inception/SMI Deployer</td>
    <td>212.212.212.100</td>
    <td>10.81.103.100</td>
    <td></td>
  </tr>
  <tr>
    <td>master1</td>
    <td>212.212.212.11</td>
    <td>10.81.103.102</td>
    <td></td>
  </tr>
  <tr>
    <td>master2</td>
    <td>212.212.212.12</td>
    <td>10.81.103.103</td>
    <td></td>
  </tr>
  <tr>
    <td>master3</td>
    <td>212.212.212.13</td>
    <td>10.81.103.104</td>
    <td></td>
  </tr>
  <tr>
    <td>etcd1</td>
    <td>212.212.212.14</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>etcd2</td>
    <td>212.212.212.15</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>etcd3</td>
    <td>212.212.212.16</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>oam1</td>
    <td>212.212.212.17</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>oam2</td>
    <td>212.212.212.18</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>oam3</td>
    <td>212.212.212.19</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>proto1</td>
    <td>212.212.212.20</td>
    <td>11.0.0.2</td>
    <td>12.0.0.2</td>
  </tr>
  <tr>
    <td>proto2</td>
    <td>212.212.212.21</td>
    <td>11.0.0.3</td>
    <td>12.0.0.3</td>
  </tr>
  <tr>
    <td>service1</td>
    <td>212.212.212.22</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>service2</td>
    <td>212.212.212.23</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>session1</td>
    <td>212.212.212.24</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>session2</td>
    <td>212.212.212.25</td>
    <td></td>
    <td></td>
  </tr>
</table>

We would also need Virtual IPs which will be used keepalived between similar nodes to provide HA:

<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th>VIP Type/Type</th>
    <th>IP Address</th>
    <th>Remarks</th>
  </tr>
  <tr>
    <td>Master VIP1</td>
    <td>10.81.103.101</td>
    <td>For management access</td>
  </tr>
  <tr>
    <td>Master VIP2</td>
    <td>212.212.212.101</td>
    <td>k8s-api-net access</td>
  </tr>
  <tr>
    <td>Proto VIP1</td>
    <td>212.212.212.102</td>
    <td>k8s-api-net access</td>
  </tr>
  <tr>
    <td>Proto VIP2</td>
    <td>11.0.0.1</td>
    <td>svc-net-1 VIP, will be used for peering with cnBNG UP</td>
  </tr>
 <tr>
    <td>Proto VIP3</td>
    <td>12.0.0.1</td>
    <td>svc-net-2 VIP, will be used for Radius communication</td>
  </tr>
</table>

## Step 2: Inception Server Deployment

Follow the steps detailed before on [Inception Server Deployment](https://xrdocs.io/cnbng/tutorials/inception-server-deployment-guide/). However attach two NICs to Inception Server VM , as per the NIC attachment plan described in Step-1.

## Step 3: SSH Key Generation
This is for security reasons, where cnBNG CP VM can only be accessed by ssh key and not through password by default.

- SSH Login to Inception VM
- Generate SSH key using:

```
ssh-keygen -t rsa
```

- Note down ssh keys in a file from .ssh/id_rsa (private) and ./ssh/id_rsa.pub (public)
- Remove line breaks from private key and replace them with string "\n"
- Remove line breaks from public key if there are any.

## Step 4: cnBNG CP Cluster Configuration

- Let us first define the VMWare environment

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

- We will now define the software repos for cnBNG CP and CEE. cnBNG software is available as a tarball and it can be hosted on local http server for offline deployment. In this step we configure the software repository locations for tarball. We setup software cnf for both cnBNG CP and CEE. URL and SHA256 depends on the version of the image and the url location, so these two could change for your deployment.

```
software cnf bng
  url             http://192.168.107.148/your/image/location/bng.2021.04.m0.i74.tar
  sha256          e36b5ff86f35508539a8c8c8614ea227e67f97cf94830a8cee44fe0d2234dc1c
  description     bng-products
exit
software cnf cee
  url         http://192.168.107.148/your/image/location/cee-2020.02.6.i04.tar
  sha256      b5040e9ad711ef743168bf382317a89e47138163765642c432eb5b80d7881f57
  description cee-products
exit
```

- Next we define Cluster configuration based on our deployment schemel. The configuration is self explanatory. However do note two kinds of VIPs define here for master.   

```
clusters cnbng-cp-cluster1
 environment vmware
 addons ingress bind-ip-address 10.81.103.101
 addons ingress bind-ip-address-internal 212.212.212.101
 addons ingress enabled
 addons istio enabled
 configuration master-virtual-ip        212.212.212.101
 configuration master-virtual-ip-cidr   24
 configuration master-virtual-ip-interface ens192
 configuration additional-master-virtual-ip 10.81.103.101
 configuration additional-master-virtual-ip-cidr 24
 configuration additional-master-virtual-ip-interface ens224
 configuration virtual-ip-vrrp-router-id 60
 configuration pod-subnet               192.212.0.0/16
 configuration allow-insecure-registry  true
 node-defaults initial-boot default-user username1
 node-defaults initial-boot default-user-ssh-public-key "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDDH4uQdrxTvgFg32szcxLUfwrxmI513ySoDkjznncofvkL/JV122JAsGHvL6T39jBRKytuPpLhvr9ZvwKlOZ5BeeUB+Zj8tuZdQRqIpvonhtcESc11oRM6w5dQjHUIaDdK/vBQQg5hSQ0CVzvVU2DS8E+csMbBoPzaUVi+qPjqfBEt9MzMysc1DnnXN1fFMDVzTAsb5mgt/cLjcfbo6qVgLt9irFJ/AfoQGltQi9M4vehsprYZngGL5U9+qt9qn4lbaxTt2HqNQJAuk701LVhZNXKCfHmgg6en3IYk7YZI1IhhAnYAqlQLuC7df9DqE6i0iB8eTuRh5I2KsPUrCLGv cloud-user@inception"
 node-defaults initial-boot default-user-password password1
 node-defaults k8s ssh-username username1
 node-defaults k8s ssh-connection-private-key "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAwx+LkHa8U74BYN9rM3MS1H8K8ZiOdd8kqA5I8553KH75C/yV\nddtiQLBh7y+k9/YwUSsrbj6S4b6/Wb8CpTmeQXnlAfmY/LbmXUEaiKb6J4bXBEnN\ndaETOsOXUIx1CGg3Sv7wUEIOYUkNAlc71VNg0vBPnLDGwaD82lFYvqj46nwRLfTM\nzMrHNQ551zdXxTA1c0wLG+ZoLf3C43H26OqlYC7fYqxSfwH6EBpbUIvTOL3obKa2\nGZ4Bi+VPfqrfap+JW2sU7dh6jUCQLpO9NS1YWTVygnx5oIOnp9yGJO2GSNSIYQJ2\nAKpUC7gu3X/Q6hOotIgfHk7kYeSNirD1KwixrwIDAQABAoIBADKgQKne5MYlil4E\nGeBjfwM7Yy+EEZJrrysbabor52bOave9NVo67aczHHXeusLLUYX92WrlOV7xCtzS\nPnF4HaOHaO+2PwdyvRp9BdFm4YjX53npXDGk9URN8zim+MaRo6cFtnxcZza+qW1u\nDMwwsfKI/178TtV2W6SZbpkpZkwQJ/cN1fljxuQYubOujMZErc1Q7TlIQYrG8XVz\naOTVkYBU+O2DOKkWp4AM9qe/0RIdjmz1ZZHRB83iF3/OaU3j2mhlhRQwTiS3jcGX\nkWoovz/GmFAcw+VSz81tz2UFmGNczkl8F5t4afYq31M7r+restSd6Mc7dcqWA7pR\nYflz7SECgYEA+qleQDAxe9cSRyGk0aCXV7NKQnvmIReSiRxibNne2FTnLaBH7iM7\nGe8XT23gFLCvM1qmpaLxjQZEP4A7WTaZuTEQAvVHYM+4m+1XDz4+ePaSCFwru8Q/\nLvE0h45JGfuHiXIWQNsyycSnDa+50BQ1a+EruzsmdHDvVnhO/QK0gzsCgYEAx0dg\nVbOjGzEvuGTfTI8qqRYz2XQjSKr6IHqRGY/h2hVcJo0Is+fr9n2QlebJ9oIOVnw7\npSlPgnMNcI9b74aCGuVnVUW/23TRkTvq6goNBa2V++d9EHPNo2gahc1tuJZzkp8I\nnZONBHzU8jwe9nBbK7uuz9i8cgSnMaWbS6Q8PB0CgYEAoXzoWdYyqyQ+hFEqjFs3\n5ap+lyKXeo5jO65rwtECfsEERyLR9JwCAY1FqUiSawIBfcZTQrcdg8ubwIVutuU0\nWFlBhYZcPATXXK2lvw5M1UWVg4lOK6QdSLLhMsv6UKD6CxTTPWl66P6m2Wxy+5lp\naV0h/Xf4KGBx8XWE/f/2J+0CgYEAlfJGMZZut4pGLwhv4WqkngBf2VMDLa3Bcejn\n/4T9W5zQ7w0WLFDpg1quDa1P8JWh9j+ancc81Zp+1WB5u/zJLzXIkChgmeAHxLGC\nLMKNU+VuwtJHj7ajWD6AHogZ9Ff49K2HzRH2fRb1IKROY/7dC0Y43ppmCaEosTm8\nZalZzZ0CgYAF6KnlPw1qbpVZYyVcPw38omeisMs/92ovf4oqzJpZ5zCW1iKKRpQR\nq4oDLca0zJS/AUmXFTIG//VHvsna4A2JrmNS4rI+bEgwSDrRKHmX8Idctdu1ifgS\nrRtv1s5RfTZ67eWVYE6rUn1+EFBZwAzZorOXL3CzbjZEXUCJeOH6Lw==\n-----END RSA PRIVATE KEY-----"
```
- Define Common Netplan for all VMs. We will only define k8s-api-net here. Based on attachments we will add new networks.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
network:
    version: 2
    ethernets:
{ % if master_vm is defined and master_vm == 'true' %}   
        ens192:
            addresses:
            - \{{K8S_SSH_IP\}}/24
            dhcp4: false
            routes:
            -   metric: 50
                to: 0.0.0.0/0
                via: 212.212.212.101                                        
        ens224:
            dhcp4: false
            nameservers:
                addresses:
                - 64.102.6.247
                search:
                - cisco.com
            gateway4: 10.81.103.1                                                     
{\% else \%}
        ens192:
            addresses:
            - {{K8S_SSH_IP}}/24
            dhcp4: false           
            routes:
            -   metric: 50
                to: 0.0.0.0/0
                via: 212.212.212.101    
{\% endif \%}
</code>
</pre>
</div>
