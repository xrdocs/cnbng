---
published: true
date: '2021-12-13 08:39 +0530'
title: Multi-server cnBNG CP Deployment in VMWare
author: Gurpreet D
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



