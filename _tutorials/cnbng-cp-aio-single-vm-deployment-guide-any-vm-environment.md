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

Prerequisites: 
	- Inception Server (SMI Deployer) is pre deployed
	- cnBNG CP VM is pre-deployed using SMI Base ISO image

Step 1: Inception CM Deployment
