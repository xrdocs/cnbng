---
published: true
date: '2021-12-06 19:12 +0530'
title: IPoE DS Subscriber Bringup in cnBNG
author: Gurpreet Dhaliwal
profile: hidden
position: hidden
---
## Introduction
To bringup an IPoE subscriber in cnBNG. We need to setup some initial configurations on both cnBNG CP and cnBNG UP (ASR9k). In this tutorial we will bringup an IPoE DS subscriber session with plan policies and required ACL received during authorization from freeradius.  

## Topology
In this topology we are using CSR1000v as the CPE, cnBNG will assign both IPv4 and IPv6 (NA + PD) prefixes to CPE. The prefix assignment will happen through IPAM which is integral part of cnBNG CP. 

Radius profile configured here is to activate 100mbps plan service with v4 and v6 ACLs applied to dynamic subscriber interface on UP. 

![ipoe-ds-topology1.png]({{site.baseurl}}/images/ipoe-ds-topology1.png)

## Prerequisite:
- cnBNG CP is already deployed and Ops Center is accessible. We will be using Ops Center CLI interface for configurations in this tutorial
- cnBNG CP service network can reach cnBNG UP:
	- In multi VM deployment, this means protocol VM VIP for service network is reachable from cnBNG UP for PFCP communications
    - In single Node/VM AIO deployment, this means AIO VM IP is reachable from cnBNG UP

## Initial cnBNG CP Configurations (Optional)
If you have deployed cnBNG CP a fresh then most probably, initial cnBNG CP configuration is not applied on Ops Center. Follow below steps to apply initial configuratuon to cnBNG CP Ops Center

- SSH login to cnBNG CP Ops Center CLI

```
cisco@pod100-cnbng-cp:~$ ssh admin@10.0.100.1 -p 2024         
Warning: Permanently added '[10.0.100.1]:2024' (RSA) to the list of known hosts.
admin@10.0.100.1's password: 

      Welcome to the bng CLI on pod100/bng
      Copyright © 2016-2020, Cisco Systems, Inc.
      All rights reserved.
    
User admin last logged in 2021-12-01T11:42:45.247257+00:00, to ops-center-bng-bng-ops-center-5666d4cb6-dj7sv, from 10.0.100.1 using cli-ssh
admin connected from 10.0.100.1 using ssh on ops-center-bng-bng-ops-center-5666d4cb6-dj7sv
[pod100/bng] bng# 
```

- Change to config mode in Ops Center

```
[pod100/bng] bng# config
Entering configuration mode terminal
[pod100/bng] bng(config)# 
```

- Apply following initial configuration. With changes to "endpoint radius" and "udp proxy" configs. Both "endpoint radius" and "udp-proxy" should use IP of cnBNG CP service network side protocol VIP or in case of AIO it should be the IP of AIO VM used for peering between cnBNG CP and UP.

```
cdl node-type session
cdl zookeeper replica 1
cdl logging default-log-level error
cdl datastore session
 slice-names [ 1 ]
 endpoint replica 1
 endpoint settings slot-timeout-ms 750
 index replica 1
 index map    1
 index write-factor 1
 slot replica 1
 slot map     1
 slot write-factor 1
 slot notification limit 300
exit
cdl kafka replica 1
instance instance-id 1
 endpoint sm
 exit
 endpoint nodemgr
 exit
 endpoint n4-protocol
  retransmission timeout 0 max-retry 0
 exit
 endpoint dhcp
 exit
 endpoint pppoe
 exit
 endpoint radius
!! Change this IP to your setup based IP
  vip-ip 10.0.100.1
  interface coa-nas
   sla response 140000
!! Change this IP to your setup based IP
   vip-ip 10.0.100.1 vip-port 2000
  exit
 exit
 endpoint udp-proxy
!! Change this IP to your setup based IP
  vip-ip 10.0.100.1
 exit
exit
deployment
 app-name     BNG
 cluster-name pod100
 dc-name      DC
 model        small
exit
k8 bng
 etcd-endpoint      etcd:2379
 datastore-endpoint datastore-ep-session:8882
 tracing
  enable
  enable-trace-percent 30
  append-messages      true
  endpoint             jaeger-collector:9411
 exit
exit
instances instance 1
 system-id  DC
 cluster-id pod100
 slice-name 1
exit
local-instance instance 1
```

- Put system in running mode and commit the changes

```
[pod100/bng] bng(config)# system mode running 
[pod100/bng] bng(config)# commit
Commit complete.
[pod100/bng] bng(config)# 
Message from confd-api-manager at 2021-12-01 12:36:05...
Helm update is STARTING.  Trigger for update is STARTUP. 
[pod100/bng] bng(config)# 
Message from confd-api-manager at 2021-12-01 12:36:08...
Helm update is SUCCESS.  Trigger for update is STARTUP.
```

- Wait for system to deploy all PODs. Verify that all PODs are deployed for cnBNG. Four PODs will be in Init state at this moment, which is ok.

```
cisco@pod100-cnbng-cp:~$ kubectl get pods -n bng-bng
NAME                                                   READY   STATUS     RESTARTS   AGE
bng-dhcp-n0-0                                          0/2     Init:0/1   0          2m45s
bng-n4-protocol-n0-0                                   2/2     Running    0          2m44s
bng-nodemgr-n0-0                                       0/2     Init:0/1   0          2m45s
bng-pppoe-n0-0                                         0/2     Init:0/1   0          2m45s
bng-sm-n0-0                                            0/2     Init:0/1   0          2m45s
cache-pod-0                                            1/1     Running    0          2m44s
cache-pod-1                                            0/1     Running    0          16s
cdl-ep-session-c1-d0-868f578d69-8n2rw                  1/1     Running    0          2m46s
cdl-index-session-c1-m1-0                              1/1     Running    0          2m46s
cdl-slot-session-c1-m1-0                               1/1     Running    0          2m46s
documentation-5857c6db9f-82lg6                         1/1     Running    0          46h
etcd-bng-bng-etcd-cluster-0                            2/2     Running    0          2m45s
etcd-bng-bng-etcd-cluster-1                            2/2     Running    0          2m45s
etcd-bng-bng-etcd-cluster-2                            2/2     Running    0          2m45s
georeplication-pod-0                                   1/1     Running    1          2m44s
grafana-dashboard-app-infra-bng-bng-64b5f7c8b4-rsg5v   1/1     Running    0          2m45s
grafana-dashboard-cdl-bng-bng-66947dbb96-p6c8n         1/1     Running    0          2m46s
grafana-dashboard-cnbng-749978f9cb-txfnd               1/1     Running    0          2m45s
grafana-dashboard-etcd-bng-bng-76b5f7b796-whlch        1/1     Running    0          2m45s
kafka-0                                                2/2     Running    0          2m46s
oam-pod-0                                              2/2     Running    2          2m46s
ops-center-bng-bng-ops-center-5666d4cb6-dj7sv          5/5     Running    0          46h
prometheus-rules-cdl-56457cfd8-tj2d4                   1/1     Running    0          2m46s
prometheus-rules-etcd-5796974fb5-t9w6m                 1/1     Running    0          2m45s
radius-ep-n0-0                                         2/2     Running    1          2m45s
smart-agent-bng-bng-ops-center-54bcbf5576-tc4p9        1/1     Running    1          46h
udp-proxy-0                                            1/1     Running    0          2m45s
zookeeper-0                                            1/1     Running    0          2m46s
```

## cnBNG CP Configuration

cnBNG CP Configuration has following constructs/parts:
- IPAM
- Profile DHCP
- Profile AAA
- Profile Radius
- Profile Feature-Template
- Profile Subscriber
- User-Plane

Let's understand each one in step-by-step and apply in Ops Center in config mode.

### IPAM
This is where we define subscriber address pools for IPv4, IPv6 (NA) and IPv6 (PD). These are the pools from which CPE will get the IPs. IPAM assigns addresses dynamically by splitting address pools into smaller chunks. And then associating each chunk with a user-plane. The pools get freed up dynamically and re-allocated to different user-planes on need basis. 

```
ipam
 instance 1
  source local
  address-pool pool-ISP1
   vrf-name default
   ipv4
    split-size
     per-cache 65536
     per-dp    65536
    exit
    address-range 110.0.0.1 110.10.255.255
   exit
   ipv6
    address-ranges
     split-size
      per-cache 65536
      per-dp    65536
     exit
     address-range f:2::1 f:2::10:ffff
    exit
    prefix-ranges
     split-size
      per-cache 65536
      per-dp    65536
     exit
     prefix-range 2001:db1:: length 48
    exit
   exit
  exit
 exit
exit
```

### Profile DHCP
At present cnBNG only supports DHCP server option. That means cnBNG CP acts as a DHCP server to assign IPs to CPE/subscribers. In profile DHCP we define the DHCP server and which IPAM pool to use by default for subscriber. We can use different pools for IPv4, IPv6 (IANA) and IPv6 (IAPD).

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

### Profile AAA
This profile defines the AAA parameters, like which Radius group to be used authorization and accounting. Also what username value to be used. Here we also define other attrbutes and formats. In this example we will be using MAC based authorization

```
profile aaa aaa_mac
 authorization
  type subscriber method-order [ local ]
  username identifier client-mac-address
  password cisco
 exit
 accounting
  method-order [ local ]
 exit
exit
```

### Profile Radius
Under this profile, Radius groups are created.

```
profile server-group local
 radius-group local
exit

profile radius
 algorithm round-robin
 deadtime  3
 detect-dead-server response-timeout 60
 max-retry 2
 timeout   5
!! Use your radius IP and port for auth here. Here we use 10.0.100.4 as Radius IP with 1812 auth port and 1813 acct port
 server 10.0.100.4 1812
  type   auth
  secret cisco
 exit
 server 10.0.100.4  1813
  type   acct
  secret cisco
 exit
 attribute
  nas-identifier CISCO-cnBNG
!! This should be CP UDP Proxy IP
  nas-ip         10.0.100.1
 exit
 server-group local
  server auth 10.0.100.4 1812
  exit
  server acct 10.0.100.4 1813
  exit
 exit
exit
!! We can also set COA client, use client IP as per the setup
profile coa
 client 10.0.100.4
  server-key cisco
 exit
exit
```

### Profile Feature-template
This profile defines subscriber feature template. This is the template which will be applied to dynamic subscriber interface. We also enable service/ session accounting here.

```
profile feature-template ipoe-1
 vrf-name default
 ipv4
  mtu 1500
 exit
 session-accounting
  enable
  aaa-profile       aaa_mac
  periodic-interval 1800
 exit
exit
```

We can also define service profiles using feature-template, which gets applied on per subscriber session. The service profile in case of radius can be applied during authorization using service activate attribute or it can also be applied using CoA.

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
This profile can be attached on per access port level or per user-plane level. This profile for IPoE defines which dhcp server profile to applu, along with feature-template and profile for aaa auth. 

```
profile subscriber subscriber-profile_ipoe-1
 dhcp-profile               dhcp-server1
 session-type               ipv4v6
 activate-feature-templates [ ipoe-1 ]
 aaa authorize aaa_mac
exit
```

### User-plane
This construct define the association configs. Peering IP as well as subscriber profile to be attached to user-plane or at port level.

```
user-plane ASR9k-1
!! This should be UP IP to which to peer
 peer-address ipv4 10.0.100.2
 subscriber-profile subscriber-profile_ipoe-1
exit
```

## cnBNG UP Configuration

UP Configuration has mainly three constructs and then feature definitions specially QoS and ACL

- Loopback for BNG and association/cnbng-nal Configurations
- DHCP Configuration
- Access Interface

### Loopback Creation
We need to create a Loopback for BNG use.

```
interface Loopback1
 ipv6 enable
```

### Association Configuration

This is where we define association between cnBNG CP and UP. The auto-loopback with secondary-address-upadte enabled will make sure we are able to assign IPs dynamically from IPAM. 

cnbng-nal location 0/RP0/CPU0
 hostidentifier ASR9k-1
 up-server ipv4 10.0.100.2 vrf default
 cp-server primary ipv4 10.0.100.1
 auto-loopback vrf default
  interface Loopback1
!! Any dummy IP
   primary-address 1.1.1.1
  !
 !
!! retry-count specifies how many times UP should retry the connection with CP before declaring CP as dead
 cp-association retry-count 10
 secondary-address-update enable
!

**Note**: NAL stands for Network Adaptation Layer for Cloud Native BNG in IOS-XR
{: .notice--info}

### DHCP Configuration
This is where we associate access interfaces with cnBNG DHCP profile. cnBNG specific DHCP profile makes sure DHCP packets are punted to cnBNG CP through CPRi/GTP-u tunnel.

```
dhcp ipv4
 profile cnbng_v4 cnbng
 !
 interface Bundle-Ether1.101 cnbng profile cnbng_v4

dhcp ipv6
 profile cnbng_v6 cnbng
 !
 interface Bundle-Ether1.101 cnbng profile cnbng_v6
```

### Access Interface Configuration
We define and associate access interface to cnBNG. This way control packets (based on configurations) get routed to the cnBNG CP. The contruct follows ASR9k Integarted BNG model, if you are familiar with.

```
interface Bundle-Ether1.101
 ipv4 point-to-point
 ipv4 unnumbered Loopback1
 ipv6 address 2001::1/64
 ipv6 enable
 load-interval 30
 encapsulation dot1q 101
 ipsubscriber
  ipv4 l2-connected
   initiator dhcp
  !
  ipv6 l2-connected
   initiator dhcp
  !
 !
!
```

## CPE Configs
Following is CSR1000v config which is used as CPE in this example.

```
interface Bundle-Ether1.101
 encapsulation dot1Q 101
 ip address dhcp
 ipv6 address dhcp rapid-commit
 ipv6 address autoconfig
 ipv6 dhcp client pd prefix-from-isp
```

## Radius Profile
Following is Freeradius profile used in this tutorial

```
0050.5688.61ae Cleartext-Password := "cisco"
   Cisco-AVpair += "subscriber:sa=FT_Plan_100mbps",
   Cisco-AVpair += "subscriber:inacl=iACL_BNG_IPv4",
   Cisco-AVpair += "subscriber:ipv6_inacl=iACL_BNG_IPv6"
```

## Verifications

- Verfiy that the cnBNG CP-UP association is up and Active on cnBNG CP

```
[pod100/bng] bng# show peers
GR                                                                                        CONNECTED                                                         INTERFACE  
INSTANCE  ENDPOINT      LOCAL ADDRESS    PEER ADDRESS     DIRECTION  POD INSTANCE   TYPE  TIME       RPC     ADDITIONAL DETAILS                             NAME       
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
0         RadiusServer  -                10.0.100.4:1812  Outbound   radius-ep-0    Udp   3 days     Radius  Status: Active,Type: Auth                      <none>     
0         RadiusServer  -                10.0.100.4:1813  Outbound   radius-ep-0    Udp   3 days     Radius  Status: Active,Type: Acct                      <none>     
1         n4            10.0.100.1:8805  10.0.100.2:8805  Inbound    bng-nodemgr-0  Udp   3 days     UPF     Name: pod100-cnbng-up1,Nm: 0/0,Status: ACTIVE  <none>  
```

- Verify that the CP-UP Association is Up and Active on cnBNG UP

```
RP/0/RP0/CPU0:pod100-cnbng-up1#show cnbng-nal cp connection status location 0/0/CPU0 
Wed Dec  1 13:59:35.533 UTC

Location: 0/0/CPU0

User-Plane configurations:
-------------------------
 IP             : 10.0.100.2        
 GTP Port       : 2152
 PFCP Port      : 8805
 VRF            : default

Control-Plane configurations:
----------------------------
 PRIMARY IP     : 10.0.100.1        
 GTP Port       : 2152
 PFCP Port      : 8805 
 
 Association retry count: 10

 Connection Status: Up
 Connection Status time stamp:  Wed Dec  1 13:59:29 2021

 Connection Prev Status: Down
 Connection Prev Status time stamp:  Wed Dec  1 13:59:28 2021

 Association status: Active
 Association status time stamp: Wed Dec  1 13:59:28 2021
```

- Verify subscriber session is up on cnBNG CP

```
[pod100/bng] bng# show subscriber session 
subscriber-details 
{
  "subResponses": [
    {
      "records": [
        {
          "cdl-keys": [
            "16777220@sm",
            "acct-sess-id:pod3_DC_16777220@sm",
            "upf:pod100-cnbng-up1",
            "port-id:pod100-cnbng-up1/Bundle-Ether1.101",
            "feat-template:ipoe-1",
            "type:sessmgr",
            "mac:0050.5688.61ae",
            "sesstype:ipoe",
            "up-subs-id:pod100-cnbng-up1/1073741888",
            "smupstate:smUpSessionCreated",
            "smstate:established",
            "afi:dual"
          ]
        }
      ]
    }
  ]
}
```

- Verify that the subscriber session is up and working on cnBNG UP

```
RP/0/RP0/CPU0:pod100-cnbng-up1#show cnbng-nal subscriber all  location 0/0/CPU0                         
Mon Dec  6 05:02:13.744 UTC

Location: 0/0/CPU0
Codes: CN - Connecting, CD - Connected, AC - Activated,
       ID - Idle, DN - Disconnecting, IN - Initializing


CPID(hex)  Interface               State  Mac Address     Subscriber IP Addr / Prefix (Vrf) Ifhandle
---------------------------------------------------------------------------------------------------
1000014    BE1.101.ip1073741856 AC     0050.5688.61ae  110.1.0.21 (default) 0x1000030 
                                                          f:2::1:6 (IANA)
                                                          2001:db1:0:7::/64 (IAPD)
Session-count: 1

RP/0/RP0/CPU0:pod100-cnbng-up1#ping 110.1.0.21 source 110.1.0.1
Mon Dec  6 05:02:48.908 UTC
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 110.1.0.21, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/4/12 ms
RP/0/RP0/CPU0:pod100-cnbng-up1#ping f:2::1:6 source BE1.101.ip1073741856
Mon Dec  6 05:03:00.757 UTC
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to f:2::1:6, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/4/13 ms
RP/0/RP0/CPU0:pod100-cnbng-up1#
```

- Notice that the subscriber subnet route is programmed in RIB along with subscriber route

```
RP/0/RP0/CPU0:pod100-cnbng-up1#show route subscriber 
Mon Dec  6 05:01:51.410 UTC

A    110.1.0.0/16 [1/0] via 0.0.0.0, 3d12h
A    110.1.0.21/32 is directly connected, 3d12h, Bundle-Ether1.101.ip107374185

RP/0/RP0/CPU0:pod100-cnbng-up1#show route ipv6 subscriber 
Mon Dec  6 05:01:31.667 UTC

A    f:2::1:0/112 
      [1/0] via ::, 3d12h
A    f:2::1:6/128 is directly connected,
      3d12h, Bundle-Ether1.101.ip1073741856
A    2001:db1::/48 
      [1/0] via ::, 3d12h
A    2001:db1:0:7::/64 
      [2/0] 3d12h, Bundle-Ether1.101.ip1073741856
```

- Notice that the QoS poilicy and ACLs are applied to subscriber interface

```
RP/0/RP0/CPU0:pod100-cnbng-up1#show subscriber running-config interface name BE1.101.ip1073741856
Mon Dec  6 04:59:58.094 UTC
Building configuration...
!! IOS XR Configuration 7.5.1.01I
subscriber-label 0x40000020
dynamic-template
 type user-profile U00000020
  ipv4 access-group iACL_BNG_IPv4 ingress
  ipv6 enable
  ipv4 mtu 1500
  ipv4 unnumbered Loopback1
 !
 type service-profile FT_Plan_100mbps
  service-policy input PM_Plan_100mbps_input
  service-policy output PM_Plan_100mbps_output
 !
!
end
```

- The policy and ACL definitions exist on cnBNG UP

```
RP/0/RP0/CPU0:pod100-cnbng-up1#show running-config policy-map 
Mon Dec  6 05:03:49.479 UTC
policy-map PM_Plan_100mbps_input
 class class-default
  police rate 100 mbps 
   exceed-action drop
  ! 
 ! 
 end-policy-map
! 
policy-map PM_Plan_100mbps_output
 class class-default
  police rate 100 mbps 
   exceed-action drop
  ! 
 ! 
 end-policy-map
! 
RP/0/RP0/CPU0:pod100-cnbng-up1#show running-config ipv4 access-list 
Mon Dec  6 05:04:22.184 UTC
ipv4 access-list iACL_BNG_IPv4
 10 deny tcp any any eq bgp
 20 deny tcp any any eq ldp
 30 deny udp any any eq ldp
 40 permit ipv4 any any
!

RP/0/RP0/CPU0:pod100-cnbng-up1#show running-config ipv6 access-list 
Mon Dec  6 05:04:26.554 UTC
ipv6 access-list iACL_BNG_IPv6
 10 deny tcp any any eq bgp
 20 deny tcp any any eq 646
 30 deny udp any any eq 646
 40 permit ipv6 any any
!
```