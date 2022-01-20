---
published: false
date: '2022-01-20 15:48 +0530'
title: cnBNG CP Multiserver (Baremetal) Deployment Guide
author: Gurpreet Dhaliwal
excerpt: >-
  Learn how to deploy cnBNG CP HA setup on baremetal UCS servers. Baremetal
  deployment is also called as CNDP deployment, it allows cnBNG CP to be
  deployed without need of running hypervisor. CNDP deployment offers 20%
  performance improvement over traditional hypervisor/VM based deployment.
---

{% include toc %}

In this tutorial we will learn how to deploy high capacity cnBNG CP on baremetal UCS Servers. Deployment of cnBNG CP on Baremetal Server is also known as CNDP (Cloud Native Development Platform) Deployment. The deployment of cnBNG CP is fully automated through Inception Deployer or Cluster Manager. 

We will deploy Multi server cnBNG CP in following steps:
	- CNDP Preparation
	- SSH Key Generation
	- cnBNG CP Cluster Configuration
	- cnBNG CP Deployment

## Logical Deployment Topology

![cndp-logical.png]({{site.baseurl}}/images/cndp-logical.png)

## Physical Deployment Topology

![cndp-physical.png]({{site.baseurl}}/images/cndp-physical.png)

