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