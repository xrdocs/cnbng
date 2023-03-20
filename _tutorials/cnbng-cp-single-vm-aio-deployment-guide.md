---
published: true
date: '2022-1-06 18:19 +0530'
title: cnBNG CP Single VM (AIO) Deployment Guide - VMWare
author: Gurpreet Dhaliwal
position: top
tags:
  - cisco
  - cnbng
  - cloud native bng
  - broadband network gateway
  - bng
excerpt: >-
  cnBNG Control Plane can be deployed in Single VM. This single VM deployment of
  cnBNG is called as All-in-one (AIO) deployment. AIO deployment includes
  ops-center, telemetry, grafana with full cloud native stack from SMI and cnBNG
  control plane application software. In this tutorial lean how to deploy cnBNG
  CP in Single VM.
---
{% include toc %}

## Introduction
AIO VMware Deployment means All-In-One deployment of cnBNG Control Plane in VMWare ESXi environment. This is a single VM deployment for cnBNG Control Plane in VMWare environment which deploys following in a single customized base Ubuntu VM:

- SMI cluster: Cloud Native Infra including K8s, docker, keepalived, calico, istio etc.
- CEE application: Common Execution Environment for telemetry, snmp, bulkstats, alert etc.
- cnBNG Control Plane application: Core BNG Control Plane application

The deployment is done in 3 steps, where the 3rd step uses automation provided by Inception Server to deploy cnBNG CP in All-In-One form. The main steps for deployment are:

1. Inception Server/SMI Cluster Deployer deployment using vCenter
1. SSH Key Generation for security purposes
1. cnBNG CP Deployment using Inception Server, which includes:
	- Customized Base OS Ubuntu VM Deployment (automated for ESXi)
	- SMI Cluster Deployment, which includes CN Infra (automated)
	- CEE and cnBNG CP Application deployment (automated)
    
After the deployment, we generally apply the initial init-config for cnBNG CP Ops Center and move the system to running mode. In this tutorial this step is explained towards the end.

## Networking

![aio-networking1.png]({{site.baseurl}}/images/aio-networking1.png)


## Prerequisite (optional, if DC already exists)
1. Make sure vCenter is installed (ver 6.7 is tested)
1. In the vCenter, right click and select New Data Center
	- Provide name of the data-center and click ok
1. Right click on the newly created datacenter and select New Cluster.
	- Provide name of the cluster and click ok (all other options remain default)
![vmware-dc1.png]({{site.baseurl}}/images/vmware-dc1.png)
1. Now add a host to the cluster. By selecting add host from right click menu on newly created cluster. Follow on screen instructions to add the host.

## Step 1: Inception CM Deployment
Refer to [Inception Server Deployment Guide](https://xrdocs.io/cnbng/tutorials/inception-server-deployment-guide/).

## Step 2: SSH Key Generation
This is for security reasons, where cnBNG CP VM can only be accessed by ssh key and not through password by default.

- SSH Login to Inception VM
- Generate SSH key using:

```
ssh-keygen -t rsa
```
- Note down ssh keys in a file from .ssh/id_rsa (private) and ./ssh/id_rsa.pub (public)
- Remove line breaks from private key and replace them with string "\n"
- Remove line breaks from public key if there are any.

**Note**: You can use following commands to replace line breaks:
sed -z 's/\n/\\n/g' .ssh/id_rsa
sed -z 's/\n/\\n/g' .ssh/id_rsa.pub
{: .notice--info}

## Step 3: cnBNG CP deployment using Inception CM
- Login to SMI Cluster Deployer or Cluster Manager on inception VM:

```
cloud-user@inception:~$ ssh admin@localhost -p 2022
admin@localhost's password: 

      Welcome to the Cisco SMI Cluster Deployer on inception
      Copyright © 2016-2020, Cisco Systems, Inc.
      All rights reserved.
    
admin connected from 172.18.0.1 using ssh on 79bd273aeec7
[inception] SMI Cluster Deployer# 
[inception] SMI Cluster Deployer#
```

- In the next few steps we will be setting up configuration in SMI Cluster Deployer. We will be using config mode and after adding all the configs we will be using commit.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
[inception] SMI Cluster Deployer# config
Entering configuration mode terminal
[inception] SMI Cluster Deployer(config)# <mark>your-configs-to-apply</mark>
[inception] SMI Cluster Deployer(config)# commit
[inception] SMI Cluster Deployer(config)# end
</code>
</pre>
</div>

- Create environment configs. This config is for inception deployer to get access to vCenter.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
environments vmware
 vcenter server         <mark>your-vcenter-ip</mark>
 vcenter allow-self-signed-cert true
 vcenter user           <mark>your-vcenter-username</mark>
 vcenter password       <mark>your-vcenter-password</mark>
 vcenter datastore      <mark>your-vcenter-host-local-datastore</mark>
 vcenter cluster        cnBNG_SMI_CL01
 vcenter datacenter     cnBNG_DC_01
 vcenter host           <mark>your-vcenter-host</mark>
 vcenter nics "VM Network"
 exit
exit
</code>
</pre>
</div>

- Define Netplan for cnBNG CP Ubuntu VM:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
 network:
    version: 2
    ethernets:
        ens192:
            dhcp4: false
            addresses:
            - { {K8S_SSH_IP}}/24
            routes:
            - to: 0.0.0.0/0
              via: <mark>your-gateway-ip</mark>
            nameservers:
                addresses:
                - <mark>your-dns1</mark>
                - <mark>your-dns2</mark>
                search:
                - <mark>your-domain</mark>
</code>
</pre>
</div>

- Replace line breaks by "\n". Netplan will look like this:

```
 "\n     network:\n        version: 2\n        ethernets:\n            ens192:\n                dhcp4: false\n                addresses:\n                - { {K8S_SSH_IP}}/24\n                routes:\n                - to: 0.0.0.0/0\n                  via: your-gateway-ip\n                nameservers:\n                    addresses:\n                    - your-dns1\n                    - your-dns2\n                    search:\n                    - your-domain\n\n\n"
```

**Note**: You can use following command to replace line breaks:\n
sed -z 's/\n/\\n/g' netplan.yaml
{: .notice--info}

Don't forget to remove spaces in { { K8S_SSH_IP } } in the configs above.
{: .notice--info}

- Create Cluster configs with SSH public and private keys copied in Step-2 and netplan from previous step:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
clusters <mark>your-cnbng-cp-cluster-name</mark>
 environment vmware
 addons ingress bind-ip-address <mark>your-cnbng-cp-vm-ip</mark>
 addons ingress enabled
 addons istio enabled
 configuration master-virtual-ip <mark>your-cnbng-cp-vm-ip</mark>
 configuration master-virtual-ip-interface ens192
 configuration pod-subnet    192.202.0.0/16
 configuration allow-insecure-registry true
 node-defaults initial-boot default-user <mark>your-cnbng-cp-vm-login-name</mark>
 node-defaults initial-boot default-user-ssh-public-key "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDDH4uQdrxTvgFg32szcxLUfwrxmI513ySoDkjznncofvkL/JV122JAsGHvL6T39jBRKytuPpLhvr9ZvwKlOZ5BeeUB+Zj8tuZdQRqIpvonhtcESc11oRM6w5dQjHUIaDdK/vBQQg5hSQ0CVzvVU2DS8E+csMbBoPzaUVi+qPjqfBEt9MzMysc1DnnXN1fFMDVzTAsb5mgt/cLjcfbo6qVgLt9irFJ/AfoQGltQi9M4vehsprYZngGL5U9+qt9qn4lbaxTt2HqNQJAuk701LVhZNXKCfHmgg6en3IYk7YZI1IhhAnYAqlQLuC7df9DqE6i0iB8eTuRh5I2KsPUrCLGv cloud-user@inception"
 node-defaults k8s ssh-username <mark>your-cnbng-cp-vm-login-name</mark>
 node-defaults k8s ssh-connection-private-key "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAwx+LkHa8U74BYN9rM3MS1H8K8ZiOdd8kqA5I8553KH75C/yV\nddtiQLBh7y+k9/YwUSsrbj6S4b6/Wb8CpTmeQXnlAfmY/LbmXUEaiKb6J4bXBEnN\ndaETOsOXUIx1CGg3Sv7wUEIOYUkNAlc71VNg0vBPnLDGwaD82lFYvqj46nwRLfTM\nzMrHNQ551zdXxTA1c0wLG+ZoLf3C43H26OqlYC7fYqxSfwH6EBpbUIvTOL3obKa2\nGZ4Bi+VPfqrfap+JW2sU7dh6jUCQLpO9NS1YWTVygnx5oIOnp9yGJO2GSNSIYQJ2\nAKpUC7gu3X/Q6hOotIgfHk7kYeSNirD1KwixrwIDAQABAoIBADKgQKne5MYlil4E\nGeBjfwM7Yy+EEZJrrysbabor52bOave9NVo67aczHHXeusLLUYX92WrlOV7xCtzS\nPnF4HaOHaO+2PwdyvRp9BdFm4YjX53npXDGk9URN8zim+MaRo6cFtnxcZza+qW1u\nDMwwsfKI/178TtV2W6SZbpkpZkwQJ/cN1fljxuQYubOujMZErc1Q7TlIQYrG8XVz\naOTVkYBU+O2DOKkWp4AM9qe/0RIdjmz1ZZHRB83iF3/OaU3j2mhlhRQwTiS3jcGX\nkWoovz/GmFAcw+VSz81tz2UFmGNczkl8F5t4afYq31M7r+restSd6Mc7dcqWA7pR\nYflz7SECgYEA+qleQDAxe9cSRyGk0aCXV7NKQnvmIReSiRxibNne2FTnLaBH7iM7\nGe8XT23gFLCvM1qmpaLxjQZEP4A7WTaZuTEQAvVHYM+4m+1XDz4+ePaSCFwru8Q/\nLvE0h45JGfuHiXIWQNsyycSnDa+50BQ1a+EruzsmdHDvVnhO/QK0gzsCgYEAx0dg\nVbOjGzEvuGTfTI8qqRYz2XQjSKr6IHqRGY/h2hVcJo0Is+fr9n2QlebJ9oIOVnw7\npSlPgnMNcI9b74aCGuVnVUW/23TRkTvq6goNBa2V++d9EHPNo2gahc1tuJZzkp8I\nnZONBHzU8jwe9nBbK7uuz9i8cgSnMaWbS6Q8PB0CgYEAoXzoWdYyqyQ+hFEqjFs3\n5ap+lyKXeo5jO65rwtECfsEERyLR9JwCAY1FqUiSawIBfcZTQrcdg8ubwIVutuU0\nWFlBhYZcPATXXK2lvw5M1UWVg4lOK6QdSLLhMsv6UKD6CxTTPWl66P6m2Wxy+5lp\naV0h/Xf4KGBx8XWE/f/2J+0CgYEAlfJGMZZut4pGLwhv4WqkngBf2VMDLa3Bcejn\n/4T9W5zQ7w0WLFDpg1quDa1P8JWh9j+ancc81Zp+1WB5u/zJLzXIkChgmeAHxLGC\nLMKNU+VuwtJHj7ajWD6AHogZ9Ff49K2HzRH2fRb1IKROY/7dC0Y43ppmCaEosTm8\nZalZzZ0CgYAF6KnlPw1qbpVZYyVcPw38omeisMs/92ovf4oqzJpZ5zCW1iKKRpQR\nq4oDLca0zJS/AUmXFTIG//VHvsna4A2JrmNS4rI+bEgwSDrRKHmX8Idctdu1ifgS\nrRtv1s5RfTZ67eWVYE6rUn1+EFBZwAzZorOXL3CzbjZEXUCJeOH6Lw==\n-----END RSA PRIVATE KEY-----"
 node-defaults netplan template "    network:\n        version: 2\n        ethernets:\n            ens192:\n                dhcp4: false\n                addresses:\n                - {{K8S_SSH_IP}}/24\n                routes:\n                - to: 0.0.0.0/0\n                  via: your-gateway\n                nameservers:\n                    addresses:\n                    - your-dns1\n                    - your-dns2\n                    search:\n                    - your-domain\n\n\n"
 node-defaults os ntp enabled
 node-defaults os ntp servers <mark>your-ntp-server1</mark>
 exit
 node-defaults os ntp servers <mark>your-ntp-server2</mark>
 exit
</code>
</pre>
</div>

In the above config, we have enabled ingress for Grafana access and configured two ntp servers.
{: .notice--info}

- Create AIO Node config:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
clusters <mark>your-cnbng-cp-cluster-name</mark>
 nodes <mark>your-cnbng-cp-vm-name</mark>
  k8s node-type master
  k8s ssh-ip <mark>your-cnbng-cp-vm-ip</mark>
  k8s node-labels disktype ssd
  exit
  k8s node-labels smi.cisco.com/node-type oam
  exit
  initial-boot default-user <mark>your-cnbng-cp-vm-login-name</mark>
  initial-boot default-user-password <mark>your-password1</mark>
  vmware datastore <mark>your-esxi-host-local-datastore</mark>
  vmware host <mark>your-esxi-host</mark>
  vmware sizing ram-mb 16384
  vmware sizing cpus 8
  vmware sizing disk-root-gb 150
! attach NIC based on the port group config
  vmware nics <mark>"VM Network"</mark>
  exit
  os ntp enabled
 exit
exit
</code>
</pre>
</div>

- cnBNG software is available as a tarball and it can be hosted on local http server for offline deployment. In this step we configure the software repository locations for tarball. We setup software cnf for both cnBNG CP and CEE. URL and SHA256 depends on the version of the image and the url location, so these two could change for your deployment.

```
software cnf bng
 url             http://192.168.107.148/your/image/location/bng.2021.04.m0.i74.tar
 sha256          e36b5ff86f35508539a8c8c8614ea227e67f97cf94830a8cee44fe0d2234dc1c
 description     bng-products
exit
software cnf cee
 url         http://192.168.107.148/your/image/location/cee-2020.02.6.i04.tar
 sha256      b5040e9ad711ef743168bf382317a89e47138163765642c432eb5b80d7881f57
 description cee-products
exit
```

- Setup Ops Center configs inside cluster for cnBNG

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

- Deploy cnBNG CP cluster using cluster sync command.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
[inception] SMI Cluster Deployer# <mark>clusters your-cnbng-cp-cluster-name actions sync run</mark>   
This will run sync.  Are you sure? [no,yes] yes
message accepted
[inception] SMI Cluster Deployer# 
</code>
</pre>
</div>

- Monitor sync using below command. It will take 45mins to sync and deploy the cluster. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
<mark>monitor sync-logs your-cnbng-cp-cluster-name</mark>

E.g.

[inception] SMI Cluster Deployer# monitor sync-logs podX
. . .
. . .
TASK [cluster-manager : Deleting cluster manager chart directory] **************
Thursday 18 November 2021  14:38:49 +0000 (0:00:00.042)       0:08:42.683 ***** 
skipping: [cnbng-cp]

PLAY RECAP *********************************************************************
<mark>cnbng-cp                   : ok=483  changed=209  unreachable=0    failed=0    skipped=441  rescued=0    ignored=0</mark>   
localhost                  : ok=57   changed=8    unreachable=0    failed=0    skipped=20   rescued=0    ignored=0   

Thursday 18 November 2021  14:38:49 +0000 (0:00:00.064)       0:08:42.748 ***** 
=============================================================================== 
2021-11-18 14:38:49.716 DEBUG cluster_sync.podX: <mark>Cluster sync successful</mark> 
2021-11-18 14:38:49.717 DEBUG cluster_sync.podX: Ansible sync done 
2021-11-18 14:38:49.717 INFO cluster_sync.podX: _sync finished.  Opening lock 
</code>
</pre>
</div>

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
</pre>
</div>
