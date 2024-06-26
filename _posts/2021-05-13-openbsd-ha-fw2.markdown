---
layout: post
title:  "High Availability Router/Firewall using OpenBSD part 2... Now with VLANs!"
date:   2021-05-13 11:31:02 -0400
tags: openbsd carp pfsync ifstate pf firewall router vlans
categories: networking openbsd
---
{:refdef: style="text-align: center;"}
![OpenBSD Banner](/images/banner1.gif)
{: refdef}

<br>

## Introduction
I installed OpenBSD on a Soekris net5501-70 to serve as my home firewall/router back in 2012. It continues to run rock solid to this day, but is limited in its capability. I set out to replace it in 2019, but ended up pairing them for high availability. The more powerful device functions as the primary gateway and the Soekris serves as a hot standby that automatically takes over if the primary goes down. The specifics of the original implementation are documented in part one of this series. I mentioned in the [first article]({% link _posts/2019-03-19-openbsd-ha-fw.markdown %}) that I planned to add a managed switch and vlans because I wanted greater isolation from appliances, IOT devices, guest networks, etc. This is the story of that endeavor.

<br>

## Challenges, bottlenecks and missing features
In early 2020, I bought a used Cisco Catalyst WS-C2960-48-TS-S and segmented my network into multiple vlans. That model of Catalyst is technically a Layer 2-only switch (Caveat: IOS 12.2(55)SE and newer support limited Layer 3 functionality such as static routes). I thought this was fine though since OpenBSD has support for vlans, is very easy to configure and I wanted to try my hand at using it for that purpose anyway. I configured each router to have an IP and also a shared CARP interface to serve as the primary gateway for each vlan. Unfortunately this meant it was handling all routing through it’s 1Gbps interface (or 100Mbps in the case of the secondary) as a [router-on-a-stick](https://en.wikipedia.org/wiki/Router_on_a_stick). Despite that less than optimal design, performance was adequate for my home network so I ran it like this for a while.

In addition to the router-on-a-stick induced bottleneck, the OpenBSD routers both being connected to all vlans was creating some challenges. For example, I wanted to restrict access to all the services provided by the router (e.g. SSH, DNS, NTP, etc.) to their respective vlan50 or 192.168.50.0/24 network addresses. Up until this point, each subnet would access the router’s services via the interface on that subnet (e.g. DNS was running on 192.168.100.3:53 for vlan100, but I wanted it to only be accessible from 192.168.50.3:53 for all vlans). Allowing access on only a single subnet would give me a smaller footprint to monitor activity for security purposes. Unfortunately the multihomed design caused a problem when I wanted to create a TCP session with the secondary router’s vlan50 interface from the vlan100 or 192.168.100.0/24 client network. Traffic had to go through the primary router to access the secondary, which would properly send them off, but it never saw the responses since the secondary was directly connected to the requesting network as well. This meant the secondary would send responses directly out its own vlan100 interface rather than back the way it received them (i.e. the primary router’s vlan50 IP). Therefore the primary would eventually drop the connection due to timeout on the entry in the pf state table. The same was true if we had the primary route connections destined for the secondary to the secondary IP directly. The following diagram provides a visual of the problem and also some pf.conf code that would have solved it.


<br>

![Network Topology Problem](/images/ha-fw-topology-problem.png)

*2024-06-24 Update:* In hindsight, I don't believe the pf code in the diagram would have actually solved the problem. I think it would have actually required the use of [rdr-to, received-on, and nat-to as desribed in the OpenBSD FAQ](https://www.openbsd.org/faq/pf/rdr.html#rdrnat).

 
<br>

Fast forward to the end of 2020, my wife and I decided to build a new house. The house is a sprawling single level ranch, so multiple WiFi Access Points would be best to ensure adequate coverage. I went with a couple of Ruckus R610s since they have great performance, support a single management UI when using their Unleashed firmware, and I found them for a great price. I am having the builders drop a few Cat-6 cables in the ceiling so I can just plug them in and go. The only problem is the Catalyst does not support Power-Over-Ethernet (POE) so it needed to be replaced.

I found a used Brocade ICX6450-48P (also made by Ruckus), which has support for POE. It also has 2x10Gbps SPF+ ports in addition to 2x1Gbps SPF+ ports. An improvement over the Catalyst, which only had two 1G SPF ports. More on that in a bit. It also had both Layer 2 and Layer 3 capabilities, which meant I could remove the vlans from my OpenBSD firewalls. That allowed for a much simpler configuration on the firewalls, but also significantly improved performance since the routing between the vlans was handled in hardware on the switch. 

I certainly could have made it function properly using OpenBSD router-on-a-stick, but I now have a server in my dmz vlan connecting to the Brocade with a 10Gbps connection. I plan to stream and store multiple 4K security camera feeds to this device, mirror and record IDS traffic and serve media from this host at a minimum so I wanted a large pipe connected to it. As mentioned earlier, that connection would have been limited if I routed across vlans through the 1Gbps router interfaces. The best answer was to remove the vlans from the OpenBSD routers and instead have the L3 switch handle the inter-vlan routing since it has a 176 Gbps switching capacity and 132 Mpps forwarding capacity.

<br>

## Network Topology 4.0
The following diagram shows how everything is physically wired together along with some logical details. The routers still each have a physical connection for WAN access (only one active at a time) and a separate interface for transferring the pf state table between one another. The biggest difference from part 1 is they now only have a single physical connection for internal access and one shared CARP interface that all devices route through to the internet. One benefit of this design to note is that should the switch ever die on me, I can temporarily flatten the network by connecting any unmanaged switch that I have laying around and have everything just work since the OpenBSD gateways are not configured to require vlan tagging.

There are now two WiFi Access Points that support three different subnets via separate SSIDs that are isolated by vlans. Also, the AP is configured so devices on the IOT network are isolated from talking to anything else on the same subnet. Each AP has a unique IP, but the Unleashed firmware allows for an additional shared IP to manage everything from one interface. 

In addition, the internal switch is now a Layer 3 managed switch with multiple vlan interfaces that act as the gateway for each subnet and handles all inter-vlan routing. The OpenBSD gateways generally only handle anything that leaves the network (Caveat: they route anything on the vlan50 network headed for an internal subnet to the L3 switch interface so it can pass it along to the appropriate device).  ACLs were also added to control inter-vlan access and every device routes through its vlan50 interface to access the internet by way of the highly available OpenBSD systems.

<br>

![Network Topology 4.0](/images/ha-fw-topology-4.0.png)

<br>

## Configuration changes since Part 1
As was noted above, the OpenBSD systems only use one physical link for internal network traffic now so vr2 and em2 are no longer used.


### Soekris (R1)
/etc/hostname.vr3
```
description "Server Network"
inet 192.168.50.2 255.255.255.0 192.168.50.255
!route add 172.16.50.0/24 192.168.50.5
!route add 192.168.100.0/24 192.168.50.5
!route add 192.168.200.0/24 192.168.50.5
!route add 192.168.250.0/24 192.168.50.5
```

Now that the Brocade is handling the inter-vlan routing, the OpenBSD machine no longer connects to those subnets. Therefore it doesn’t know where they are and wants to route through the WAN to get there so we need to tell it to reach those networks via the switch instead (192.168.50.5). We do that by calling the “route add” command in the hostname.vr3 file so the static route is persisted across reboots.

<br>

/etc/hostname.carp50
```
vhid 50 carpdev vr3 pass serverpasswd advskew 100 192.168.50.1 255.255.255.0
```

I renamed the CARP device to carp50 and changed the IP/subnet it sits on.

<br>

/etc/ifstated.conf
```
# Initial State
init-state auto

# Macros
if_master="carp50.link.up"
if_backup="carp50.link.down"


state auto {
        if $if_backup {
                set-state backup
        }
        if $if_master {
                set-state master
        }
}

state master {
        init {
		# Delete slave anchors in pf
		run "pfctl -a slave -F all"

		# Clean up stale routes; dhclient will create default route.
		run "/sbin/route -qn delete default"

		# Renew the ip lease - hopefully stays the same, for pfsync.
		run "/sbin/dhclient vr0"

		# Enable ddclient so it will monitoring for ip changes
		run "rcctl enable ddclient"
		run "rcctl start ddclient"

		# notify root whenever master changes
	    	run "echo master firewall is now `hostname` | mail -s 'carp master changed' root@localhost"

        }
	if $if_backup {
                set-state backup
        }
}

state backup {
        init {
		# This process should be terminated, first.
		run "/usr/bin/pkill -9 dhclient"

		# Delete IP, reset mac and bring wan if down
		run "/sbin/ifconfig vr0 delete"

		# Clean up stale routes and arp cache
		run "/sbin/route -qn delete default"

		# Allows us out to internet via the master host.
		run "/sbin/route -qn add default 192.168.50.3"

		# Load slave anchors in pf
		run "pfctl -a slave -f /etc/pf.slave"

		# Enable ddclient so it will monitoring for ip changes
		run "rcctl stop ddclient"
		run "rcctl disable ddclient"

        }
        if $if_master {
                set-state master
        }
}
```


In the original ifstated.conf the entire route table was flushed for the host when primary/secondary state was changed.. This had the side effect of clearing the static routes that were just set above for the internal vlans. Therefore rather than flushing all, we now just delete the default route instead. 

<br>

### PC Engines (R2)
/etc/hostname.em3
```
description "Server Network"
inet 192.168.50.3 255.255.255.0 192.168.50.255
!route add 172.16.50.0/24 192.168.50.5
!route add 192.168.100.0/24 192.168.50.5
!route add 192.168.200.0/24 192.168.50.5
!route add 192.168.250.0/24 192.168.50.5
```

/etc/hostname.carp50
```
vhid 50 carpdev em3 pass serverpasswd 192.168.50.1 255.255.255.0
```


/etc/ifstated.conf
```
# Initial State
init-state auto

# Macros
if_master="carp50.link.up"
if_backup="carp50.link.down"

if_link_up="em0.link.up && em3.link.up"
if_link_down="em0.link.down || em3.link.down"


state auto {
	if $if_link_down {
                set-state backup
        }
        if $if_master {
                set-state master
        }
        if $if_backup {
                set-state backup
        }
}

state master {
        init {
		# Delete slave anchors in pf
		run "pfctl -a slave -F all"

		# Clean up stale routes; dhclient will create default route.
		run "/sbin/route -qn delete default"

		# Renew the ip lease - hopefully stays the same, for pfsync.
		run "/sbin/dhclient em0"

		# Enable ddclient so it will monitoring for ip changes
		run "rcctl enable ddclient"
		run "rcctl start ddclient"

		# notify root whenever master changes
	    	run "echo master firewall is now `hostname` | mail -s 'carp master changed' root@localhost"

        }
	if $if_link_down {
		run "/sbin/ifconfig -g carp carpdemote"
        }
	if $if_backup {
                set-state backup
        }
}

state backup {
        init {
		# This process should be terminated, first.
		run "/usr/bin/pkill -9 dhclient"

		# Delete IP, reset mac and bring wan if down
		run "/sbin/ifconfig em0 delete"

		# Clean up stale routes and arp cache
		run "/sbin/route -qn delete default"

		# Allows us out to internet via the master host.
		run "/sbin/route -qn add default 192.168.50.2"

		# Load slave anchors in pf
		run "pfctl -a slave -f /etc/pf.slave"

		# Enable ddclient so it will monitoring for ip changes
		run "rcctl stop ddclient"
		run "rcctl disable ddclient"

        }
        if $if_link_up {
		run "/sbin/ifconfig -g carp -carpdemote"
        }
        if $if_master {
                set-state master
        }
}
```

<br>

### Virtual LANs

The switch and the wireless access points are the only devices that know anything about the vlans. The APs only know that they need to tag traffic from each SSID with a respective vlan ID. Three vlans are currently available directly over WiFi via separate SSIDs: vlan100, vlan200, and vlan250. 

The switch handles the actual management of the vlans. It controls which ports are associated with which vlan and whether they are tagged or untagged traffic. As mentioned previously, the switch also has IP’s on each vlan, which act as the default gateway for every subnet except for vlan50. The latter is because vlan50 is used to reach the internet therefore the CARP IP is the default gateway for that subnet. 

Here is a breakdown of the current vlans:

- vlan50
  - The server/management network offering services such as SSH, DNS, DHCPD, etc.
- vlan100
  - Contains our own “trusted” laptops, tablets, and phones
- vlan200
  - Where I place untrusted devices such as game consoles, tvs, home appliances, & security systems, etc.
- vlan250
  - Provides access for guest devices 
- vlan172 
  - This is the “DMZ” network which contains a server I intend to access from anywhere, but is not yet accessible from the internet.

<br>

One thing to note here is that DHCP only works on a single L2 broadcast domain for obvious reasons. This means that I either had to run DHCP on the switch itself or use a DHCP-relay to forward to the vlan50 network so the OpenBSD servers could see and respond to the requests. The switch can handle both options, but I opted to keep using the OpenBSD servers so I can later integrate DHCP hosts with the DNS servers (not to mention dhcpd is much easier to configure and already set up and managed in Ansible). This required the use of the “ip helper” command on the switch. When enabled, the switch sees the DHCP broadcast and converts them to a unicast request, sends them directly to the DHCP servers and returns the response to the requester.

<br>

### ACLs
The whole point of separate subnets was to prevent things like my TVs from being able to initiate contact with my client machines. Since the OpenBSD firewalls were no longer responsible for inter-vlan routing, I needed to create ACLs on the switch to restrict access between vlans now that pf wasn’t standing between them. This isn’t a tutorial on Brocade switches though, so I am not going to post the commands used or ACLs configured. Rest assured access is tightly controlled between the vlans to only allow access to specific local services while also permitting global internet access. Please note that [Ruckus has clear documentation available online](https://support.ruckuswireless.com/documents) and [Terry Henry posts video walkthroughs on Youtube](https://www.youtube.com/channel/UCdRaXJBx3VZj_cJNbfBCEQA) if you would like to learn more about it. They both helped me greatly. That said, I am happy to provide assistance or answer questions so please feel free to reach out.

<br>

## References
[hostname.if manpage](https://man.openbsd.org/hostname.if)  
[Ruckus Documentation](https://support.ruckuswireless.com/documents)  
[Terry Henry's Youtube Channel]( https://www.youtube.com/channel/UCdRaXJBx3VZj_cJNbfBCEQA)  
[https://www.shanekillen.com/2014/10/brocade-switch-dhcp-and-ip-helper.html](https://www.shanekillen.com/2014/10/brocade-switch-dhcp-and-ip-helper.html)  

