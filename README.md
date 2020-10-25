

ZFS High-Availability NAS
=========================

The [ZFS filesystem](https://en.wikipedia.org/wiki/ZFS) has been a game-changer in the way we approach local application data storage, shared storage solutions, replication and general data backup. 

I've been a long-time proponent of ZFS storage in a variety of scenarios, going back to my first experiences with OpenSolaris in 2008, buying my own [ZFS Thumper/Thor in 2009](https://www.flickr.com/photos/ewwhite/albums/72157615393965826), adopting [ZFS on Linux](http://zfsonlinux.org) for production use in 2012, and through continued contributions to the [StackExchange/ServerFault](http://serverfault.com/users/13325/ewwhite?tab=profile) communities.

ZFS Advantages:

- More intelligent storage for application servers and a serious replacement for **[LVM](https://en.wikipedia.org/wiki/Logical_Volume_Manager_%28Linux%29)**.
- Great shared storage options to back virtualization environments.
- Useful for large-scale backup targets.
- Atomic snapshots. 
- Flexible replication. 
- Transparent filesystem compression.

ZFS Downsides:

- *Good* ZFS implementations sometimes require specialized knowledge and [adherence to best practices](http://nex7.blogspot.com/2013/03/readme1st.html).
- It's easy to make irreversible mistakes. (often due to poor design or inappropriate hardware choices).
- There's a lot of contradictory ZFS information online due to the filesystem's appeal to *"home lab"* users.
- **Options for high availability are costly, tightly associated with commercial storage solutions and may not work well.**

Expanding on the final point, the traditional method of achieving high availability on a ZFS storage array means paying for licensing of a commercial product suite. The most common options in the marketplace are [NexentaStor](https://nexenta.com/products/nexentastor), [QuantaStor](http://www.osnexus.com/storage-appliance-os/), [RSF-1](http://www.high-availability.com/zfs-ha-plugin/) and [Zetavault](http://www.zeta.systems/zetavault/high-availability/).

These products are proven to work and have existing user bases, but some have [particularly unattractive licensing schemes](https://static1.squarespace.com/static/545a62d4e4b02643996231f0/t/54b803dae4b06fad9b8d8b85/1421345754630/osn_quantastor_2015_price_guide.pdf). Notably, NexentaStor and QuantaStor have capacity-based licensing models that don't work well for my use case. The high-availability licensing add-ons to both solutions are expensive, yet still place the onus of the hardware build, validation, deployment and ongoing support on the customer. Zetavault has made positive strides in providing an [interesting shared-nothing architecture](http://www.zeta.systems/blog/2016/10/11/High-Availability-Storage-On-Dell-PowerEdge-&-HP-ProLiant/) that may work for some situations.

As a consultant, I work with small businesses who have simple VMware estates (2-6 hosts), but need robust storage to back the solution. They typically have less than 6 Terabytes of usable storage requirements, but the cost of lower tier SAN storage is way out of line with their technology budgets.

A licensed highly-available build of a 16TB raw commercial ZFS array built atop commodity hardware approaches $25-30k. Without HA, that solution may be less than $8k. The HA option is very close to the price of a fully-integrated storage array like [Nimble](https://www.nimblestorage.com) or [Tegile](http://www.tegile.com). The value proposition of the commercial ZFS solution is low when placed in this context. 

Objectives
---------
Build a highly-available dual-controller storage array using open-source technologies.  
This solution should be capable of presenting shared storage to any NFS client.  
This should be the ZFS equivalent of an dual-controller array like the [HP StorageWorks MSA2040](http://www8.hp.com/us/en/products/disk-storage/product-detail.html?oid=5386548#!tab=features).

____

The key to this high-availability storage design is a shared SAS-attached storage enclosure,  or JBOD.
Examples of this include:

- HP StorageWorks [D2600](https://www.hpe.com/h20195/v2/GetPDF.aspx/c04111697.pdf)/[D2700](https://www.hpe.com/h20195/v2/GetPDF.aspx/c04111697.pdf)/[D3600](http://www8.hp.com/h20195/v2/GetPDF.aspx/c04227611.pdf)/[D3700](http://www8.hp.com/h20195/v2/GetPDF.aspx/c04227611.pdf) 2U enclosures.
- HP StorageWorks [MDS600](http://www.bargainhardware.co.uk/content/HP%20Storage%20Works%20MSD600.pdf) and [D6000-series](http://www8.hp.com/h20195/v2/GetDocument.aspx?docname=c04111505&search=mds600) high-density enclosures
- Sun Microsystems/Oracle [J4410](https://docs.oracle.com/cd/E26765_01/html/E26399/maintenance__hardware__overview__shelf__sas2.html#scrolltoc) JBOD.
- Dell PowerVault [MD1200](http://www.dell.com/downloads/global/products/pvaul/en/storage-powervault-md1200-specsheet.pdf).
- DataON [DNS-1660](http://www.dataonstorage.com/dns-jbod-enclosure/dns-1660-4u-60-bay-6g-35inch-sassata-jbod.html).

These external SAS enclosures all feature:

- Redundant power supplies and fans.
- SAS expander logic on the backplane.
- Redundant SAS controllers (*I/O modules*) with end-to-end multipathing to accommodate dual-ported SAS disks. ***Dual-ported SAS drives are critical to the success of this design!***
- SCSI enclosure services (SES) sensors for communication of enclosure component status.

[![HP StorageWorks D2700 25-bay enclosure](http://i.imgur.com/gNwRuCb.png)](http://i.imgur.com/gNwRuCb.png)

To complement the shared JBOD enclosure, we need two servers (*head nodes or controllers*) to provide client connectivity and compute/RAM resources for the ZFS array.

It make sense to scale the specifications of the head nodes to meet the anticipated workload of the installation. Variables may include:

- **PCIe slot connectivity** - For NICs, SAS host bus adapters and future expansion.
- **RAM** - ZFS leverages system RAM for read and write caching. Maximizing RAM amount (within reason) can help accelerate I/O workloads.
- **Speed and number of CPUs** - This impacts data compression performance.
- **Cost** - .

My recommendation is to use an Intel [Nehalem](https://en.wikipedia.org/wiki/Nehalem_%28microarchitecture%29), [Westmere](https://en.wikipedia.org/wiki/Westmere_%28microarchitecture%29) or newer CPU with four or more cores. 

Example budget parts manifest
--------------

Low-cost build manifest for a simple 12TB usable storage array *(under $3.5k)*:

> - 2 x [HP ProLiant DL360 G7](https://www.ebay.com/sch/i.html?_from=R40&_trksid=m570.l1313&_nkw=dl360+g7&_sacat=0) rackmount servers (~$750 total) 
> - 2 x [HP H221](http://h20564.www2.hpe.com/hpsc/doc/public/display?docId=emr_na-c03231137) SAS HBA ($240 total) 
> - 4 x SAS SFF-8088 external SAS cables ($100 total) 
> - 1 x [HP StorageWorks D2600](http://www.ebay.com/itm/AJ940A-HP-STORAGEWORKS-D2600-DISK-ENCLOSURE-Rail-kit-included-/162148323089) 12-bay SAS enclosure (~$600 total) 
> - 10 x [HP 2TB](http://www.ebay.com/itm/HP-507616-B21-508010-001-2TB-SAS-7-2K-6GB-3-5-HDD-/170672527477?hash=item27bce01875:g:mh8AAOSwbYZXdEFM) dual-port SAS hard drives ($700 total) 
> - 1 x [SAS SSD](http://www.ebay.com/itm/152044917171) for L2ARC ($200) - _2.5" disks will need a [3.5" adapter](https://www.amazon.com/gp/product/B00DCBFF8Q/ref=oh_aui_detailpage_o02_s00?ie=UTF8&psc=1)_
> - 1 x [Stec ZeusRAM](http://www.ebay.com/sch/i.html?_odkw=zeusram&_osacat=0&_from=R40&_trksid=p2045573.m570.l1313.TR0.TRC0.H0.TRS0&_nkw=zeusram&_sacat=0) SAS DRAM SSD for ZIL/SLOG ($600)

_HP StorageWorks D2600 fully disassembled._
[![enter image description here](http://i.imgur.com/bTOqqxP.jpg)](http://i.imgur.com/bTOqqxP.jpg)

_Front view of HP ProLiant DL360 G7 head nodes and D2600 JBOD._
[![enter image description here](http://i.imgur.com/bo790aX.jpg)](http://i.imgur.com/bo790aX.jpg)

_Rear view of servers and JBOD with 6G SAS cabling._
[![enter image description here](http://i.imgur.com/goIxUMr.jpg)](http://i.imgur.com/goIxUMr.jpg)

Example high-end parts manifest
-------------------------------

Build manifest for a 24TB usable storage array using new components *(under $15k)*:

> - 2 x HP ProLiant DL360 Gen8/Gen9 rackmount servers ($6000 total)
> - 2 x HP H241 SAS HBA ($400 total)
> - 4 x SAS SFF-8644 external cables ($280 total)
> - 1 x HP StorageWorks D3600 12-bay SAS enclosure ($2500 total)
> - 10 x HP 4TB dual-port SAS hard drives ($4000 total)
> - 1 x read-optimized SSD for L2ARC ($400 total)
> - 1 x low-latency SSD for ZIL/SLOG ($800)

**Note**: *The ZIL/SLOG device can be mirrored, depending on requirements. I tend to use the Stec ZeusRAM 8GB SSD for this purpose. Failure or wearout are very uncommon due to the nature of the device's architecture. If using a non DRAM-based SSD, it may make sense to mirror the SLOG.* 

_Front view of HP ProLiant DL360p Gen8 head nodes and D3600 JBOD._
[![enter image description here](http://i.imgur.com/GHXJn5o.jpg)](http://i.imgur.com/GHXJn5o.jpg)

_Rear view of servers and JBOD with 12G SAS cabling._
[![enter image description here](http://i.imgur.com/CCH7FDn.jpg)](http://i.imgur.com/CCH7FDn.jpg)

***

Physical setup
--------------

The purpose of the shared JBOD in this setup is to have a multipath configuration that provides cabling, controller, host adapter and path redundancy. 

The most basic recommended setup between two head nodes and a single JBOD enclosure is a single 2-port SAS HBA in each host and four SAS cables with the following arrangement.

> node1 HBA port 1 -> JBOD controller1 port 2  
> node1 HBA port 2 -> JBOD controller2 port 2 
> 
> node2 HBA port 1 -> JBOD controller1 port 1  
> node2 HBA port 2 -> JBOD controller2 port 1  

_Examples of how to scale with multiple enclosures and SAS cabling rings._
![Oracle SAS cabling diagram](http://i.imgur.com/5bpqh3Z.png)

Cluster configuration
---------

**Assumptions and requirements:**

- Root access.
- Two server systems with RHEL or CentOS 7 installed on local disks.
- The ZFS filesystem configured via the [ZFS on Linux repository](https://github.com/zfsonlinux/zfs/wiki/RHEL-and-CentOS).
- Familiarity with basic ZFS operations: zpool/zfs filesystem creation and modification.
- Understanding of network bonding under RHEL Linux.

These steps describe the construction of a two host, single JBOD cluster that manages a single ZFS pool comprised of 10 data drives *(9 pool+hot spare)* and separate SLOG (ZIL) and L2ARC drives. 
___

**Corosync/Pacemaker setup:**  
_Additional information about the [RHEL High Availability Add-On](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/High_Availability_Add-On_Reference/ch-overview-HAAR.html)._

For simplification, disable firewalld for the build process. This can definitely be modified later.

    systemctl stop firewalld
    systemctl disable firewalld

Install the ZFS filesystem by downloading the ZFS and kernel-devel packages.
  
    yum localinstall http://download.zfsonlinux.org/epel/zfs-release.el7_6.noarch.rpm

Consider making the yum repo modification to use the kABI-tracking module versus DKMS:
https://github.com/zfsonlinux/zfs/wiki/RHEL-and-CentOS#kabi-tracking-kmod

    yum install kernel-devel zfs

Install the RHEL cluster suite and multipath software.

    yum install pcs fence-agents-all device-mapper-multipath

Start the Pacemaker and Corosync services.

    systemctl enable pcsd
    systemctl enable corosync
    systemctl enable pacemaker
    systemctl start pcsd
   
The multipath daemon will not start without a configuration file present.

    mpathconf --enable
    systemctl start multipathd
    systemctl enable multipathd

Download the [ZFS zpool Pacemaker OCF agent](https://github.com/skiselkov/stmf-ha).
This allows the ZFS pool to be exported and imported to each cluster node during failover.

    cd /usr/lib/ocf/resource.d/heartbeat/
    wget https://github.com/skiselkov/stmf-ha/raw/master/heartbeat/ZFS
    chmod +x ZFS

Networking depends on how clients will consume data from this NAS.

For most of my builds, I use [LACP bonding](https://en.wikipedia.org/wiki/Link_aggregation#Link_Aggregation_Control_Protocol) from the storage array to the switches and maintain an IP on the data network for system management. The most basic requirement is to have an IP for each host, plus a virtual IP (VIP) that can float to the active node. NFS clients will use this virtual IP.

I prefer to segregate traffic by VLAN, so I create a single master LACP bond interface with separate VLAN interfaces, e.g. `bond0.10`,`bond0.777`,`bond0.91`.  

Example `/etc/sysconfig/network-scripts/` [interface script files here](http://pastebin.com/g3fiUm41).

NetworkManager isn't necessary for this since the bond interfaces will likely require hand-editing.

    systemctl stop NetworkManager
    systemctl disable NetworkManager

Populate `/etc/hosts` files with the cluster members' hostnames and add a second heartbeat address ring for Corosync.

    # Management addresses of both nodes
    172.16.40.15	zfs-node1.ewwhite.net zfs-node1
    172.16.40.16	zfs-node2.ewwhite.net zfs-node2
    
    # Cluster ring address for heartbeat
    192.168.91.1	zfs-node1-ext
    192.168.91.2	zfs-node2-ext

Ensure each of the addresses can be reached from either host.

___

**Determine a drive layout:** 

Due to the use of `dm-multipath`, we want to build the ZFS pool using the device mapper disk identifiers rather than the normal /dev entries. 

There are a few ways to determine the drive layout and identify disks. This will depend on the HBAs and JBOD enclosure in use. It makes sense to record the SAS [WWN](https://en.wikipedia.org/wiki/World_Wide_Name) of each of the disks you'll be using.

In some cases, it is possible to enumerate drives and identifying WWNs programmatically or by examining sysfs entries. For example, with the HP H221 SAS HBA:

    cd /sys/class/enclosure
    
    [root@zfs1-1 /sys/class/enclosure]# ls
    0:0:11:0  0:0:23:0
    
    [root@zfs1-1 /sys/class/enclosure/0:0:11:0]# ll
    total 0
    drwxr-xr-x 3 root root    0 Jul  6 11:57 0
    drwxr-xr-x 3 root root    0 Jul  6 11:57 1
    drwxr-xr-x 3 root root    0 Jul  6 11:57 10
    drwxr-xr-x 3 root root    0 Jul  6 11:57 11
    drwxr-xr-x 3 root root    0 Jul  6 11:57 2
    drwxr-xr-x 3 root root    0 Jul  6 11:57 3
    drwxr-xr-x 3 root root    0 Jul  6 11:57 4
    drwxr-xr-x 3 root root    0 Jul  6 11:57 5
    drwxr-xr-x 3 root root    0 Jul  6 11:57 6
    drwxr-xr-x 3 root root    0 Jul  6 11:57 7
    drwxr-xr-x 3 root root    0 Jul  6 11:57 8
    drwxr-xr-x 3 root root    0 Jul  6 11:57 9
    -r--r--r-- 1 root root 4096 Jul  6 11:57 components
    lrwxrwxrwx 1 root root    0 Jul  6 11:57 device -> ../../../0:0:11:0
    drwxr-xr-x 2 root root    0 Jul  6 11:57 power
    lrwxrwxrwx 1 root root    0 Jul  6 11:57 subsystem -> ../../../../../../../../../../../../../class/enclosure
    -rw-r--r-- 1 root root 4096 Jul  4 23:27 uevent
    
The 0 through 11 above represent the 12 drive slots in the HP StorageWorks D2600 I'm using in this build. 

    [root@zfs1-1 /sys/class/enclosure/0:0:11:0/0]# ll
    total 0
    -rw-r--r-- 1 root root 4096 Jul  6 11:59 active
    lrwxrwxrwx 1 root root    0 Jul  6 11:59 device -> ../../../../../../../port-0:0:0/end_device-0:0:0/target0:0:0/0:0:0:0
    -rw-r--r-- 1 root root 4096 Jul  6 11:59 fault
    -rw-r--r-- 1 root root 4096 Jul  6 11:59 locate
    drwxr-xr-x 2 root root    0 Jul  6 11:59 power
    -rw-r--r-- 1 root root 4096 Jul  6 11:59 status
    -r--r--r-- 1 root root 4096 Jul  6 11:59 type
    -rw-r--r-- 1 root root 4096 Jul  6 11:59 uevent

Note the `device` directory and `locate` options. 
Running `echo 1 > locate` will illuminate the drive beacon on the device present in that slot.

    [root@zfs1-1 /sys/class/enclosure/0:0:11:0/0]# cd device

    [root@zfs1-1 /sys/class/enclosure/0:0:11:0/0/device]# cat sas_address
    0x5000c500236004a2

The output of `multipath -ll` should show the paths, selection policy, SCSI device name and /dev/mapper device name (_e.g. 35000c500236061b3_) for each of the drives in the enclosure.

    [root@zfs1-1 ~]# multipath -ll
    35000c500236061b3 dm-11 HP      ,EF0450FARMV
    size=419G features='0' hwhandler='0' wp=rw
    |-+- policy='round-robin 0' prio=1 status=active
    | `- 0:0:3:0  sde 8:64   active ready running
    `-+- policy='round-robin 0' prio=1 status=enabled
      `- 0:0:16:0 sdq 65:0   active ready running
    35000c5007772e5ff dm-3 HP      ,EF0450FARMV
    size=419G features='0' hwhandler='0' wp=rw
    |-+- policy='round-robin 0' prio=1 status=active
    | `- 0:0:2:0  sdd 8:48   active ready running
    `-+- policy='round-robin 0' prio=1 status=enabled
      `- 0:0:15:0 sdp 8:240  active ready running
    35000c500236032f7 dm-0 HP      ,EF0450FARMV
    size=419G features='0' hwhandler='0' wp=rw
    |-+- policy='round-robin 0' prio=1 status=active
    | `- 0:0:19:0 sdt 65:48  active ready running
    `-+- policy='round-robin 0' prio=1 status=enabled
      `- 0:0:6:0  sdh 8:112  active ready running

The `lsscsi` command can also help display topology. 
_Note how the drives appear twice in the SAS layout._

    [root@zfs1-2 ~]# lsscsi
    [0:0:0:0]    disk    HP       EF0450FARMV      HPD6  /dev/sdb
    [0:0:1:0]    disk    HP       EF0450FARMV      HPD6  /dev/sdc
    [0:0:2:0]    disk    HP       EF0450FARMV      HPD6  /dev/sdd
    [0:0:3:0]    disk    HP       EF0450FARMV      HPD6  /dev/sde
    [0:0:4:0]    disk    HP       EF0450FARMV      HPD6  /dev/sdf
    [0:0:5:0]    disk    HP       EF0450FARMV      HPD6  /dev/sdg
    [0:0:6:0]    disk    HP       EF0450FARMV      HPD6  /dev/sdh
    [0:0:7:0]    disk    HP       EF0450FARMV      HPD6  /dev/sdi
    [0:0:8:0]    disk    HP       EF0450FARMV      HPD6  /dev/sdj
    [0:0:9:0]    disk    HP       EF0450FARMV      HPD6  /dev/sdk
    [0:0:10:0]   disk    HITACHI  HUSSL4010BSS600  A110  /dev/sdl
    [0:0:11:0]   disk    STEC     ZeusRAM          C023  /dev/sdm
    [0:0:12:0]   enclosu HP       D2600 SAS AJ940A 0150  -
    [0:0:13:0]   disk    HP       EF0450FARMV      HPD6  /dev/sdn
    [0:0:14:0]   disk    HP       EF0450FARMV      HPD6  /dev/sdo
    [0:0:15:0]   disk    HP       EF0450FARMV      HPD6  /dev/sdp
    [0:0:16:0]   disk    HP       EF0450FARMV      HPD6  /dev/sdq
    [0:0:17:0]   disk    HP       EF0450FARMV      HPD6  /dev/sdr
    [0:0:18:0]   disk    HP       EF0450FARMV      HPD6  /dev/sds
    [0:0:19:0]   disk    HP       EF0450FARMV      HPD6  /dev/sdt
    [0:0:20:0]   disk    HP       EF0450FARMV      HPD6  /dev/sdu
    [0:0:21:0]   disk    HP       EF0450FARMV      HPD6  /dev/sdv
    [0:0:22:0]   disk    HP       EF0450FARMV      HPD6  /dev/sdw
    [0:0:23:0]   disk    HITACHI  HUSSL4010BSS600  A110  /dev/sdx
    [0:0:24:0]   disk    STEC     ZeusRAM          C023  /dev/sdy
    [0:0:25:0]   enclosu HP       D2600 SAS AJ940A 0150  -
    [1:0:0:0]    disk    HP       LOGICAL VOLUME   6.64  /dev/sda
    [1:3:0:0]    storage HP       P410i            6.64  -

Once there's some idea of the SAS address layout of the drives, we can create a zpool. 

    # Create a zpool using the /dev/mapper devices.
    #
    # The most critical cluster pool creation option is cachefile=none
    # It allows the HA agents to manage the pool versus the OS.
    #
    # Set ashift=12 if using 4k-sector disks.
    # Set ashift=13 for NVMe or certain SSD drives.
    #
    zpool create vol1 -o ashift=12 -o autoexpand=on -o autoreplace=on -o cachefile=none \ 
    raidz1 35000c500236004a3 35000c50023614aef 35000c5007772e5ff \ 
    raidz1 35000c500236061b3 35000c5002362f347 35000c500544508b7 \ 
    raidz1 35000c500236032f7 35000c50023605c1b 35000c5002362ffab \
    spare 35000c500236031ab

The result:

    [root@zfs1-1 ~]# zpool status -v
      pool: vol1
     state: ONLINE
      scan: scrub repaired 0 in 1h14m with 0 errors on Sun Aug  7 05:00:49 2016
    config:
    
    	NAME                   STATE     READ WRITE CKSUM
    	vol1                   ONLINE       0     0     0
    	  raidz1-0             ONLINE       0     0     0
    	    35000c500236004a3  ONLINE       0     0     0
    	    35000c50023614aef  ONLINE       0     0     0
    	    35000c5007772e5ff  ONLINE       0     0     0
    	  raidz1-1             ONLINE       0     0     0
    	    35000c500236061b3  ONLINE       0     0     0
    	    35000c5002362f347  ONLINE       0     0     0
    	    35000c500544508b7  ONLINE       0     0     0
    	  raidz1-2             ONLINE       0     0     0
    	    35000c500236032f7  ONLINE       0     0     0
    	    35000c50023605c1b  ONLINE       0     0     0
    	    35000c5002362ffab  ONLINE       0     0     0
    	logs
    	  35000a7203008de44    ONLINE       0     0     0
    	cache
    	  35000cca0130c0a00    ONLINE       0     0     0
    	spares
    	  35000c500236031ab    AVAIL

**Note**: *While the above shows a 3 x RAIDZ1 pool, the specifics of the design and layout depend on the application, redundancy and performance requirements. ZFS mirrors or RAID2/3 are always options. For this particular setup though, 9 x 7200 RPM SAS disks in the 3 x 3 setup [consistently outperformed](http://i.imgur.com/SeSlmH8.png) a group of 10 disks arranged in mirrors.*

**Recommended ZFS filesystem settings for EL7.x**

There are some ZFS filesystem settings that make sense to set at the pool level when working with RHEL/CentOS 7.x.

Disable access times.

    zfs set atime=off vol1

Change acltype to enable filesystem ACLs.

    zfs set acltype=posixacl vol1

Enable `xattr=sa` workaround for [file space reclamation bug](https://github.com/zfsonlinux/zfs/issues/1548) under SELinux and to prevent certain performance issues in edge cases.

    zfs set xattr=sa vol1

I enable [lz4 compression](http://cyan4973.github.io/lz4/) on almost all ZFS filesystems by default. I set this at the pool level so descendant and automatically-created filesystems inherit the property. The impact is low; essentially trading CPU for storage efficiency.

    zfs set compression=lz4 vol1

**Cluster initialization:**

The EL6/EL7 Corosync and Pacemaker packages install a service account named "hacluster".  Assign a password to the user. This will be used later for the cluster authorization and the management GUI.

_Set this password on each cluster node!_

    passwd hacluster

Enable and start the Pacemaker and Corosync system services.

    systemctl enable pcsd
    systemctl enable corosync
    systemctl enable pacemaker
    systemctl start pcsd
    
Authorize the cluster nodes. 
Here, I use the name "zfs-cluster", but choose something appropriate.

    pcs cluster auth zfs-node1 zfs-node2
    pcs cluster setup --start --name zfs-cluster zfs-node1,zfs-node1-ext zfs-node2,zfs-node2-ext

Set the quorum policy to "ignore" since this is a two-node cluster:

    pcs property set no-quorum-policy=ignore

Create a STONITH device using SCSI reservations of the ZFS pool disks. List all of the devices that should be associated with this specific pool. 

    pcs stonith create fence-vol1 fence_scsi pcmk_monitor_action="metadata" pcmk_host_list="zfs-node1,zfs-node2" devices="/dev/mapper/35000c500236061b3,/dev/mapper/35000c500236032f7,/dev/mapper/35000c5007772e5ff,/dev/mapper/35000c50023614aef,/dev/mapper/35000a7203008de44,/dev/mapper/35000c500236004a3,/dev/mapper/35000c5002362ffab,/dev/mapper/35000c500236031ab,/dev/mapper/35000c50023605c1b,/dev/mapper/35000c500544508b7,/dev/mapper/35000c5002362f347" meta provides=unfencing 

Create a ZFS pool resource to correspond to the zpool configuration. The pool name here is "vol1".

    pcs resource create vol1 ZFS pool="vol1" importargs="-d /dev/mapper/" op start timeout="90" op stop timeout="90" --group=group-vol1

Define a virtual IP that floats between nodes. This is associated with the ZFS pool. 
A benefit is that this gives you the option to run dual-active clustering with mutual failover by pinning a ZFS pool to a head node.

    pcs resource create vol1-ip IPaddr2 ip=192.168.77.18 cidr_netmask=24 --group group-vol1

Set a default "stickiness" value to prevent flapping between nodes after a failover event.

    pcs resource defaults resource-stickiness=100

The result from the `pcs status` command should look like:

    [root@zfs1-1 ~]# pcs status 
    Last change: Tue Jul 19 19:29:16 2016 by root via crm_attribute on zfs1-1 Stack: corosync Current DC: zfs1-2 (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum 2 nodes and 3 resources configured
    
    Online: [ zfs1-1 zfs1-2 ]
    
    Full list of resources:
    
     fence-vol1	(stonith:fence_scsi):	Started zfs1-1  Resource Group: group-vol1
         vol1	(ocf::heartbeat:ZFS):	Started zfs1-1
         vol1-ip	(ocf::heartbeat:IPaddr2):	Started zfs1-1
    
    PCSD Status:   zfs1-1: Online   zfs1-2: Online
    
    Daemon Status:   corosync: active/enabled   pacemaker: active/enabled  pcsd: active/enabled

**NFS export**

The elegance of this design comes from eschewing the traditional NFS cluster resources and instead relying on the ZFS `sharenfs` property for exports. 

The old way of doing this would mean having separate resources for:

- The ZFS zpool
- The zpool virtual IP
- The NFS server daemon
- An exportfs resource for _each_ NFS server export
- An NFS notify resource to inform clients during failover
- And Pacemaker cluster constraints to maintain the ordering of the above resources.

In this design, we just use a fencing/STONITH resource, a resource for the zpool and the virtual IP.

For NFS exports, just define the `sharenfs` property on the filesystem(s) you wish to export.

	zfs set sharenfs=rw=@192.168.77.0/24,sync,no_root_squash,no_wdelay vol1/management

This condenses the NFS daemon, exports and NFS notify steps into the zpool export/import process and speeds up failover times.

Here's a demonstration of the following actions on the array while a CentOS VMware virtual machine is being installed. The average failover time is 7 seconds.

- Controlled [failover from node1 to node2](https://player.vimeo.com/video/178573023?autoplay=1#t=0m3s) (standby).
- Controlled [failover from node2 to node1](https://player.vimeo.com/video/178573023?autoplay=1#t=0m36s) (unstandby/standby).
- [Failback from node1 to node2](https://player.vimeo.com/video/178573023?autoplay=1#t=1m5s).
- Hard [power-off of node2](https://player.vimeo.com/video/178573023?autoplay=1#t=1m36s) and automated failover to node1.

VMware and the virtual machine are unaffected.

*[Video: ZFS HA Cluster failover under VMware.](https://vimeo.com/178573023)* 
<a href="https://player.vimeo.com/video/178573023?autoplay=1" target=_blank>![ZFS HA Cluster failover examples under VMware](http://i.imgur.com/dacuzO6.png)</a>


Tuning
------

(in progress)

- `tuned` profiles.
- 4k advanced sector disks.
- ZFS filesystem parameters.
- `/etc/modprobe.d/zfs.conf` options.
- [Multiple queue I/O scheduling](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/7.2_Release_Notes/storage.html) under EL7.2+.

Operations
----------

(in progress)

**Monitoring**

- [zfswatcher](http://zfswatcher.damicon.fi) for ZFS filesystem and pool status/health.
- [netdata](http://netdata.firehol.org) for realtime system analytics.
- [M/Monit](https://mmonit.com) and [Monit](https://mmonit.com/monit/) for thresholding, daemon management and alerts.
- [New Relic](https://newrelic.com/server-monitoring) for external monitoring and thresholding.

**CLI operations**

    # Get cluster status
    pcs status

	# Place cluster node in standby (manual failover)
    pcs cluster standby [node name]

	# Remove cluster node from standby state
    pcs cluster unstandby [node name]

	# Clean up cluster resources
    pcs resource cleanup

**Web GUI**

The Red Hat High Availability Add On also presents a Web GUI for cluster management.
This is enabled by default and is accessible via https port 2224 on either cluster node.

https://nodename:2224

Log on using the `hacluster` account and corresponding password.
Add the nodes by hostname.
Most cluster monitoring and management functions are available here, and it's possible to initiate planned failover (for maintenance) using this interface.

[![enter image description here](http://i.imgur.com/iCJ6gls.png)](http://i.imgur.com/iCJ6gls.png)
