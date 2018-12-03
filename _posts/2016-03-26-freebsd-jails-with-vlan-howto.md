---
layout: post
title: FreeBSD Jails with VLAN HOWTO 
date: 2016-03-26 19:54:17.000000000 -07:00
categories: [articles]
tags: [freebsd, networking]
published: true
copyright: Copyright (c) Shawn Debnath. All rights reserved.
license: CC-BY-NC-SA https://creativecommons.org/licenses/by-nc-sa/4.0/
---

_Last Updated: May 4, 2016_

This article discusses how to set up jails on a FreeBSD 11-CURRENT system utilizing VIMAGE (aka VNET) to provide a virtualized independent network stack with support for broad stroke VLAN tagging for each jail. Steps covered here do not employ the use of jail management frameworks, such as, iocage or ez-jail. This allows one to understand the underlying process and be able to fine tune it as needed.

_A separate article will cover optimizations that can be utilized by virtue of being on ZFS._

## Prerequisites

* You have a machine installed with FreeBSD 11-CURRENT on ZFS.
* We will be building world and kernel and using that as the base for the jails. Hence basic knowledge of FreeBSD system administration is assumed. If you've never compiled and installed a FreeBSD base system and kernel, this article may be hard to follow. Refer to the FreeBSD Handbook, especially chapter 8: 'Configuring the FreeBSD Kernel' and chapter 23: 'Updating and Upgrading FreeBSD'.

## Assumptions

* Your host's ethernet interface is _em0_.
* Your IP network is _192.168.6.0/24_ with gateway at _192.168.6.1_.
* The host will be assigned IP _192.168.6.66_.
* The guest jails will be assigned IPs in the range _192.168.6.100-254_.
* VLAN ID for all network interfaces will be _6_.
* Jails will be stored in ZFS datasets under _/jail_ directory.

## Configure the host system

Steps include:

* Rebuild world and kernel
* Update rc.conf
* Enable jails related sysctl variables
* Configure jails support

### Rebuild world and kernel
On the host system, we need to re-compile the kernel to include EPAIR(4) and IF_BRIDGE(4) devices and the VIMAGE option to enable virtualized network stack capabilities. We also enable NULLFS so that we can mount ports and the src tree inside jails.

    # Virtual networking for jail
    options         VIMAGE
    device          epair
    device          if_bridge
    
    # Enable nullFS to mount src and port directories
    options         NULLFS

It's not a bad idea to fetch the latest 11-CURRENT sources to get the latest fixes and rebuild and install world along with kernel at this point.

### Update rc.conf
First, let's set up the networking configuration.  To enable VLANs, we need to create a cloned interface to set the VLAN parameters. We will also create a bridge that the paired networked interfaces for the jails will be trunked on.

    ifconfig_em0="up"
    cloned_interfaces="vlan0 bridge0"
    ifconfig_vlan0="inet 192.168.6.66 netmask 255.255.255.0 vlan 6 vlandev em0"
    defaultrouter="192.168.6.1"
    ifconfig_bridge0="addm vlan0 up"

This setup will enable the host to access the network via _vlan0_ interface through the _em0_ physical adapater. Each paired interface will allow the guest to access the network by having its traffic flow to _vlan0_ via _bridge0_. As an added benefit, any traffic on the system, host and jails included, automatically get tagged with VLAN 6. _See note at the end of the article for alternate configuration._

Next, we need to enable jails support:

    jail_enable="YES"
    jail_list=""
    
We also need to restrict some of the services on the host so that they don't interfere with the jails:

    dumpdev="AUTO"                  # Set to AUTO to enable crash dump, otherwise NO
    rpcbind_enable="NO"             # Disable the RPC daemon
    cron_flags="$cron_flags -J 15"  # Prevent jails running cron jobs at same time
    syslogd_flags="-ss"             # Disable syslogd listening for incoming conns
    sendmail_enable="NONE"          # Completely disable sendmail
    clear_tmp_enable="YES"          # Clear /tmp at startup
    inetd_flags="-wW -a $EXT_IP"    # Restrict inetd to not interfere with jails

### Enable jails related sysctl variables
To enable bridge to work with epair devices, a few settings need to be enabled via sysctl:

    net.inet.ip.forwarding=1
    net.link.bridge.pfil_onlyip=0
    net.link.bridge.pfil_bridge=0
    net.link.bridge.pfil_member=0
    security.bsd.unprivileged_read_msgbuf=0
    security.jail.sysvipc_allowed=1

### Configure jails support
FreeBSD implements jails v2 which requires configuration to be in _/etc/jail.conf_. To get started, we will create _/etc/jail.conf_ with the default configuration applicable to all jails:

    # Jail configuration - /etc/jail.conf
    
    allow.raw_sockets = "0";
    allow.set_hostname = "0";
    allow.sysvipc = "1";
    allow.mount.devfs;
      
    host.hostname  = "${name}";
    path  = "/jail/${name}";
    mount.fstab  = "/etc/jails/fstab.${name}";
    mount.devfs  = "1";
    devfs_ruleset  = "4";
    exec.consolelog  = "/var/log/jail_${name}_console.log";

Jail specific configuration will be added to _/etc/jail.conf_ as we create each jail below.

Each jail will have it's own fstab which will be placed under _/etc/jails/_, so  create that directory:

    sudo mkdir /etc/jails

Reboot your system or restart networking manually:

    sudo service netif restart
    sudo service routing restart

Running _ifconfig_ should result in output similar to this (_lo0 omitted for brevity_):

    em0:flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=42098<VLAN_MTU,VLAN_HWTAGGING,VLAN_HWCSUM,WOL_MAGIC,VLAN_HWTSO>
        ether 68:05:ca:36:3c:26
	    nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
	    media: Ethernet autoselect (1000baseT <full-duplex>)
	    status: active
    
    vlan0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	    ether 68:05:ca:36:3c:26
	    inet 192.168.6.66 netmask 0xffffff00 broadcast 192.168.6.255 
	    nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
	    media: Ethernet autoselect (1000baseT <full-duplex>)
	    status: active
	    vlan: 6 parent interface: em0
	    groups: vlan

    bridge0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	    ether 02:0b:47:0d:33:00
	    nd6 options=9<PERFORMNUD,IFDISABLED>
	    groups: bridge
	    id 00:00:00:00:00:00 priority 32768 hellotime 2 fwddelay 15
    	maxage 20 holdcnt 6 proto rstp maxaddr 2000 timeout 1200
	    root id 00:00:00:00:00:00 priority 32768 ifcost 0 port 0
	    member: vlan0 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
	            ifmaxaddr 0 port 1 priority 128 path cost 2000000

Here's what the routing table looks like:

    $ netstat -r
    Routing tables

    Internet:
    Destination        Gateway            Flags     Netif Expire
    default            192.168.6.1        UGS       vlan0
    localhost          link#3             UH          lo0
    192.168.6.0/24     link#4             U         vlan0
    192.168.6.66       link#4             UHS         lo0

## Create the jail

For demonstration purposes, we will create a jail called webserver whose intent is to run a web server. The installation and configuration of the web server is out of scope for this guide.

### Create ZFS dataset for the jail

We will create a ZFS dataset with quota for 2GB (_substitute your zpool name accordingly_):

    sudo zfs create zroot/webserver
    sudo zfs set mountpoint=/jail/webserver zroot/webserver
    sudo zfs set quota=2g zroot/webserver

### Populate filesystem

We will use the pre-built system binaries to populate our jail as follows:

    cd /usr/src
    sudo make installworld DESTDIR=/jail/webserver
    sudo make distribution DESTDIR=/jail/webserver

If you chose to use a release distribution for the jail instead of 11-CURRENT binaries, extract the content from an appropriate ISO image accordingly. You may also wish to build the binaries from sources other than 11-CURRENT.

### Create the jail's rc.conf

We set up the jail's rc.conf similar to the host:

    hostname="webserver"
    sendmail_enable="NONE"          # Completely disable sendmail
    inetd_flags="-wW -a 192.168.6.100"    # Restrict inetd to not interfere with jails
    rpcbind_enable="NO"             # Disable the RPC daemon
    cron_flags="$cron_flags -J 15"  # Prevent jails running cron jobs at same time
    syslogd_flags="-ss"             # Disable syslogd listening for incoming conns

### Create the jail's fstab on host

Create a new file _/etc/jails/fstab.webserver_ and set it's contents to:

    /usr/src              /jail/webserver/usr/src            nullfs      rw     0     0
    /usr/ports            /jail/webserver/usr/ports          nullfs      rw     0     0

### Create the jail specific configuration in jail.conf

Finally, we need to set up the jail specific configuration in jail.conf. Here, we are assigning _192.168.6.100_ to the jail.  We specify that the jail will be using its own virtual networking stack with the _vnet_ command and set the vnet interface is set to _epair100b_. Since the traffic flows through the bridge to the vlan0 interface on the host, all packets are automatically tagged with our VLAN ID of 6. There is no need for additional VLAN configuration within the jail.

    # VIMAGE jail with upnp-based dlna server with its own NIC address
    webserver {
      $if  = "100";
      $ip_addr  = "192.168.6.${if}";   # Jail ip address
      $ip_route  = "192.168.6.1";  # Gateway or host's ip address
      $mask = "255.255.255.0";  # Netmask
      vnet;
      vnet.interface  = "epair${if}b";
    
      # Commands to run on host before jail is created
      exec.prestart  = "ifconfig epair${if} create up";
      exec.prestart  += "ifconfig epair${if}a up";
      exec.prestart  += "ifconfig bridge0 addm epair${if}a up";
    
      # Commands to run in jail after it is created
      exec.start  = "/sbin/ifconfig lo0 127.0.0.1 up";
      exec.start  += "/sbin/ifconfig epair${if}b up";
      exec.start  += "/sbin/ifconfig epair${if}b ${ip_addr} netmask ${mask} up";
      exec.start  += "/sbin/route add default ${ip_route}";
      exec.start  += "/bin/sh /etc/rc";
    
      exec.stop  = "/bin/sh /etc/rc.shutdown";
      exec.poststop  = "ifconfig bridge0 deletem epair${if}a";
      exec.poststop  += "ifconfig epair${if}a destroy";
      persist;
    }

Notes:

* Prior to the jail being created (_see exec.start_), we create the _epair(4)_ interfaces and add the _epair100a_ interface to _bridge0_ on the host. This is the virtualized network adapter for the jail.
* Once the jail is up (_see exec.start_), we activate the _lo0_ and _epair100b_ interfaces.

## Start the jail

    sudo service jail start webserver

If all went well, you should see the jail listed (not IP Address is not shown for VNET based jails):

    $ jls
       JID  IP Address      Hostname                      Path
         1                  webserver                     /jail/webserver

On the host, ifconfig should report the newly created _epair100a_ interface along with it being a member of _bridge0_:

    bridge0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST>     metric 0 mtu 1500
        ether 02:0b:47:0d:33:00
        nd6 options=9<PERFORMNUD,IFDISABLED>
        groups: bridge
        id 00:00:00:00:00:00 priority 32768 hellotime 2 fwddelay 15
        maxage 20 holdcnt 6 proto rstp maxaddr 2000 timeout 1200
        root id 00:00:00:00:00:00 priority 32768 ifcost 0 port 0
        member: epair100a flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
            ifmaxaddr 0 port 7 priority 128 path cost 2000
        member: vlan flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
            ifmaxaddr 0 port 1 priority 128 path cost 2000000
    
    epair100a: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST> metric 0 mtu 1500 options=8<VLAN_MTU>
        ether 02:ff:70:00:07:0a
        inet6 fe80::ff:70ff:fe00:70a%epair100a prefixlen 64 scopeid 0x7
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
        media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
        status: active
        groups: epair
 
 And ifconfig output inside the jail should report the corresponding _epair100b_ interface:

    epair100b: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=8<VLAN_MTU>
        ether 02:ff:c0:00:0a:0b
        inet6 fe80::ff:c0ff:fe00:a0b%epair101b prefixlen 64 tentative scopeid 0x2
        inet 192.168.6.100 netmask 0xffffff00 broadcast 192.168.6.255
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
        media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
        status: active
        groups: epair

## Alternate Configuration
I initially wrote this article detailing my attempt to set up a jail environment in a DMZ. In the original post, I had bridged the em0 interface (without vlan or ip configuration) with the jail epair interfaces via bridge0. I left the responsibility of tagging packets with the VLAN ID to vlan0 on the host and to a separate vlan interface in each jail. @helio on freenode pointed out that if a jail were to be compromised, one could modify the vlan interface within the jail potentially causing havoc. Luckily, all traffic from this host was being routed through smart switches with aggressive ingress and egress filtering so it was not a concern, however, can't ignore best practices.

If you are not in a similar situation, you may want to provide the flexibility of each jail specifying which VLAN it belongs to. In that case, remove vlan0 and add em0 to the bridge. The host traffic will still flow through vlan0 but this makes the phsyical raw interface available to the jails which they can then tag as appropriate. VLAN setup within the jail is similar to that of the host.

