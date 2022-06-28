---
published: true
date: '2022-02-14 18:03 +0530'
title: PPPoE DS (DHCPv6) Subscriber Bringup in cnBNG
author: Gurpreet Dhaliwal
excerpt: >-
  Learn how to bring a PPPoE Dualstack session in this tutorial. This tutorial
  covers pap/chap auth for PPPoE subscriber and accounting.
position: top
---
{% include toc %}

## Introduction

To bringup a PPPoE dualstack(DS) subscriber in cnBNG, we need to setup some configurations on both cnBNG CP and cnBNG UP (ASR9k). In this tutorial we will bringup a PPPoE DS subscriber session with plan policies and required ACL received during authentication (pap/chap) from freeradius.  We will use IPCP to assign IPv4 address and DHCPv6 for IPv6 address (IANA+IAPD) assignment to the subscriber. 

## Topology

In this topology we are using Spirent as the CPE, cnBNG will assign both IPv4 and IPv6 (NA + PD) prefixes to CPE. IPv4 prefix will be assigned using IPCP whereas for IPv6 prefix assignments client will use DHCPv6. The prefix assignment will happen through IPAM which is integral part of cnBNG CP. 

Radius profile configured here is to activate 100mbps plan service with v4 and v6 ACLs applied to dynamic subscriber interface on UP. 

![pppoe-ds-topo.png]({{site.baseurl}}/images/pppoe-ds-topo.png)

## Prerequisite

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
  <mark>keepalive interval 30 retry 5</mark>
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

### User-plane
This construct define the association configs. Peering IP as well as subscriber profile to be attached to user-plane or at port level. In this tutorial we will attach subscriber profile at port level.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
user-plane ASR9k-1
 <mark>!!! this should be the IP of ASR9k to which this control-plane will peer with</mark>
 peer-address ipv4 192.168.107.142
 <mark>!!! the port-id here is the ASR9k access port or interface name</mark>
 port-id <mark>Bundle-Ether1.102</mark>
  subscriber-profile subscriber-profile_pppoe-1
 exit
exit
</code>
</pre>
</div>

## cnBNG UP Configuration

UP Configuration has mainly four constructs for cnBNG

- Loopback for cnBNG CP-UP association
- DHCP Configuration
- Access Interface
- Feature definitions: QoS, ACL

### Loopback Creation
We need to create a Loopback for cnBNG use.

```
interface Loopback1
 ipv6 enable
```

### Association Configuration

This is where we define association settings between cnBNG CP and UP. The auto-loopback with "secondary-address-upadte enable" will allow dynamic IP address allocations using IPAM. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
cnbng-nal location 0/RSP0/CPU0
 hostidentifier ASR9k-1
 <mark>!!! cnBNG UP routable IP (may be loopback or direct interface IP) used for peering with cnBNG CP</mark>
 up-server ipv4 <mark>192.168.107.142</mark> vrf default
 <mark>!!! cnBNG CP IP (generally protocol VIP) used for peering with cnBNG UP</mark>
 cp-server primary ipv4 <mark>192.168.107.165</mark>
 auto-loopback vrf default
  interface Loopback1
   <mark>!!! Any dummy IP</mark>
   primary-address <mark>1.1.1.1</mark>
  !
 !
 cp-association retry-count 5
 secondary-address-update enable
!
</code>
</pre>
</div>

**Note**: NAL stands for Network Adaptation Layer for Cloud Native BNG in IOS-XR
{: .notice--info}
**Note**: cnBNG CP and UP doesnot require to be on same LAN, they need L3 connectivity for peering
{: .notice--info}

### DHCP Configuration
This is where we associate access interfaces with cnBNG DHCP profile. cnBNG specific DHCP profile makes sure DHCP packets are punted to cnBNG CP through CPRi/GTP-u tunnel. Since PPPoE subscruibers use dhcp only for IPv6 address assignment, dhcp ipv4 profile is not needed for PPPoE subscribers.
```
dhcp ipv6
 profile cnbng_v6 cnbng
 !
 interface Bundle-Ether1.102 cnbng profile cnbng_v6
```

### Access Interface Configuration
We define and associate access interface to cnBNG. This way control packets (based on configurations) get routed to the cnBNG CP. The contruct follows ASR9k Integarted BNG model, if you are familiar with.

```
interface Bundle-Ether1.102
 ipv4 point-to-point
 ipv4 unnumbered Loopback1
 ipv6 enable
 pppoe enable
 encapsulation ambiguous dot1q 102 second-dot1q any
!
```

**Note**: Thsi example uses ambiguous VLAN for access interface which allows 1:1 VLAN model. cnBNG also supports N:1 VLAN model for subscribers.
{: .notice--info}

## Radius Profile
Following is Freeradius profile used in this tutorial

```
cisco Cleartext-Password:="cisco"
  cisco-avpair += "subscriber:inacl=iACL_BNG_IPv4",
  Cisco-AVpair += "subscriber:ipv6_inacl=iACL_BNG_IPv6",
  cisco-avpair += "subscriber:sa=FT_Plan_100mbps"
```

## Verifications

- Verfiy that the cnBNG CP-UP association is up and Active on cnBNG CP ops-center

<div class="highlighter-rouge">
<pre class="highlight">
<code>
[cnbng-tme-lab/bng] bng# <mark>show peers</mark>
Mon Feb  14 13:24:13.982 UTC+00:00
GR                                                                                                  CONNECTED                                                INTERFACE  
INSTANCE  ENDPOINT      LOCAL ADDRESS         PEER ADDRESS          DIRECTION  POD INSTANCE   TYPE  TIME       RPC     ADDITIONAL DETAILS                    NAME       
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
0         RadiusServer  -                     192.168.107.152:1812  Outbound   radius-ep-0    Udp   5 hours    Radius  Status: Active,Type: Auth             &lt;none&gt;     
0         RadiusServer  -                     192.168.107.152:1813  Outbound   radius-ep-0    Udp   5 hours    Radius  Status: Active,Type: Acct             &lt;none&gt;      
<mark>1         n4            192.168.107.165:8805  192.168.107.142:8805  Inbound    bng-nodemgr-0  Udp   4 hours    UPF     Name: ASR9k-1,Nm: 0/0,Status: ACTIVE  &lt;none&gt;  </mark>    
</code>
</pre>
</div>

- Verify that the CP-UP Association is Up and Active on cnBNG UP

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP0/CPU0:ASR9k-1#<mark>show cnbng-nal cp connection status </mark>
Sun Jan 23 08:51:52.041 IST

Location: 0/RSP0/CPU0


User-Plane configurations:
-------------------------
 IP             : <mark>192.168.107.142</mark>   
 GTP Port       : 2152
 PFCP Port      : 8805
 VRF            : default


Control-Plane configurations:
----------------------------
 PRIMARY IP     : <mark>192.168.107.165</mark>  
 GTP Port       : 2152
 PFCP Port      : 8805 
 
 Association retry count: 5

 Connection Status: <mark>Up</mark>
 Connection Status time stamp:  Sun Jan 23 04:35:05 2022

 Connection Prev Status: Down
 Connection Prev Status time stamp:  Sun Jan 23 04:06:52 2022

 Association status: <mark>Active</mark>
 Association status time stamp: Sun Jan 23 04:35:04 2022
</code>
</pre>
</div>

- Verify subscriber session is up on cnBNG CP ops-center

<div class="highlighter-rouge">
<pre class="highlight">
<code>
[cnbng-tme-lab/bng] bng# <mark>show subscriber session detail</mark> 
Mon Feb  14 13:31:58.431 UTC+00:00
subscriber-details 
{
  "subResponses": [
    {
      "subLabel": "16777219",
      <mark>"mac": "0010.9400.0009"</mark>,
      "acct-sess-id": "cnbng-tme-lab_DC_16777219",
      "upf": "ASR9k-1",
      "port-id": "Bundle-Ether1.102",
      "up-subs-id": "2147711504",
      "sesstype": "ppp",
      <mark>"state": "established"</mark>,
      "subCreateTime": "Mon, 14 Feb 2022 09:59:04 UTC",
      "transId": "6",
      "subsAttr": {
        "attrs": {
          "Authentic": "RADIUS(1)",
          "Framed-Protocol": "PPP(1)",
          "Interface-Id": "0x7026f8fae3ac86e5",
          <mark>"addr": "20.0.0.4"</mark>,
          <mark>"addrv6": "2001::5"</mark>,
          "authen-type": "pap(1)",
          "client-mac-address": "0010.9400.0009",
          "connect-progress": "DUAL_STACK_OPEN(249)",
          <mark>"delegated-prefix": "2001:1::/64"</mark>,
          "dhcpv6-client-id": "0x00010001620a2570001094000009",
          "inner-vlan-id": "1",
          "outer-vlan-id": "102",
          "physical-adapter": "0",
          "physical-chassis": "0",
          "physical-port": "1",
          "physical-slot": "0",
          "physical-subslot": "0",
          "port-type": "PPPoE over QinQ(34)",
          "pppoe-session-id": "3",
          "protocol-type": "ppp(2)",
          "service-type": "Framed(2)",
          "string-session-id": "cnbng-tme-lab_DC_16777219",
          "username": "cisco",
          "vrf": "default"
        }
      },
      "subcfgInfo": {
        "committedAttrs": {
          "attrs": {
            "accounting-list": "aaa_pppoe-1",
            "acct-interval": "1800",
            "addr-pool": "pool-ISP1",
            <mark>"inacl": "iACL_BNG_IPv4"</mark>,
            "ipv4-mtu": "1500",
            <mark>"ipv6_inacl": "iACL_BNG_IPv6"</mark>,
            "ppp-authentication": "pap,chap",
            "ppp-ipcp-reneg-ignore": "true",
            "ppp-ipv6cp-reneg-ignore": "true",
            "ppp-keepalive-interval": "30",
            "ppp-keepalive-retry": "5",
            "ppp-lcp-reneg-ignore": "true",
            "ppp-max-bad-auth": "4",
            "ppp-max-failure": "5",
            "ppp-timeout-abs-minutes": "1440",
            "ppp-timeout-authentication": "5",
            "ppp-timeout-retry": "4",
            "session-acct-enabled": "true",
            "vrf": "default"
          }
        },
        "activatedServices": [
          {
            "serviceName": "pppoe-1",
            "serviceAttrs": {
              "attrs": {}
            }
          },
          {
            "serviceName": "FT_Plan_100mbps",
            "serviceAttrs": {
              "attrs": {
                <mark>"sub-qos-policy-in": "PM_Plan_100mbps_input"</mark>,
                <mark>"sub-qos-policy-out": "PM_Plan_100mbps_output"</mark>
              }
            }
          }
        ]
      },
      "smupstate": "smUpSessionCreated",
      "v4AfiState": "Up",
      "v6AfiState": "Up",
      "interimInterval": "1800",
      "interimSentToUp": "1740",
      "sessionAccounting": "enable",
      "serviceAccounting": "disable",
      "upAttr": {
        "attrs": {
          "Interface-Id": "0x7026f8fae3ac86e5",
          "addr": "20.0.0.4",
          "addrv6": "2001::5",
          "delegated-prefix": "2001:1::/64",
          "ppp-keepalive-interval": "30",
          "ppp-keepalive-retry": "5",
          "ppp-local-magic-number": "4060452463",
          "ppp-mtu": "1500",
          "ppp-peer-magic-number": "2318123741"
        }
      },
      "chargingInfo": {
        "sessionType": "CHARGING_AF_IPv4_IPv6",
        "sessionAccounting": {
          "periodicInterval": 1800,
          "accountingProvision": "Enable",
          "stateInfo": "CHARGING_STATE_START_ACK",
          "accountingStart": {
            "reqStartSuccess": "Mon, 14 Feb 2022 09:59:13 UTC"
          },
          "accountingUpdate": {
            "reqInterimSuccess": "Mon, 14 Feb 2022 13:29:20 UTC",
            "periodicAccountingProvision": "Enable",
            "InterimIntervalTimeout": 1800,
            "totalInterimReq": 10,
            "totalInterimFailure": 0
          },
          "accountingStop": {},
          "sessionDataStats": {
            "inputPkts": 431,
            "outputPkts": 495,
            "inputOctet": 30142,
            "outputOctet": 26437
          },
          "acct-sess-id": "cnbng-tme-lab_DC_16777219"
        }
      },
      "sess-events": [
        "Time, Event, Status",
        "2022-02-14 09:59:04.504 +0000 UTC, SessionCreate, success",
        "2022-02-14 09:59:08.524 +0000 UTC, SessionActivate, success",
        "2022-02-14 09:59:13.552 +0000 UTC, SessionUpdate, success",
        "2022-02-14 09:59:13.741 +0000 UTC, SessionUpdate, success",
        "2022-02-14 10:00:15.836 +0000 UTC, SessionUpdate, success",
        "2022-02-14 10:06:47.424 +0000 UTC, SessionUpdate, success",
        "2022-02-14 10:06:57.01 +0000 UTC, SessionUpdate, success"
      ],
      "dhcpAuditId": 1,
      "pppAuditId": 4
    }
  ]
}
</code>
</pre>
</div>

- Verify that the subscriber session is up and working on cnBNG UP

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP0/CPU0:ASR9k-1#<mark>show cnbng-nal subscriber all</mark>
Sun Jan 23 08:59:04.282 IST

Location: 0/RSP0/CPU0
Codes: CN - Connecting, CD - Connected, AC - Activated,
       ID - Idle, DN - Disconnecting, IN - Initializing


CPID(hex)  Interface               State  Mac Address     Subscriber IP Addr / Prefix (Vrf) Ifhandle
---------------------------------------------------------------------------------------------------
<mark>1000003    BE1.102.pppoe2147711504 AC     0010.9400.0009  20.0.0.4 (default) 0x67420   
                                                          2001::5 (IANA)
                                                          2001:1::/64 (IAPD)</mark>
Session-count: 1

RP/0/RSP0/CPU0:ASR9k-1#<mark>show subscriber running-config interface name BE1.102.pppoe2147711504</mark>
Sun Jan 23 08:59:43.855 IST
Building configuration...
!! IOS XR Configuration 7.4.2.32I
subscriber-label 0x80037a10
dynamic-template
 type user-profile U00037a10
  <mark>ipv6 access-group iACL_BNG_IPv6 ingress</mark>
  <mark>ipv4 access-group iACL_BNG_IPv4 ingress</mark>
  ipv4 mtu 1500
  ipv4 unnumbered Loopback1
  ipv6 enable
 !
 type service-profile FT_Plan_100mbps
  <mark>service-policy input PM_Plan_100mbps_input</mark>
  <mark>service-policy output PM_Plan_100mbps_output</mark>
 !
!
end

* Suffix indicates the configuration item can be added by aaa server only
RP/0/RSP0/CPU0:ASR9k-1#
</code>
</pre>
</div>

**Note**: "show subscriber running-config" is the same CLI used on ASR9k integrated BNG, so ignore- Suffix indicates the configuration item can be added by aaa server only, from the output as there is no direct interaction of cnBNG UP with AAA.
{: .notice--info}

**Note**: The ACL and QoS policies applied on subscriber interface must be defined on ASR9k (cnBNG UP), prior to subscriber session bring-up.
{: .notice--info}


- Notice that the subscriber subnet route is programmed in RIB along with subscriber route

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP0/CPU0:ASR9k-1#<mark>show route subscriber </mark>
Sun Jan 23 09:06:19.940 IST

<mark>A    20.0.0.0/16 [1/0] via 0.0.0.0, 04:31:06</mark>
A    20.0.0.4/32 is directly connected, 03:44:28, Bundle-Ether1.102.pppoe2147711504

RP/0/RSP0/CPU0:ASR9k-1#<mark>show route ipv6 subscriber </mark>
Sun Jan 23 09:06:25.292 IST

<mark>A    2001::/111 </mark>
      [1/0] via ::, 04:30:20
A    2001::1/128 is directly connected,
      1w2d, Unknown
A    2001::5/128 is directly connected,
      03:36:50, Bundle-Ether1.102.pppoe2147711504
<mark>A    2001:1::/48 </mark>
      [1/0] via ::, 03:36:51
A    2001:1::/64 
      [2/0] 03:36:50, Bundle-Ether1.102.pppoe2147711504
RP/0/RSP0/CPU0:ASR9k-1#
</code>
</pre>
</div>
