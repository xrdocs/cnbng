---
published: true
date: '2022-12-16 18:02 +0530'
title: Day0 AIO YAML Template
hidden: true
position: hidden
---
<div class="highlighter-rouge">
<pre class="highlight">
<code>
<mark>images:</mark>
    bng:
      name: bng.2022.04.0
      url: http://192.168.107.152/images-2022/cnbng-cp/bng.2022.04.0.SPA.tgz
      sha256: 66e1ad17183226dc0b46a3e62b85e9b992b42c1d69e84f0e8b397c553c6901d7

    cee:
      name: cee-2022.02.1.i08
      url: http://192.168.107.152/images-2022/cnbng-cp/cee-2022.02.1.i08.tar
      sha256: bbff1b7d1f66595c0ffa399e7ae58884e1b1a8c5a61e19398006c53ab3f635b0

<mark>vcenter_server:</mark>
  ip: 192.168.107.146
  user: administrator@cnbng.com
  password: Bgl16dlab@!23
  datacenter: cnBNG_DC_01
  cluster: cnBNG_SMI_CL01
  host: 192.168.107.136
  datastore: C1S1Datastore
  nic: VM Network

<mark>cnbng_cp:</mark>
  cluster:
     name: <mark>demo-cp</mark>
     ntp: <mark>ntp.esl.cisco.com</mark>
     istio: <mark>'false'</mark>
     cee_ops_center:
        ssh_port: 2023
        netconf_port: 3024
     bng_ops_center:
        ssh_port: 2024
        netconf_port: 3022
  node_vm:
    ip: <mark>192.168.107.163</mark>
    gateway: <mark>192.168.107.129</mark>
    dns: <mark>64.104.128.236</mark>
    domain: <mark>cisco.com</mark>
    sizing:
      vcpu: 16
      memory: 32768
      storage_gb: 300

<mark>smi_deployer:</mark>
  ip: 192.168.107.174
  user: 'admin'
  password: 'Cisco@123'
</code>
</pre>
</div>
