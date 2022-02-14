---
published: true
date: '2022-02-14 18:03 +0530'
title: PPPoE DS Subscriber Bringup in cnBNG
author: Gurpreet Dhaliwal
excerpt: >-
  Learn how to bring a PPPoE Dualstack session in this tutorial. This tutorial
  covers pap/chap auth for PPPoE subscriber and accounting.
position: hidden
---
## Introduction

To bringup a PPPoE dualstack(DS) subscriber in cnBNG, we need to setup some configurations on both cnBNG CP and cnBNG UP (ASR9k). In this tutorial we will bringup a PPPoE DS subscriber session with plan policies and required ACL received during authentication (pap/chap) from freeradius.  We will use IPCP to assign IPv4 address and DHCPv6 for IPv6 address (IANA+IAPD) assignment to the subscriber. 

## Topology

In this topology we are using Spirent as the CPE, cnBNG will assign both IPv4 and IPv6 (NA + PD) prefixes to CPE. IPv4 prefix will be assigned using IPCP whereas for IPv6 prefix assignments client will use DHCPv6. The prefix assignment will happen through IPAM which is integral part of cnBNG CP. 

Radius profile configured here is to activate 100mbps plan service with v4 and v6 ACLs applied to dynamic subscriber interface on UP. 

![pppoe-ds-topo.png]({{site.baseurl}}/images/pppoe-ds-topo.png)

## Prerequisite:

- cnBNG CP is already deployed and Ops Center is accessible. We will be using Ops Center CLI interface for configurations in this tutorial
- cnBNG CP initial configuration is applied
- cnBNG CP service network can reach cnBNG UP:
	- In multi VM deployment, this means protocol VM VIP for service network is reachable from cnBNG UP for PFCP communications
    - In Baremetal CNDP deployment, this means protocol VM VIP IP corresponding to the service network to reach cnBNG UP for PFCP communications
	- In single Node/VM AIO deployment, this means AIO VM IP is reachable from cnBNG UP
    
## cnBNG CP Configuration

cnBNG CP Configuration has following constructs/parts:
- IPAM
- Profile PPPoE
- Profile DHCP
- Profile AAA
- Profile Radius
- Profile Feature-Template
- Profile Subscriber
- User-Plane

Let's understand each one step-by-step and apply in Ops Center in config mode.

### IPAM
This is where we define subscriber address pools for IPv4, IPv6 (NA) and IPv6 (PD). These are the pools from which CPE will get the IPs. IPAM assigns addresses dynamically by splitting address pools into smaller chunks. And then associating each chunk with a user-plane. The pools get freed up dynamically and re-allocated to different user-planes on need basis. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
ipam
 instance 1
  source local
  address-pool pool-ISP1
   vrf-name default
   ipv4
    split-size
     per-cache 262144
     per-dp    262144
    exit
    <mark>address-range 20.0.0.1 20.0.255.254</mark>
   exit
   ipv6
    address-ranges
     split-size
      per-cache 262144
      per-dp    262144
     exit
     <mark>address-range 2001::1 2001::1:100</mark>
    exit
    prefix-ranges
     split-size
      per-cache 65536
      per-dp    65536
     exit
     <mark>prefix-range 2001:1:: length 48</mark>
     <mark>prefix-range 2001:2:: length 48</mark>
    exit
   exit
  exit
exit
</code>
</pre>
</div>

### Profile PPPoE
This profile is same as the BBA Group which was defined on ASR9k integrated BNG solution. We define service names etc. For this tutorial we will keep it simple and only specify the MTU.

```
profile pppoe ppp1
 mtu 1494
exit
```

### Profile DHCP
Incase of PPPoE DS subscribers we will be using the DHCPv6 server to assign the IPv6 (IANA+IAPD) prefixes to CPE. For thsi example we will have cnBNG CP act as a DHCP server to assign IPv6 addresses to CPE/subscribers. In profile DHCP we define the DHCP server and which IPAM pool to use by default for subscriber. We can use different pools for IPv4, IPv6 (IANA) and IPv6 (IAPD). 

```
profile dhcp dhcp-server1
 ipv4
  mode server
  server
   pool-name   pool-ISP1
   dns-servers [ 8.8.8.8 ]
   lease days 1
  exit
 exit
 ipv6
  mode server
  server
   iana-pool-name pool-ISP1
   iapd-pool-name pool-ISP1
   lease days 1
  exit
 exit
exit
```

**Note**: The definition of IPv4 server profile is not needed for PPPoE subscribers. For PPPoE subscribers IPv4 address will be assigned by IPCP and from IPAM directly.
{: .notice--info}

### Profile AAA
This profile defines the AAA parameters, like which Radius group to be used for authentication/authorization and accounting. In this tutorial we will be using radius group defined as "local" under radius profile for authentication and accounting.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
profile aaa aaa_pppoe-1
 authentication
  method-order [ <mark>local</mark> ]
 exit
 accounting
  method-order [ <mark>local</mark> ]
 exit
exit
</code>
</pre>
</div>

### Profile Radius
Under this profile, Radius groups are created.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
profile server-group <mark>local</mark>
 radius-group <mark>local</mark>
exit

profile radius
 algorithm round-robin
 deadtime  3
 detect-dead-server response-timeout 60
 max-retry 2
 timeout   5
 <mark>!!! Radius server IP and port definitions for auth and acct</mark>
 server <mark>192.168.107.152 1812</mark>
  type   auth
  secret cisco
 exit
 server <mark>192.168.107.152 1813</mark>
  type   acct
  secret cisco
 exit
 attribute
  nas-identifier CISCO-BNG
  <mark>!!! This should be protocol VIP to reach Radius</mark>
  nas-ip         <mark>192.168.107.165</mark>
 exit
 server-group <mark>local</mark>
  server auth <mark>192.168.107.152 1812</mark>
  exit
  server acct <mark>192.168.107.152 1813</mark>
  exit
 exit
exit
<mark>!!! we can also set COA client</mark>
profile <mark>coa</mark>
 client <mark>192.168.107.152</mark>
  server-key cisco
 exit
exit
</code>
</pre>
</div>

### Profile Feature-template
This profile defines subscriber feature template. This is the template which will be applied to dynamic subscriber interface. We also enable service/ session accounting here.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
profile feature-template pppoe-1
 vrf-name default
 ipv4
  mtu 1500
 exit
 session-accounting
  enable
  aaa-profile       aaa_pppoe-1
  periodic-interval 1800
 exit
 ppp
  authentication [ pap chap ]
  <mark>!!! will use IPAM pool-ISP1 for IPv4 address assignment using IPCP</mark>
  ipcp peer-address-pool <mark>pool-ISP1</mark>
  ipcp renegotiation ignore
  ipv6cp renegotiation ignore
  lcp renegotiation ignore
  max-bad-auth   4
  max-failure    5
  timeout absolute 1440
  timeout authentication 5
  timeout retry  4
  <mark>!!! the following command will offload PPP keepalives to cnBNG UP</mark>
  keepalive interval 30 retry 5
 exit
exit
</code>
</pre>
</div>

We can also define service profiles using feature-template, which gets applied on per subscriber session. The service profile in case of radius can be applied during authentication/authorization using service activate attribute or it can also be applied using CoA.

```
profile feature-template FT_Plan_100mbps
 qos
  in-policy  PM_Plan_100mbps_input
  out-policy PM_Plan_100mbps_output
 exit
exit
```

**Note**: In above policy-map PM_Plan_100mbps_input and PM_Plan_100mbps_output are expected to be defined on userplane.
{: .notice--info}

### Profile Subscriber
This profile can be attached on per access port level or per user-plane level. This profile for PPPoE defines which dhcp server profile to apply for IPv6 address assignment, along with feature-template, pppoe-profile and aaa-profile to be used for auth/acct. 

```
profile subscriber subscriber-profile_pppoe-1
 dhcp-profile               dhcp-server1
 pppoe-profile              ppp1
 session-type               ipv4v6
 activate-feature-templates [ pppoe-1 ]
 event session-activate
  aaa authenticate aaa_pppoe-1
 exit
exit
```



