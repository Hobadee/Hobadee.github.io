---
layout: post
title:  "Creating a Ceph cluster with iSCSI for VMWare"
date:   2019-12-13 09:20:22 -0800
categories: ceph iscsi vmware storage
---
## Introduction
Currently, we are running shared storage for our VMWare cluster on a single FreeNAS box.  While this has worked great for us for several years now (I say as I'm knocking on wood as hard as possible) we need a solution that is more robust; where we can perform maintenance on it without taking all systems offline.

Ceph appeared to be the perfect solution to this; we could create a cluster with multiple iSCSI interfaces, then take nodes or disks offline as needed for maintenance.  Unfortunatly, the documentation for Ceph isn't the best out there.  While there is a fair amount of it, it is largely targetted at those who already know something about Ceph, and the documentation for beginners is lacking, confusing, and in some cases wrong.  This article serves to narrow that gap in documentation, making it easy to bootstrap your own cluster and begin playing with it and learning about it.

**Please note:** I am in no way a Ceph expert, nor is this setup performance optimized.  This is simply what I did and the pitfalls I encountered to get a cluster working with the features we needed.  (I plan on doing another article on actual optimizations we do)  I know for a fact that there are further optimizations that need to be done before we can put this cluster in production!  This is intended to start you down a path where you can begin playing, testing, and optimizing your own cluster.


## Assumptions
I assume you have at least a basic understanding of Ansible and Linux system administration.


## Equipment and Pre-Configuration
I started with 4 Dell R510 8-Bay machines.  Because we don't currently have the appropriate controllers (we have one on the way for testing) I am using the PERC H300 controllers they came with.  I have setup 2 disks in a RAID-1 for the OS, and the remaining 6 disks are setup as individual RAID-0s for data.  In the long run, we will replace the RAID controller with a JBOD controller so the data disks can present as JBOD.

I did a clean minimal install of CentOS 7.  While CentOS 8 is out right now, it removed the drivers for a bunch of older RAID controllers, ours included.  Wanting to avoid spending time getting Cent 8 to work with our controllers, I just opted to just install Cent7.

After Cent7 was installed, I ran our own Ansible playbook that we use to bootstrap servers.  This playbook does a few basic things for all our new servers:
* Creates an 'ansible' user with password-less sudo permissions
* Locks out the root password so root can't login
* Binds to AD so we can login with our AD credentials
* Configures sudo for AD so admins can sudo
* Installs a few packages we assume to be installed at all times (git, zsh, emacs, etc...)
* Install, configure, and enable SELinux

As a side note, in the future we plan to expand this playbook to also do things like setup logging to go to our syslog server, setup performance logging and AIDE, and configure additional network interfaces.

So other than the items above, the machines we are starting with are clean, minimal, installs of CentOS 7.


## Network Setup
For this cluster, we wanted it to be as redundant as possible.  To achieve this, each Ceph host has 3 network interfacs; Management, Ceph Frontend, and Ceph Backend.  The management interface is how we control the node; it is accessible on LAN and is the only way into the nodes.  The other 2 networks are a completely segregated SAN with no access unless you are physically on the SAN.

Here is a diagram of our SAN.  Green are the Ceph nodes which will be iSCSI gateways, and light green nodes will not have iSCSI gateways.  As you can see, if all links are working properly, there are 4 paths between any given ESXi host, and an iSCSI target.  We can have a complete frontend switch failure and not loose availability.

{% graphviz %}
digraph {
  node [shape=rectangle, style=filled];
  FESW1 [fillcolor=lightblue];
  FESW2 [fillcolor=lightblue];
  BESW1 [fillcolor=lightblue];
  node [shape=box3d, style=filled];
  ESXi01 [];
  ESXi02 [];
  ESXi03 [];
  node [shape=cylinder, style=filled];
  STOR01 [fillcolor=green, fontcolor=white];
  STOR02 [fillcolor=green, fontcolor=white];
  STOR03 [fillcolor=lightgreen];
  STOR04 [fillcolor=lightgreen];

  ESXi01 -> FESW1 [arrowhead=none];
  ESXi01 -> FESW2 [arrowhead=none];
  ESXi02 -> FESW1 [arrowhead=none];
  ESXi02 -> FESW2 [arrowhead=none];
  ESXi03 -> FESW1 [arrowhead=none];
  ESXi03 -> FESW2 [arrowhead=none];

  FESW1 -> FESW2 [constraint=false, arrowhead=none];

  FESW1 -> STOR01 [arrowhead=none];
  FESW2 -> STOR02 [arrowhead=none];
  FESW1 -> STOR03 [arrowhead=none];
  FESW2 -> STOR04 [arrowhead=none];

  STOR01 -> BESW1 [arrowhead=none];
  STOR02 -> BESW1 [arrowhead=none];
  STOR03 -> BESW1 [arrowhead=none];
  STOR04 -> BESW1 [arrowhead=none];
}
{% endgraphviz %}


## Process
When I began looking into it, I realized that Ceph would be best deployed with an automation tool.  Seeing that there is an official [ceph-ansible][ceph-ansible] project, I decided to use that.  Unfortunatly, the documentation for beginners there is rather broken.  There are MANY options to configure in the YAML files, but they aren't well documented; some of them are mandatory, some aren't.  Some will prevent Ansible from setting up your cluster, some won't.  There is no documentation that I could find of the proper combination of options for a baseline system, so after much guesswork and trail & error, I came up with the following configs.  (Sensitive information has been <<redacted>>)

First my `hosts.yml` file.

**hosts.yml**
``` yml
all:
  hosts:
    stor0[1:4].<<redacted>>:
      ansible_user: ansible
  children:
    mgrs:
      hosts:
        stor0[1:3].<<redacted>>:
    mons:
      hosts:
        stor0[1:3].<<redacted>>:
    osds:
      hosts:
        stor0[1:4].<<redacted>>:
          devices:
            - /dev/sdb
            - /dev/sdc
            - /dev/sdd
            - /dev/sde
            - /dev/sdf
            - /dev/sdg
    iscsigws:
      hosts:
        stor0[1:2].<<redacted>>:
    grafana-server:
      hosts:
        stor01.<<redacted>>:
```

The example configs all say to configure your OSD devices inside the `group_vars/osds.yml` file, but that only works if each machine has the exact same number of drives in the same configuration.  We plan on adding 12-bay R510s in the near future (when we deprovision our FreeNAS machines) so globally configuring a set number of drives doesn't make sense for us.  Fortunatly, you can configure this option per-machine in your `hosts.yml` file.  Right now we just have our 8-bay machines with 6 OSD drives each, so I grouped them together, but it will be simple to add our 12-bay machines later.

The only other thing to note is that we set our `ansible_user`.  I could have easily set this in `all.yml`, but I put it here.  No real reason why, it just felt better to have it here than in with Ceph-specific stuff.

Now here are my `group_vars`.  I have stripped away all the comments so you can just see the config variables that I actually used.

**group_vars/all.yml**
``` yml
---
dummy:
fetch_directory: fetch/
configure_firewall: True
ceph_mon_firewall_zone: public
ceph_mgr_firewall_zone: public
ceph_osd_firewall_zone: public
ceph_iscsi_firewall_zone: public
ceph_dashboard_firewall_zone: public
centos_package_dependencies:
  - epel-release
  - libselinux-python
ceph_origin: repository
ceph_repository: community
ceph_stable_release: nautilus
cephx: true
monitor_address_block: 10.7.11.0/24
ip_version: ipv4
public_network: 10.7.11.0/24
cluster_network: 10.7.12.0/24
ceph_tcmalloc_max_total_thread_cache: 0
dashboard_admin_password: "<<redacted>>"
dashboard_rgw_api_no_ssl_verify: true
```


**group_vars/mgrs.yml**
``` yml
---
dummy:
ceph_mgr_modules: 
  - dashboard
  - prometheus
  - influx
ceph_mgr_packages:
  - ceph-mgr
  - ceph-mgr-dashboard
  - ceph-mgr-diskprediction-local
  - python2-pip
```


**group_vars/iscsigws.yml**
``` yml
---
dummy:
api_user: "admin"
api_password: "<<redacted>>"
```


**group_vars/mons.yml**
``` yml
dummy:
```


**group_vars/osds.yml**
``` yml
dummy:
```

Note that I left `mons.yml` and `osds.yml` empty as I expect I will use them at some point.  The rest of the samples I don't expect to use in our use-case so I just left them as-is.  (non-existent, as far as Ansible is concerned)


## Pitfalls
### Test runs
Prior to this run, I did several test runs.  In fact, it took me about 6 days of banging my head against the wall trying to solve various issues.  Here are a few notes that I figured out.

* If your monitor nodes can't form a quorum, and all the other troubleshooting steps have been completed, (Check firewalls, SELinux, mon maps, etc...) check your MTU settings and ensure your switches have *higher* MTUs than your hosts.  MTU settings will prevent monitors from talking to each other but won't give you any errors hinting at MTU being an issue.
* Some settings, especially deprecated iSCSI settings, will prevent ceph-ansible from creating a usable cluster.  I did't have time to map out specific culprits, but when in doubt, start with very small configs and add items if absolutely needed.  (Most of the needed items will be checked for in the initial steps of ceph-ansible.  I came across one that wasn't, but the error message was pretty obvious.)


## Final Cluster

### First run
Log: ceph-ansible.2019-12-13T153612.log
Failed due to missing python influxdb dependancy.  You will need to run `pip install influxdb`.

### Second run
Log: ceph-ansible.2019-12-13T154530.log
Still failed due to missing python influxdb dependancy.  Huh!?

### Third run
Log: ceph-ansible.2019-12-13T161906.log
I commented out the `influx` line from the `ceph_mgr_modules` section inside `group_vars/mgrs.yml`
Failed due to OSDs failing to start.  This is likely because I didn't do a `purge_cluster` before re-installing, so OSDs were still assigned to the old cluster and the new cluster provisioning won't wipe them.  Gotta manually wipe them.

### Forth run
Log: ceph-ansible.2019-12-13T234628.log
I ran `purge-cluster` in an attempt to delete the stuck OSDs.  Prior to running it, I tried to manually wipe the partition table of the OSD drives directly from the hosts, but some process was using them so I decided against doing that.  After purging the cluster, I used `wipefs -a` to kill the partition table of each OSD disk

### Fifth run
Log: ceph-ansible.2019-12-13T235634.log
5th time is the charm?  Yes!  Yes it is!  We have a Ceph cluster!


## Post-Install Setup & Notes

### iSCSI Config
While you are supposed to be able to configure iSCSI through the web interface, in reality you must configure 1 target through the `gwcli` program first before the web interface works.  The [Ceph iSCSI Target CLI][ceph-iscsi-target-cli] directions are actually pretty good for this.

I should also note that using the web interface to further configure iSCSI appeared to corrupt the iSCSI configuration from time to time.  I would recommend staying away from the web interface and sticking solely to the `gwcli` program for iSCSI configuration.  (It's easy once you get the hang of it.)


### Resizing a LUN
It appears that resizing an iSCSI LUN is a little more involved than some other systems.  In particular, the FreeNAS system we are coming from you can just resize the block image and rescan the LUN and it has been resized.  It appears that with Ceph, you must restart the iSCSI gateways.  I will need to do more experimentation into this though as I was trying to use the web interface (which was causing some other issues; see above) and I only did minimal experimentation on this.

Here is the process I used.  (Using the web interface)
1. Resize the RBD image
1. Unmount the Datastore
1. Unmount the LUN
1. Restart the iSCSI gateways.  (1 at a time)
1. Remount the LUN
1. Remount the Datastore


### Benchmarking
I ran an initial benchmark using Bonnie++.  The output is below:

Bonnie++ Output
```
bonnie++ -d /tmp/bonnie/ -s 5G
Writing a byte at a time...done
Writing intelligently...done
Rewriting...done
Reading a byte at a time...done
Reading intelligently...done
start 'em...done...done...done...done...done...
Create files in sequential order...done.
Stat files in sequential order...done.
Delete files in sequential order...done.
Create files in random order...done.
Stat files in random order...done.
Delete files in random order...done.
Version  1.97       ------Sequential Output------ --Sequential Input- --Random-
Concurrency   1     -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
ceph-test-linux  5G   651  95 101525  14 61141  11  1072  99 66183   5  3794 112
Latency             26668us     397ms    1066ms   29247us     541ms     115ms
Version  1.97       ------Sequential Create------ --------Random Create--------
ceph-test-linux     -Create-- --Read--- -Delete-- -Create-- --Read--- -Delete--
              files  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP
                 16 18315  80 +++++ +++ +++++ +++ 29574  79 +++++ +++ +++++ +++
Latency             12265us     804us     541us     837us     170us     802us
1.97,1.97,ceph-test-linux,1,1576316721,5G,,651,95,101525,14,61141,11,1072,99,66183,5,3794,112,16,,,,,18315,80,+++++,+++,+++++,+++,29574,79,+++++,+++,+++++,+++,26668us,397ms,1066ms,29247us,541ms,115ms,12265us,804us,541us,837us,170us,802us
```




[ceph-iscsi-target-cli]: https://docs.ceph.com/docs/master/rbd/iscsi-target-cli/
[ceph-ansible]: https://github.com/ceph/ceph-ansible
