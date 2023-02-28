---
published: false
date: '2023-02-28 10:47 +0530'
title: cnBNG CP (AIO) Single VM Deployment Guide - Any VM Environment
---

cnBNG Control Plane deployment in single VM in any NFVI environment where cnBNG CP VM is pre-deployed is called as cnBNG CP AIO Manual Deployment. This is a single VM deployment for cnBNG Control Plane in a customized Ubuntu VM (provided by SMI) and includes following:
	- SMI cluster (Cisco K8s Distribution)
	- CEE application (Application Infrastructure for Monitoring and Alerting)
	- cnBNG Control Plane application (Control Plane application for BNG)

For AIO deployment following steps are followed:
	1. Inception Server Cluster Manager deployment
	2. Base ISO Ubuntu VM deployment for cnBNG CP
	3. cnBNG CP base ISO Ubuntu OS customizations
	4. SSH Key generation
	5. SMI, CEE and cnBNG CP deployment using SMI Deployer

