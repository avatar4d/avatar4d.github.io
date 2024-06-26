---
layout: post
title:  "High Availability Router/Firewall using OpenBSD"
date:   2019-03-19 11:44:28 -0400
tags: openbsd carp pfsync ifstate pf firewall router
categories: networking openbsd
---
{:refdef: style="text-align: center;"}
![OpenBSD Banner](/images/banner1.gif)
{: refdef}

<br>

## Introduction 

I have been running OpenBSD on a Soekris net5501-70 for my router/firewall since early 2012. Because I run a multitude of services on this system (more on that later), the meager 500Mhz AMD Geode + 512MB SDRAM was starting to get a little sluggish while trying to do anything via the terminal. Despite the perceived performance hit during interactive SSH sessions, it still supported a full 100Mbit connection with NAT so I wasn’t overly eager to change anything. Luckily though, my ISP increased the bandwidth available on my plan tier to 150Mbit+. Unfortunately, the Soekris only contained 4xVIA Rhine Fast Ethernet. So now I was using a slow system and wasting time/money by not being able to fully utilize my connection.
Naturally I looked back to Soekris for an upgrade that would allow me to take advantage of this new speed since it served me well for so long, but soon discovered that Soekris stopped innovating and closed US operations a few years ago. After widening the search, I decided to try the PC Engines APU4C4. This included a 4 Core 1Ghz AMD GX-412TC CPU, 4GB of DDR3-1333 DRAM and 4xIntel PRO/1000 Gigabit Ethernet. A huge improvement.
Now that I had two appliances, I figured why not try setting them up in failover mode with CARP and pfsync. At the very least this would come in handy during the post-patching reboots and  while down for an upgrade every six months. A small win, but at least I could upgrade whenever I wanted without impacting the family’s internet connectivity. On the other hand, should the hardware fail I won't be scrambling and setting up some old hardware or running to buy some cheap consumer product.


Note: While there are a lot of things I still have planned to do with this setup and may already have noted any deficiencies, I welcome suggestions and feedback.

<br>

## New Network Topology 

The following diagram details out how everything is wired together. I split my LAN and Wi-Fi into separate networks to add an additional layer of separation between the secured servers on the hardwire and potential rogues coming in through the wireless. I may eventually flatten this once I feel all services are locked down better, but for now it gives me a better (unwarranted?) sense of security. I plan to eventually replace the three unmanaged switches with a single managed switch with vlans, but that is for a future article.



{:refdef: style="text-align: center;"}
![Network Topology](/images/ha-fw-network-topology.png)
{: refdef}

<br>

## Description 

As I mentioned above, a number of services for the network are provided directly from the router/firewall. While I realize this isn’t necessarily best practice, it is for my home network where I am constrained by budget and compelled by convenience to put them all on the router. Those daemons include:


- dhcpd
```
Listens on vr2, vr3, em2, em3
```

<br>

Since I’m running dhcpd on both routers. which operating on the same subnets, I chose to split the addresses each serves (r2 has last octet of 100-150, r1 has 151-250) rather than trying to do anything magical.
- tftpd
  - Currently listening on vr2 only
Need to do more work here

- opensmtpd (vr2, em2)
  - Listens on vr2, em2
Acts merely as a relay for all LAN servers to personal email server

- openntpd
  - Listens on vr2, em2
Created as a pool in nsd for round-robin client access

- unbound
  - Listens on vr2, vr3, em2, em3
  - Configured for DNSSEC validation and upstream dns-over-tls to Quad9

- nsd
  - Listens on vr2, vr3, em2, em3
  - Hosts personal domain for easy access of local services

- avahi_daemon
  - Listens on vr2, vr3, em2, em3
  - Exposes ZFS array on LAN server to OSX clients for TimeMachine backups

<br>

It should go without saying, but each system of course runs openssh (listening on vr2, vr3, em2 and em3) and requires keys for access (i.e. no password auth). I also use ddclient to keep a domain I own up to date if the ISP DCHP provided address ever changes. In addition, I am using munin for monitoring system trends and pflow for connection tracking with nfsen. Data from the latter two are pushed to and accessible from the server on the LAN.

While much of the above falls outside the scope of this post, I will detail most of the OpenBSD specific configuration files below.

<br>

## Configuration 

I’m using Ansible to manage all of these files so anything highlighted is controlled with jinja2 to ensure the proper config is pushed to the correct system.

<br>

**Caveat:** This is accurate for OpenBSD 6.4

**Caveat 2:** I modified some of the configs below during the creation of this post so there may be mistakes since they are no longer copypasta

<br>

### Both R1 and R2 

/etc/sysctl.conf
```
net.inet.ip.forwarding=1
net.inet.carp.preempt=1
ddb.panic=0
```


/etc/pf.conf
```
#Include Ansible configured interface file so pf.conf can be the same on both routers
include "/etc/pf.if"


#Allow pfsync on link connecting both routers
pass quick on $sync_if proto pfsync


#Allow carp traffic on physical devices carp uses
pass quick on { $wifi_if $lan_if } proto carp

# ifstated controlled anchor reference
anchor slave
```
<br>

<font color="red"><b>Note:</b> you should have more rules than this. I am just explicitly noting the additional rules added for this project.</font>

<br>

### Soekris (R1) 

/etc/boot.conf
```
stty com0 19200
set tty com0
```
<br>

/etc/mygate
```
192.168.0.3
```

I want to give the backup system internet access at boot or my pf.conf rules won’t load so I give it the IP of the other host’s physical link (not the CARP device) via the mygate file. Note: You must ensure your pf rules permit this communication.

<br>

/etc/hostname.vr0
```
down
```

<br>

/etc/hostname.vr1
```
inet 10.0.0.2 255.255.255.0 10.0.0.255
```

<br>

/etc/hostname.vr2
```
inet 192.168.0.2 255.255.255.0 192.168.0.255
```

<br>

/etc/hostname.vr3
```
inet 172.16.0.2 255.255.255.0 172.16.0.255
```

<br>

/etc/hostname.carp2
```
vhid 2 carpdev vr2 pass <carp2 password> advskew 100 192.168.0.1 255.255.255.0
```

This contains the floating vIP. Since this is the backup, I skew the advertisements so the other router becomes master

<br>

/etc/hostname.carp3
```
vhid 3 carpdev vr3 pass <carp3 password> advskew 100 172.16.0.1 255.255.255.0
```

This contains the floating vIP. Since this is the backup, I skew the advertisements so the other router becomes master

<br>

/etc/hostname.pfsync0
```
syncdev vr1
```

This syncs the firewall state table between the routers so connections don't drop.

<br>

/etc/hostname.pflow0
```
flowsrc 192.168.0.2 flowdst <ip of nfsen host>:<port nfsen is listen to> pflowproto 10
```

Sends netflow data for later analysis.

<br>

/etc/ifstated.conf

```
# Initial State
init-state auto


# Macros
if_carp_up="carp2.link.up && carp3.link.up"
if_carp_down="!carp2.link.up || !carp3.link.up"


state auto {
       if $if_carp_up {
               set-state master
       }


       if $if_carp_down {
               set-state backup
       }
}


state master {
        init {
                # Delete slave anchors in pf
                run "pfctl -a slave -F"


                # WAN hostname.if(5) should be started as 'down' with no ipaddr.
                # Spoof MAC so it can get an IP from ISP without a modem reboot
                run "/sbin/ifconfig vr0 lladdr 00:0d:b9:50:11:94 up"


                # Clean up stale routes; dhclient will create default route.
                run "/sbin/route -qn flush"


                # Renew the ip lease - hopefully stays the same, for pfsync.
                run "/sbin/dhclient vr0"


                # Enable avahi so new router can take over TimeMachine backups
                run "rcctl enable avahi_daemon"
                run "rcctl start avahi_daemon"


    # Enable ddclient so it will monitor for ip changes
                run "rcctl enable ddclient"
                run "rcctl start ddclient"


                # notify root whenever master changes
                run "echo master firewall is now `hostname` | mail -s 'carp master changed' root@localhost"
        }


        if $if_carp_down {
                set-state backup
        }
}


state backup {
        init {
                # This process should be terminated, first.
                run "/usr/bin/pkill -9 dhclient"


                # Delete IP, reset mac and bring wan if down
                run "/sbin/ifconfig vr0 delete lladdr de:ad:00:00:be:ef down"


                # Clean up stale routes and arp cache
                run "/sbin/route -qn flush"


                # Allows us out to internet via the master host.
                run "/sbin/route -qn add default 192.168.0.3"


                # Load slave anchors in pf so this system can communicate with the outside world 
    # from LAN interface via the LAN interface on the other router
                run "pfctl -a slave -f /etc/pf.slave"


                # Disable avahi so new router can take over TimeMachine backups.
                # Otherwise avahi will peg cpu
                run "rcctl disable avahi_daemon"
                run "rcctl stop avahi_daemon"


    # Disable ddclient so it will monitor for ip changes
                run "rcctl disable ddclient"
                run "rcctl stop ddclient"
        }




       if $if_carp_up {
               set-state master
       }
}
```

This is the brains of the operation that controls which system is managing the data flow

<br>

/etc/pf.if
```
ext_if="vr0"
sync_if="vr1"
lan_if="vr2"
wifi_if="vr3"
```
<br>

/etc/pf.slave
```
pass on vr2 from vr2:0 to any
```
<br>

### PC Engines (R2)
/etc/boot.conf
```
stty com0 115200
set tty com0
```

<br>

/etc/hostname.em0
```
dhcp
```

<br>

/etc/hostname.em1
```
inet 10.0.0.3 255.255.255.0 10.0.0.255
```

<br>

/etc/hostname.em2
```
inet 192.168.0.3 255.255.255.0 192.168.0.255
```

<br>

/etc/hostname.em3
```
inet 172.16.0.3 255.255.255.0 172.16.0.255
```

<br>

/etc/hostname.carp2
```
vhid 2 carpdev em2 pass <carp2 password> 192.168.0.1 255.255.255.0
```

This contains the floating vIP.

<br>

/etc/hostname.carp3
```
vhid 3 carpdev em3 pass <carp3 password> 172.16.0.1 255.255.255.0
```

This contains the floating vIP.

<br>

/etc/hostname.pfsync0
```
syncdev em1
```

This syncs the firewall state table between the routers so connections don't drop

<br>

/etc/hostname.pflow0
```
flowsrc 192.168.0.3 flowdst <ip of nfsen host>:<port nfsen is listen to> pflowproto 10
```

Sends netflow data for later analysis

<br>

/etc/ifstated.conf
```
# Initial State
init-state auto


# Macros
if_carp_up="carp2.link.up && carp3.link.up"
if_carp_down="!carp2.link.up || !carp3.link.up"


state auto {
       if $if_carp_up {
               set-state master
       }


       if $if_carp_down {
               set-state backup
       }
}


state master {
        init {
                # Delete slave anchors in pf
                run "pfctl -a slave -F"


                # WAN hostname.if(5) started with dhcp upon boot so pf.conf rules will load so this will reissue request.
                # This is actually the MAC of this machine, but since backup replaces with spoof we set it here so it can get an IP from ISP without a modem reboot
                run "/sbin/ifconfig em0 lladdr 00:0d:b9:50:11:94 up"


                # Clean up stale routes; dhclient will create default route.
                run "/sbin/route -qn flush"


                # Renew the ip lease - hopefully stays the same, for pfsync.
                run "/sbin/dhclient em0"


                # Enable avahi so new router can take over TimeMachine backups
                run "rcctl enable avahi_daemon"
                run "rcctl start avahi_daemon"


    # Enable ddclient so it will monitor for ip changes
                run "rcctl enable ddclient"
                run "rcctl start ddclient"


                # notify root whenever master changes
                run "echo master firewall is now `hostname` | mail -s 'carp master changed' root@localhost"


        }


        if $if_carp_down {
                set-state backup
        }
}


state backup {
        init {
                # This process should be terminated, first.
                run "/usr/bin/pkill -9 dhclient"


                # Delete IP, reset mac and bring wan if down
                run "/sbin/ifconfig em0 delete lladdr de:ad:00:00:be:ef down"
         	    
    # Clean up stale routes and arp cache
                run "/sbin/route -qn flush"


                # Allows us out to internet via the master host.
                run "/sbin/route -qn add default 192.168.0.2"


                # Load slave anchors in pf so this system can communicate with the outside world 
    # from LAN interface via the LAN interface on the other router
                run "pfctl -a slave -f /etc/pf.slave"


                # Disable avahi so new router can take over TimeMachine backups.
                # Otherwise avahi will peg cpu
                run "rcctl disable avahi_daemon"
                run "rcctl stop avahi_daemon"


   # Disable ddclient so it will monitor for ip changes
                run "rcctl disable ddclient"
                run "rcctl stop ddclient"
}




       if $if_carp_up {
               set-state master
       }
}
```

This is the brains of the operation that controls which system is managing the data flow

<br>

/etc/pf.if
```
ext_if="em0"
sync_if="em1"
lan_if="em2"
wifi_if="em3"
```
<br>

/etc/pf.slave
```
pass on em2 from em2:0 to any
```

<br>

*2021-05-13 Update:* The design and configuration of this setup has continued to evolve since the initial implementation. Please continue with [Part 2]({% link _posts/2021-05-13-openbsd-ha-fw2.markdown %}) to see what has changed.

<br>

## References
[https://sites.google.com/site/bsdstuff/dhcarp](https://sites.google.com/site/bsdstuff/dhcarp)  
[https://www.openbsd.org/faq/pf/carp.html](https://www.openbsd.org/faq/pf/carp.html)  
[https://man.openbsd.org/carp](https://man.openbsd.org/carp)  
[https://man.openbsd.org/pfsync](https://man.openbsd.org/pfsync)  
[https://man.openbsd.org/ifstated](https://man.openbsd.org/ifstated)

