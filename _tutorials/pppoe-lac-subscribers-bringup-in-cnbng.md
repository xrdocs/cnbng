---
published: true
date: '2022-06-28 15:48 +0530'
title: PPPoE LAC Subscribers Bringup in cnBNG
position: hidden
author: Gurpreet Dhaliwal
---
## Introduction

In this tutorial we will learn how to bring-up a PPPoE LAC subscriber session in Cloud Native BNG (cnBNG). 

## Topology
The setup used for this tutorial is shown in figure 1. This setup uses Spirent to emulate client and L2TP Network Server (LNS). Spirent port 1/2 will be used for client emulation: PTA and LAC on same port. Which connects to the Access Network Provider ASR9k BNG UP node. LNS is emulated by Spirent port 1/1. When client tries to connect on ASR9k BNG UP, cnBNG CP authenticates the client with AAA server. Based on attribues received in Access Accept from Radius, client is either terminated as PTA session on cnBNG or as LAC session on cnBNG.

![lac-topo.png]({{site.baseurl}}/images/lac-topo.png)
