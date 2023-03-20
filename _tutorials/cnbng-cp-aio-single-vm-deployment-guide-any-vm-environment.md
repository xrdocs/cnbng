---
published: true
date: '2023-02-28 10:47 +0530'
title: cnBNG CP (AIO) Single VM Deployment Guide - Any VM Environment
position: top
tags:
  - cnbng
  - cse
---
{% include toc %}

cnBNG Control Plane deployment in single VM in any NFVI environment is called as cnBNG CP AIO Manual Deployment. In this deployment cnBNG Control Plane is deployed in a single customized Ubuntu VM. This Ubuntu VM is pre-deployed using SMI base iso image and hence the deployment is called as semi automated or manual. Following are included in this deployment:
- SMI cluster (Cisco CaaS)
- CEE application (Application Infrastructure for Monitoring and Alerting)
- cnBNG Control Plane application (Control Plane application for Cisco CUPS BNG)
    
This is to be noted that only SMI Ubuntu VM deployment in NFVI environment is manual, rest of the process to deploy SMI, CEE and cnBNG Control Plane is fully automated through SMI Deployer or Cluster Manager. 

For AIO deployment following steps are followed:
1. Inception Server Cluster Manager deployment
1. Base ISO Ubuntu VM deployment for cnBNG CP
1. cnBNG CP base ISO Ubuntu OS customizations
1. SMI, CEE and cnBNG CP deployment using SMI Deployer

## Networking

![aio-networking1.png]({{site.baseurl}}/images/aio-networking1.png)

## Prerequisites: 
- Inception Server (SMI Deployer)

## Step 1: Deploying Inception VM and Installing SMI Deployer

Refer to [Inception Server Deployment Guide](https://xrdocs.io/cnbng/tutorials/inception-server-deployment-guide/).

## Step 2: SMI Ubuntu VM Deployment (Manual)

SMI Ubuntu VM can be deployed using any standard VM deployment procedure in a given NFVI environement. This procedure is fairly straight forward and simple. To give an idea on how the deployment of VM works following are the manual steps to deploy the VM in VMWare vCenter. Procedure to deploy the VM may differ based on the chosen NFVI environment. 

1. Download the SMI Base ISO file and copy the file to the VM Datastore
1. In the vCenter, select "Create a New Virtual Machine"
1. Specify name of the VM and select the Datacenter
1. Next select the host for the VM
1. Select the datastore 
1. Select compatibility (ESXi 6.7 or later)
1. Select guest OS: Guest Family- Linux, Guest OS Version- Ubuntu Linux (64-bit)
1. Customize Hardware (sizing and networking may vary on deployments):
	1. vCPU: 8, Memory: 16GB, New Hard disk: 100Gb
	1. Under Network: select management network ("VM Network" in most cases
	1. Click New CD/DVD Drive and do the following:
		1. Select Datastore ISO file option to boot from the SMI Base .iso file. Browse to the location of the .iso file on the datastore set in Step 1.
		1. In the Device Status field, select Connect at power on checkbox.
1. After the VM boots up: login to the VM (user: cloud_user, password: Cisco_123). You will be prompted to change the password immediately
1. Now setup Networking by editing /etc/netplan/50-cloud-init.yaml file. Here is a sample file config:
```
	network:
	    ethernets:
	        eno1:
	            dhcp4: true
	        enp0s3:
	            dhcp4: true
	        ens160:
	            dhcp4: false
	            addresses:
	               - 192.168.107.166/25
	            gateway4: 192.168.107.129
	            nameservers:
	                search: [cisco.com]
	                addresses: [72.163.128.140]
	        ens3:
	            dhcp4: true
	        eth0:
	            dhcp4: true
	    version: 2
```

**Note**: Sometimes interface is not shown as ens160, in that case it is a good idea to search for the interface using ifconfig -a command. Generally lower ens number is the first NIC attached to the VM, and higher number is the last.
{: .notice--info}

## Step 3: Base ISO Ubuntu OS customization

1. SSH login to cnBNG CP AIO Ubuntu VM which was deployed in Step-2
1. Now change the hostname of the VM to: <your-cnbng-cp-cluster>-aio, using:
```
sudo hostnamectl set-hostname <your-cnbng-cp-cluster>-aio
e.g.
sudo hostnamectl set-hostname cnbng-cp-lab1-aio
```
1. Logout of the VM and login again to see hostname changes are reflected
1. Make the hostname persistent even after reload by adding "preserve_hostname: true" to /etc/cloud/cloud.cfg file if not added already or change the setting to true from false if already present
1. (optional) Replace default hostname for VM with the one you set into /etc/hosts file
1. Verify that the hostname is persistent even after reboot of the VM
1. SSH Key Generation
	1. SSH Login to Inception VM
	1. Generate SSH key using: 
```
	ssh-keygen -t rsa
```
	1. Run ssh-copy-id command, which will copy ssh keys to Base ISO Ubuntu Image for cnBNG CP AIO e.g.
```
	ssh-copy-id cloud-user@192.168.107.166
```
	1. Verify that the login using keys is working by logging to cnBNG CP AIO VM
```
	ssh cloud-user@192.168.107.166
```

## Step 4: SMI, CEE and cnBNG CP deployment using SMI Deployer

1. Login to inception VM
1. Note down ssh keys in a file from .ssh/id_rsa (private) and ./ssh/id_rsa.pub (public)
1. Remove line breaks from private key and replace them with string "\n"
1. Login to SMI Cluster Deployer or Cluster Manager on inception VM:
	```
	ssh admin@localhost -p 2022
	```
1. Create environment configs:
	```
	environments manual
		manual
	exit
	```
1. Create Cluster configs:
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    clusters cnbng
     environment manual
     addons ingress bind-ip-address 192.168.107.166
     addons ingress enabled
     addons istio enabled
     configuration master-virtual-ip 192.168.107.166
     configuration master-virtual-ip-interface ens160
     configuration pod-subnet    192.202.0.0/16
     configuration allow-insecure-registry true
     node-defaults initial-boot default-user cisco
     node-defaults initial-boot default-user-ssh-public-key <mark>"ssh public key from step-2"</mark>
     node-defaults k8s ssh-username cloud-user
     node-defaults k8s ssh-connection-private-key "-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEAtuZ56pW6nMf0e8ZMAjsKnj89YIpEfVyZ31GkrqObUua70egN\nOqDdWF8ToUHQrPnwDP/E4BplhtdiP2n/Dq8e9Xf8GRHX6KNXC6dhl5c6MKaBoh5A\nRDj8aVEfRfoDwpp5L0clg01EP0IIqkTIrSqScAheNfiKSptE5OleDS4I4mxfZsDR\nKnhAOtjgU/aHNGFDt1rYJ+QifAcp97ouyiNfsUsJwifLwBN3/EisbTH3rD0Z5fUO\n1UYmOLFW3vEI8voXAHk4BnNad6QGACCfENmCVZW3RBovr5wSdc22XXDTwp1gzJN3\nqlDcgQ6JfvE8i2JyL44Fb8KiZVvDgongXj9RGwIDAQABAoIBAB4Tjm7WCmbntrt3\n413miZt2OMicVCDtTlxb16HkQ5GBYddlum8urtduYxL8eK1JOIFauexESve+iWh2\nLLwkbgndnjYdKg0WdyTydGjyNF51sxGOufC+EjvbXDIsp9ujfVQZ9gA+f3+Lg1NE\nll9rhcMojR2A7nTQTab6/T1bmZhqBKcsX0DEFiv2o64/QeXSijzXMTLH1jASJKC7\ns+xyu3AmwE3LrecQah8xe+zZSefFWcPCQHNRWflVGUnKWFk58qbdv2X/MSzLvkYu\nU2A0RJX/U4ZOIWM1J8RUPv4MKFEgJ7LA1OT3UtOC+S6Bemlx6h/VWhoJnNhcwvm9\nd5vJoDECgYEA2cjmN0U/eeGlFZDLGRtQ4HVoUkQFMosHk1KQhEj3NZzIZnI9/VFf\njBkfFlP0bFajKZgCwwctb6EcJaRY6JsnKUuTHT5ZikVddFVU1hJkU/GwwHOX9cCM\nFuJi+yxc1G27pE82xWaUuiVRJ8wbQOkPMWrrvaR8eyUa6X8CRk3+JBkCgYEA1v6I\nhRtrxtiZ32hVr33X08vg2VDazgXAdOrBRimxkjjBPlHWekDZePkxKT+fTIzlLffQ\nIemn/gFJo4+YhuBzmv+k5I6Bo6I1M/VSoAFXKShwghg7KSfAQ5BH8oIr9PG4XEl4\nWs5cGb1NrCgqmQISLS1Cn5q6H+NDvdpgckryJVMCgYAQyBRFSga8I5EO+ltMEfjH\ncwSY4jjsTh5FUeVk7CJwdSZUDpWMQYr1RrJIjCuXdY2ZFOeRk6oCog2DMQjQ07PO\n0M4DQNyxdOrgnfqtjDlC5qrSCZY6D5473TH3XNHCZLpCzP/Rcjgfp+R7BpVLCSps\nimqj8FrPOmq6d1j7heMBcQKBgE+ltj/Rm8jrv32DcpL0BPwCwMbhbF38xYLK4VUz\n5wProLOMr+9UjPyDHNJSLpq2a8Tu1J1rqX+xTG2aqf/1sP5QDO9bV+2eDyWzkauT\nM44c3Cll/qzNfC3Lisvtq4kv74PI+Bxz7Kzgc6D+tGFA4ij4ZoEoWiGsGRGBkE9n\nMnPfAoGBAK9q6auE3HXkZ+VVD5LH2td7oze+7o1vrs2O/175oTzf/drcY5S59lK4\nudOjDIXNA/Lot080YONX7lbe2Eept21YektS9xpZ1Qld8NGoPFYsrYjqGYbojFbN\nYJMSEpOpbber+Mca7NenEusL5hK87sQCOP6OJ2RwjkGUVlraVpgQ\n-----END RSA PRIVATE KEY-----"

     node-defaults netplan template "    network:\n        version: 2\n        ethernets:\n            ens160:\n                dhcp4: false\n                addresses:\n                - {{K8S_SSH_IP}}/24\n                routes:\n                - to: 0.0.0.0/0\n                  via: 192.168.107.129\n                nameservers:\n                    addresses:\n                    - 64.104.128.236\n                    - 64.102.6.247\n                    search:\n                    - cisco.com\n\n\n"

     node-defaults os ntp enabled
     node-defaults os ntp servers <mark>clock.cisco.com</mark>
    exit
    </code>
    </pre>
    </div>

In the above config, change ntp server to the one available in the lab. Also netplan should be as per the netplan configured in the VM. 
{: .notice--info}
  
1. Create AIO Node config:
  <div class="highlighter-rouge">
  <pre class="highlight">
  <code>
  clusters cnbng
   nodes cp
    k8s node-type master
    k8s ssh-ip 192.168.107.166
    k8s node-labels disktype ssd
    exit
    k8s node-labels smi.cisco.com/node-type oam
    exit
    initial-boot default-user cloud-user
    initial-boot default-user-password <mark>your-password</mark>
   exit
  exit
  </code>
  </pre>
  </div>
  
1. cnBNG software is available as a tarball and it can be hosted on local http server for offline deployment. In this step we configure the software repository locations for tarball. We setup software cnf for both cnBNG CP and CEE. URL and SHA256 depends on the version of the image and the url location, so these two could change for your deployment
  ```
  software cnf bng
   url             http://192.168.107.148/images/CP/cp_30sep21/bng/bng.2021.04.m0.i74.tar
   allow-dev-image true
   sha256          e36b5ff86f35508539a8c8c8614ea227e67f97cf94830a8cee44fe0d2234dc1c
   description     bng-products
  exit
  software cnf cee
   url         http://192.168.107.148/images/CP/cp_30sep21/cee-2020.02.6.i04.tar
   sha256      b5040e9ad711ef743168bf382317a89e47138163765642c432eb5b80d7881f57
   description cee-products
  exit
  ```
1. Setup Ops Center configs inside cluster for cnBNG
  <div class="highlighter-rouge">
  <pre class="highlight">
  <code>
  clusters <mark>your-cnbng-cp-cluster-name</mark>
   ops-centers bng bng
    repository-local        bng
    sync-default-repository true
    netconf-ip              <mark>your-cnbng-cp-vm-ip</mark>
    netconf-port            3022
    ssh-ip                  <mark>your-cnbng-cp-vm-ip</mark>
    ssh-port                2024
    ingress-hostname        <mark>your-cnbng-cp-vm-ip</mark>.nip.io
    initial-boot-parameters use-volume-claims true
    initial-boot-parameters first-boot-password <mark>your-password1</mark>
    initial-boot-parameters auto-deploy false
    initial-boot-parameters single-node true
   exit
   ops-centers cee global
    repository-local        cee
    sync-default-repository true
    netconf-ip              <mark>your-cnbng-cp-vm-ip</mark>
    netconf-port            3024
    ssh-ip                  <mark>your-cnbng-cp-vm-ip</mark>
    ssh-port                2023
    ingress-hostname        <mark>your-cnbng-cp-vm-ip</mark>.nip.io
    initial-boot-parameters use-volume-claims true
    initial-boot-parameters first-boot-password <mark>your-password1</mark>
    initial-boot-parameters auto-deploy true
    initial-boot-parameters single-node true
   exit
  exit
  </code>
  </pre>
  </div>
  
1. Deploy cnBNG CP cluster using cluster sync command:
  ```
  clusters cnbng-cp-lab1 actions sync run debug true 
  ```	
Monitor sync using monitor command:
  ```
  monitor sync-logs cnbng-cp-lab1
  ```
  
## Verifications

- Check kubernetes cluster is deployed correctly and no PODs are crashing

```
cisco@pod1-cnbng-cp:~$ kubectl get pods -A
NAMESPACE           NAME                                                 READY   STATUS    RESTARTS   AGE
bng-bng             documentation-95c8f45d9-8g4qd                        1/1     Running   0          5m
bng-bng             ops-center-bng-bng-ops-center-77fb6479fc-dtvt2       5/5     Running   0          5m
bng-bng             smart-agent-bng-bng-ops-center-8d9fffbfb-5cdqm       1/1     Running   1          5m
cee-global          alert-logger-7c6d5b6596-jrrx6                        1/1     Running   0          5m
cee-global          alert-router-549c8fb66c-4shz6                        1/1     Running   0          5m
cee-global          alertmanager-0                                       1/1     Running   0          5m
cee-global          blackbox-exporter-n9mzx                              1/1     Running   0          5m
cee-global          bulk-stats-0                                         3/3     Running   0          5m
cee-global          cee-global-product-documentation-864c8fb66b-rm44g    2/2     Running   0          5m
cee-global          core-retriever-bntcb                                 2/2     Running   0          5m
cee-global          documentation-684cbb8cbc-95qvj                       1/1     Running   0          5m
cee-global          grafana-6f54c8cc5f-xpvj5                             6/6     Running   0          5m
cee-global          grafana-dashboard-metrics-7764c5f8f4-kgthj           1/1     Running   0          5m
cee-global          kube-state-metrics-79bdbd9db7-jmxxx                  1/1     Running   0          5m
cee-global          logs-retriever-b7l5v                                 1/1     Running   0          5m
cee-global          node-exporter-rnhwr                                  1/1     Running   0          5m
cee-global          ops-center-cee-global-ops-center-5bbdb84597-8rtk5    5/5     Running   0          5m
cee-global          path-provisioner-l89w6                               1/1     Running   0          5m
cee-global          pgpool-859f9d7d89-mtks5                              1/1     Running   0          5m
cee-global          pgpool-859f9d7d89-rwsmk                              1/1     Running   0          5m
cee-global          postgres-0                                           1/1     Running   0          5m
cee-global          postgres-1                                           1/1     Running   0          5m
cee-global          postgres-2                                           1/1     Running   0          5m
cee-global          prometheus-hi-res-0                                  4/4     Running   0          5m
cee-global          prometheus-rules-65ccd95b6c-29n6j                    1/1     Running   0          5m
cee-global          prometheus-scrapeconfigs-synch-85fdc7ccbf-jd54g      1/1     Running   0          5m
cee-global          pv-manager-9449fc649-h97f4                           1/1     Running   0          5m
cee-global          pv-provisioner-6775997cc5-hjzf6                      1/1     Running   0          5m
cee-global          restart-kubelet-75f7d                                1/1     Running   0          5m
cee-global          show-tac-manager-5cbb8d589b-rpzv9                    2/2     Running   0          5m
cee-global          smart-agent-cee-global-ops-center-69f7b8d9d5-ptrct   1/1     Running   1          5m
cee-global          thanos-query-frontend-hi-res-54bbbbbcb6-phbcq        1/1     Running   0          5m
cee-global          thanos-query-hi-res-6776585f99-bfzx9                 2/2     Running   0          5m
istio-system        istiod-cf97f695b-6qqzp                               1/1     Running   0          5m
kube-system         calico-kube-controllers-5c9bfb6cf-jnrc8              1/1     Running   0          5m
kube-system         calico-node-ptnjs                                    1/1     Running   0          5m
kube-system         cluster-cert-maintainer-7f879cf5cf-88xkn             1/1     Running   0          5m
kube-system         coredns-558bd4d5db-kkhd2                             1/1     Running   0          5m
kube-system         coredns-558bd4d5db-q5lct                             1/1     Running   0          5m
kube-system         etcd-pod2-cnbng-cp                                   1/1     Running   0          5m
kube-system         kube-apiserver-pod2-cnbng-cp                         1/1     Running   0          5m
kube-system         kube-controller-manager-pod2-cnbng-cp                1/1     Running   0          5m
kube-system         kube-proxy-lp68c                                     1/1     Running   0          5m
kube-system         kube-scheduler-pod2-cnbng-cp                         1/1     Running   0          5m
kube-system         maintainer-sgrs7                                     1/1     Running   0          5m
nginx-ingress       nginx-ingress-controller-b5857775-slgkh              1/1     Running   0          5m
nginx-ingress       nginx-ingress-default-backend-86496666db-5jgcz       1/1     Running   0          5m
registry            charts-bng-2021-04-m0-i74-0                          1/1     Running   0          5m
registry            charts-cee-2020-02-6-i04-0                           1/1     Running   0          5m
registry            registry-bng-2021-04-m0-i74-0                        1/1     Running   0          5m
registry            registry-cee-2020-02-6-i04-0                         1/1     Running   0          5m
registry            software-unpacker-0                                  1/1     Running   0          5m
smi-certs           ss-cert-provisioner-5d764d5667-27d55                 1/1     Running   0          5m
smi-secure-access   secure-access-controller-k62z5                       1/1     Running   0          5m
smi-vips            keepalived-5l789                                     3/3     Running   0          5m
```

- Check Grafana ingress and try logging to it (username: admin, password: <<your password as per CEE Ops Center config>>)
  
<div class="highlighter-rouge">
<pre class="highlight">
<code>
cisco@pod1-cnbng-cp:~$ kubectl get ingress -A
NAMESPACE    NAME                                       CLASS    HOSTS                                                          ADDRESS           PORTS     AGE
bng-bng      cli-ingress-bng-bng-ops-center             &lt;none&gt;   cli.bng-bng-ops-center.192.168.107.150.nip.io                  192.168.107.150   80, 443   5m
bng-bng      documentation-ingress                      &lt;none&gt;   documentation.bng-bng-ops-center.192.168.107.150.nip.io        192.168.107.150   80, 443   5m
bng-bng      restconf-ingress-bng-bng-ops-center        &lt;none&gt;   restconf.bng-bng-ops-center.192.168.107.150.nip.io             192.168.107.150   80, 443   5m
cee-global   cee-global-product-documentation-ingress   &lt;none&gt;   docs.cee-global-product-documentation.192.168.107.150.nip.io   192.168.107.150   80, 443   5m
cee-global   cli-ingress-cee-global-ops-center          &lt;none&gt;   cli.cee-global-ops-center.192.168.107.150.nip.io               192.168.107.150   80, 443   5m
cee-global   documentation-ingress                      &lt;none&gt;   documentation.cee-global-ops-center.192.168.107.150.nip.io     192.168.107.150   80, 443   5m
cee-global   grafana-ingress                            &lt;none&gt;   <mark>grafana.192.168.107.150.nip.io</mark>                                 192.168.107.150   80, 443   5m
cee-global   prometheus-hi-res                          &lt;none&gt;   prometheus-hi-res.192.168.107.150.nip.io                       192.168.107.150   80, 443   5m
cee-global   restconf-ingress-cee-global-ops-center     &lt;none&gt;   restconf.cee-global-ops-center.192.168.107.150.nip.io          192.168.107.150   80, 443   5m
cee-global   show-tac-manager-ingress                   &lt;none&gt;   show-tac-manager.192.168.107.150.nip.io                        192.168.107.150   80, 443   5m
registry     charts-ingress                             &lt;none&gt;   charts.192.168.107.150.nip.io                                  192.168.107.150   80, 443   5m
registry     registry-ingress                           &lt;none&gt;   docker.192.168.107.150.nip.io                                  192.168.107.150   80, 443   5m
</code>
</pre>
</div>

We can login to Grafana GUI from Chrome/ Any browser @URL: https://grafana.your-cnbng-cp-vm-ip.nip.io/

![grafana-login1.png]({{site.baseurl}}/images/grafana-login1.png)

- SSH to cnBNG CP Ops Center at port 2024
  
```
cloud-user@inception:~$ ssh admin@192.168.107.150 -p 2024
admin@192.168.107.150's password: 

      Welcome to the bng CLI on pod2/bng
      Copyright © 2016-2020, Cisco Systems, Inc.
      All rights reserved.
    
User admin last logged in 2021-11-19T04:12:32.912093+00:00, to ops-center-bng-bng-ops-center-77fb6479fc-dtvt2, from 192.168.107.150 using cli-ssh
admin connected from 192.168.107.150 using ssh on ops-center-bng-bng-ops-center-77fb6479fc-dtvt2
[pod1/bng] bng# 
[pod1/bng] bng# 
```
  
- We can also test Netconf Interface availability of cnBNG Ops Center using ssh
  
```
cloud-user@inception:~$ ssh admin@192.168.107.150 -p 3022 -s netconf    
Warning: Permanently added '[192.168.107.150]:3022' (RSA) to the list of known hosts.
admin@192.168.107.150's password: 
<?xml version="1.0" encoding="UTF-8"?>
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<capabilities>
<capability>urn:ietf:params:netconf:base:1.0</capability>
<capability>urn:ietf:params:netconf:base:1.1</capability>
<capability>urn:ietf:params:netconf:capability:confirmed-commit:1.1</capability>
<capability>urn:ietf:params:netconf:capability:confirmed-commit:1.0</capability>
<capability>urn:ietf:params:netconf:capability:candidate:1.0</capability>
<capability>urn:ietf:params:netconf:capability:rollback-on-error:1.0</capability>
<capability>urn:ietf:params:netconf:capability:url:1.0?scheme=ftp,sftp,file</capability>
<capability>urn:ietf:params:netconf:capability:validate:1.0</capability>
<capability>urn:ietf:params:netconf:capability:validate:1.1</capability>
<capability>urn:ietf:params:netconf:capability:xpath:1.0</capability>
<capability>urn:ietf:params:netconf:capability:notification:1.0</capability>
<capability>urn:ietf:params:netconf:capability:interleave:1.0</capability>
<capability>urn:ietf:params:netconf:capability:partial-lock:1.0</capability>
<capability>urn:ietf:params:netconf:capability:with-defaults:1.0?basic-mode=explicit&amp;also-supported=report-all-tagged,report-all</capability>
<capability>urn:ietf:params:netconf:capability:yang-library:1.0?revision=2019-01-04&amp;module-set-id=a6803bdbd5c766b47137ad86700fff0a</capability>
<capability>urn:ietf:params:netconf:capability:yang-library:1.1?revision=2019-01-04&amp;content-id=a6803bdbd5c766b47137ad86700fff0a</capability>
<capability>http://tail-f.com/ns/netconf/actions/1.0</capability>
<capability>http://cisco.com/cisco-bng-ipam?module=cisco-bng-ipam&amp;revision=2020-01-24</capability>
<capability>http://cisco.com/cisco-cn-ipam?module=cisco-cn-ipam&amp;revision=2021-02-21</capability>
<capability>http://cisco.com/cisco-exec-ipam?module=cisco-exec-ipam&amp;revision=2021-06-01</capability>
<capability>http://cisco.com/cisco-mobile-nf-tls?module=cisco-mobile-nf-tls&amp;revision=2020-06-24</capability>
<capability>http://cisco.com/cisco-smi-etcd?module=cisco-smi-etcd&amp;revision=2021-09-15</capability>
<capability>http://tail-f.com/cisco-mobile-common?module=tailf-mobile-common&amp;revision=2019-04-25</capability>
<capability>http://tail-f.com/cisco-mobile-product?module=tailf-cisco-mobile-product&amp;revision=2018-06-06</capability>
<capability>http://tail-f.com/ns/aaa/1.1?module=tailf-aaa&amp;revision=2018-09-12</capability>
<capability>http://tail-f.com/ns/common/query?module=tailf-common-query&amp;revision=2017-12-15</capability>
<capability>http://tail-f.com/ns/confd-progress?module=tailf-confd-progress&amp;revision=2020-06-29</capability>
<capability>http://tail-f.com/ns/kicker?module=tailf-kicker&amp;revision=2020-11-26</capability>
<capability>http://tail-f.com/ns/netconf/query?module=tailf-netconf-query&amp;revision=2017-01-06</capability>
<capability>http://tail-f.com/ns/webui?module=tailf-webui&amp;revision=2013-03-07</capability>
<capability>http://tail-f.com/yang/acm?module=tailf-acm&amp;revision=2013-03-07</capability>
<capability>http://tail-f.com/yang/common?module=tailf-common&amp;revision=2020-11-26</capability>
<capability>http://tail-f.com/yang/common-monitoring?module=tailf-common-monitoring&amp;revision=2019-04-09</capability>
<capability>http://tail-f.com/yang/confd-monitoring?module=tailf-confd-monitoring&amp;revision=2019-10-30</capability>
<capability>http://tail-f.com/yang/last-login?module=tailf-last-login&amp;revision=2019-11-21</capability>
<capability>http://tail-f.com/yang/netconf-monitoring?module=tailf-netconf-monitoring&amp;revision=2019-03-28</capability>
<capability>http://tail-f.com/yang/xsd-types?module=tailf-xsd-types&amp;revision=2017-11-20</capability>
<capability>urn:ietf:params:xml:ns:netconf:base:1.0?module=ietf-netconf&amp;revision=2011-06-01&amp;features=confirmed-commit,candidate,rollback-on-error,validate,xpath,url</capability>
<capability>urn:ietf:params:xml:ns:netconf:partial-lock:1.0?module=ietf-netconf-partial-lock&amp;revision=2009-10-19</capability>
<capability>urn:ietf:params:xml:ns:yang:iana-crypt-hash?module=iana-crypt-hash&amp;revision=2014-08-06&amp;features=crypt-hash-sha-512,crypt-hash-sha-256,crypt-hash-md5</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-inet-types?module=ietf-inet-types&amp;revision=2013-07-15</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-netconf-acm?module=ietf-netconf-acm&amp;revision=2018-02-14</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring?module=ietf-netconf-monitoring&amp;revision=2010-10-04</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-netconf-notifications?module=ietf-netconf-notifications&amp;revision=2012-02-06</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-netconf-with-defaults?module=ietf-netconf-with-defaults&amp;revision=2011-06-01</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-restconf-monitoring?module=ietf-restconf-monitoring&amp;revision=2017-01-26</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-x509-cert-to-name?module=ietf-x509-cert-to-name&amp;revision=2014-12-10</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-yang-metadata?module=ietf-yang-metadata&amp;revision=2016-08-05</capability>
<capability>urn:ietf:params:xml:ns:yang:ietf-yang-types?module=ietf-yang-types&amp;revision=2013-07-15</capability>
</capabilities>
<session-id>171</session-id></hello>]]>]]>
```
  
## Initial cnBNG CP Configurations
If you have deployed cnBNG CP a fresh then most probably, initial cnBNG CP configuration is not applied on Ops Center. Follow below steps to apply initial configuratuon to cnBNG CP Ops Center

- SSH login to cnBNG CP Ops Center CLI

```
cisco@pod100-cnbng-cp:~$ ssh admin@192.168.107.150 -p 2024         
Warning: Permanently added '[192.168.107.150]:2024' (RSA) to the list of known hosts.
admin@192.168.107.150's password: 

      Welcome to the bng CLI on pod100/bng
      Copyright © 2016-2020, Cisco Systems, Inc.
      All rights reserved.
    
User admin last logged in 2021-12-01T11:42:45.247257+00:00, to ops-center-bng-bng-ops-center-5666d4cb6-dj7sv, from 192.168.107.150 using cli-ssh
admin connected from 192.168.107.150 using ssh on ops-center-bng-bng-ops-center-5666d4cb6-dj7sv
[pod100/bng] bng# 
```

- Change to config mode in Ops Center

```
[pod100/bng] bng# config
Entering configuration mode terminal
[pod100/bng] bng(config)# 
```

- Apply following initial configuration. With changes to "endpoint radius" and "udp proxy" configs. Both "endpoint radius" and "udp-proxy" should use IP of cnBNG CP service network side protocol VIP or in case of AIO it should be the IP of AIO VM used for peering between cnBNG CP and UP.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
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
!! Change this IP to your AIO VM IP
   <mark>vip-ip 192.168.107.150</mark>
  interface coa-nas
   sla response 140000
!! Change this IP to your AIO VM IP
   <mark>vip-ip 192.168.107.150 vip-port 2000</mark>
  exit
 exit
 endpoint udp-proxy
!! Change this IP to your AIO VM IP
  <mark>vip-ip 192.168.107.150</mark>
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
</code>
</pre>
</div>

- Put system in running mode and commit the changes

<div class="highlighter-rouge">
<pre class="highlight">
<code>
[pod100/bng] bng(config)# <mark>system mode running </mark>   
[pod100/bng] bng(config)# commit
Commit complete.
[pod100/bng] bng(config)# 
Message from confd-api-manager at 2021-12-01 12:36:05...
Helm update is STARTING.  Trigger for update is STARTUP. 
[pod100/bng] bng(config)# 
Message from confd-api-manager at 2021-12-01 12:36:08...
Helm update is SUCCESS.  Trigger for update is STARTUP.
</code>
</pre>
</div>

- Wait for system to deploy all PODs. Verify that all PODs are deployed for cnBNG. Four PODs will be in <span style="background-color: #FDD7E4">Init</span> state at this moment, which is ok.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
cisco@pod100-cnbng-cp:~$ kubectl get pods -n bng-bng
NAME                                                   READY   STATUS     RESTARTS   AGE
bng-dhcp-n0-0                                          0/2     <span style="background-color: #FDD7E4">Init:0/1</span>   0          2m45s
bng-n4-protocol-n0-0                                   2/2     Running    0          2m44s
bng-nodemgr-n0-0                                       0/2     <span style="background-color: #FDD7E4">Init:0/1</span>   0          2m45s
bng-pppoe-n0-0                                         0/2     <span style="background-color: #FDD7E4">Init:0/1</span>   0          2m45s
bng-sm-n0-0                                            0/2     <span style="background-color: #FDD7E4">Init:0/1</span>   0          2m45s
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
</code>
