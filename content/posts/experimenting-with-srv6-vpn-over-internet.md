---
date: '2025-07-06T12:51:45+02:00'
draft: false
title: 'Experimenting With Srv6 VPN services Over The Internet'
author: "Anders Ballegaard"
tags:
  - IPv6
  - Srv6
  - Vyos
  - L3VPN
image: /images/content/srv6-vpn/srv6-vpn.png
description: "The start of a series of posts related to building an ipv6 mostly home network and lab"
toc: true
---
Ever since learning about SRv6, I have been interested in testing how SRv6-based VPN services work, especially over an uncontrolled network like the Internet. I happened to have some time and energy to play around with it. This post doesn't describe a production-ready setup; it's just some notes from playing around and figuring out what is possible, how it works, and getting some ideas for future tinkering.

## What is Segment routing and SRv6?
Segment routing is a modern approach to directing traffic. It works over either IPv6 or MPLS, and has many interesting features related to redundancy, traffic engineering, and services.

SRv6 is the IPv6 flavor of segment routing. Unlike SR-MPLS, it works over any IPv6 data plane (although you might want more). This flexibility makes it possible to extend SRv6-based services over the Internet, which is what we're trying to exploit today. The fact that it's just IPv6 also allows devices that traditionally don't support MPLS to be part of the network, like servers, phones, etc., although this isn't generally seen in the real world.

There are a lot of resources to learn more about segment routing; I would recommend starting with ***[segment-routing.net](https://www.segment-routing.net/)***.

## About the test setup
To reduce the number of variables, this test network consists of just two routers. I am running VyOS 2025.07.06-0022-rolling on both routers.

Each router is connected to a diffrent interface on a router inside my personal AS201911 network. The interfaces on the ISP router is quite simple, just a linknet and a static route pointing to the VPN router. 


![image](/images/content/srv6-vpn/srv6-vpn.png)

| Router | WAN Linknet | Routed prefix | Router ID |
|--------|-------------|---------------------|---------------------|
| VPN-Site-A | 2a0e:97c0:ae0:700a::2/64 | 2a0e:97c0:ae6:1000::/56 | 10.1.1.1 |
| VPN-Site-B | 2a0e:97c0:ae0:700b::2/64 | 2a0e:97c0:ae6:2000::/56 | 10.2.2.2 |

Both routers are part of the ASN 65513, and both have a static ipv6 default route configured towards the ISP Router.

## Setting up SRv6
In a more traditional campus/DC/SP deployment scenario, you would start your SRv6 deployment by deploying an SRv6 capable IGP. However since we are going over the internet, we won't have an IGP. So we are instead starting with BGP.

### Setting up BGP between the routers
BGP is already enable on the routers, so we just need to configure peerings, and srv6 options.

Let's start by configuring a peer-group, this should be applied to both routers

There is nothing fancy about this configuration, just a simple iBGP peering, with a password and VPN address families.
```
set protocols bgp peer-group INTERNAL remote-as internal
set protocols bgp peer-group INTERNAL password CorrectHorseBatteryStable
set protocols bgp peer-group INTERNAL address-family ipv4-vpn
set protocols bgp peer-group INTERNAL address-family ipv6-vpn
set protocols bgp peer-group INTERNAL capability extended-nexthop

```

Now let's create the actual peerings between the two routers using the peer group we created above.
In theory we could create a loopback interface inside the routed prefix, and if you have multiple WAN's that might be the best option, but for this example I will just create the BGP peering between the linknet IP's.
```
# On VPN-Site-A
set protocols bgp neighbor 2a0e:97c0:ae0:700b::2 peer-group INTERNAL

# On VPN-Site-B
set protocols bgp neighbor 2a0e:97c0:ae0:700a::2 peer-group INTERNAL
```
And just like that we have a BGP peering with no routes.
![bgp-peering](/images/content/srv6-vpn/bgp-confirmed.png)


### Configuring SRv6
We need to configure the routed prefix we got from the ISP as a SID, besides that we also need to tell SRv6 what interfaces to use.

Let's start by configuring a locator SID for VPN services. For this purpose, i am reserving a prefix inside the routed network.
A small sidenote, in theory you could create this setup on a router that has a DHCPv6-PD prefix, but given this part of the configuration is static, it could easily break.
```
# On VPN-Site-A
set protocols segment-routing interface eth0
set protocols segment-routing srv6 locator VPN-SERVICES behavior-usid
set protocols segment-routing srv6 locator VPN-SERVICES prefix 2a0e:97c0:ae6:1001::/64
set protocols bgp srv6 locator VPN-SERVICES

# On VPN-Site-B
set protocols segment-routing interface eth0
set protocols segment-routing srv6 locator VPN-SERVICES behavior-usid
set protocols segment-routing srv6 locator VPN-SERVICES prefix 2a0e:97c0:ae6:2001::/64
set protocols bgp srv6 locator VPN-SERVICES
```

When a new VPN is created, BGP will take out an address from our VPN-SERVICES prefix and assign that as a destination address for all traffic to the VRF (assuming we want per-vrf "label"). We can either specify the allocation manually, or let BGP figure it out. I feel trusting in BGPs abilities today, so i will auto assign.

### Building our first L3VPN
In theory we should now have a BGP peering, a routed prefix, and an SRv6 locator. So the next step is to try using it.
In this step we will create a VRF, and use that VRF on two dummy interfaces to validate connectivity.

Let's start by defining the VRF

This configuration starts out defining everything that's the same on both sides.
- The linux kernel routing table ID. If you are comming from Cisco or Juniper this is probably new, but just know linux wants a value it can use internally.
- Import/Export route targets. Note after we have created the targets we need to tell BGP that it should use it.
- SID, we are telling BGP to auto allocate a SID.
- For some reason we need to set system-as and router-id again, i just set it to the same as the global options. I guess this could be a problem if you have a design where your GRT and VPN instance have BGP peerings with eachother.
- We are redistributing connected, in this example we are putting the VRF on two dummy interfaces, obiviously if this was a real deployment, we might want to redistribute something diffrent.
- Create a dummy interface and assign it to the VRF.

We also have a few unique things per router, this includes:
- IP addresses
- Router ID
- Route distinguishers.
```
# Shared for both routers
set vrf name L3VPN-1 table 101
set vrf name L3VPN-1 protocols bgp address-family ipv4-unicast route-target vpn both 65513:101
set vrf name L3VPN-1 protocols bgp address-family ipv6-unicast route-target vpn both 65513:101

set vrf name L3VPN-1 protocols bgp address-family ipv4-unicast import vpn
set vrf name L3VPN-1 protocols bgp address-family ipv4-unicast export vpn

set vrf name L3VPN-1 protocols bgp address-family ipv6-unicast import vpn
set vrf name L3VPN-1 protocols bgp address-family ipv6-unicast export vpn

set vrf name L3VPN-1 protocols bgp sid vpn per-vrf export auto
set vrf name L3VPN-1 protocols bgp system-as 65513

set vrf name L3VPN-1 protocols bgp address-family ipv4-unicast redistribute connected
set vrf name L3VPN-1 protocols bgp address-family ipv6-unicast redistribute connected

set interfaces dummy dum101 vrf L3VPN-1
set interfaces dummy dum101 description "L3VPN test interface"

# VPN-Site-A
set interfaces dummy dum101 address 172.16.10.1/24
set interfaces dummy dum101 address 2001:db8:1::1/64
set vrf name L3VPN-1 protocols bgp address-family ipv4-unicast rd vpn export '10.1.1.1:101'
set vrf name L3VPN-1 protocols bgp address-family ipv6-unicast rd vpn export '10.1.1.1:101'
set vrf name L3VPN-1 protocols bgp parameters router-id 10.1.1.1

# VPN-Site-B
set interfaces dummy dum101 address 172.16.20.1/24
set interfaces dummy dum101 address 2001:db8:2::1/64
set vrf name L3VPN-1 protocols bgp address-family ipv4-unicast rd vpn export '10.2.2.2:101'
set vrf name L3VPN-1 protocols bgp address-family ipv6-unicast rd vpn export '10.2.2.2:101'
set vrf name L3VPN-1 protocols bgp parameters router-id 10.2.2.2
```

Now let's see if it worked, let's start by checking to see if a locator has been registered
![locator](/images/content/srv6-vpn/locator-verification.png)
As you can see a /128 has been taken out, pointing to L3VPN-1 with type End.DT46 meaning this single locator is valid for both ipv4 and ipv6.

Now let's check the route table
![Route table](/images/content/srv6-vpn/l3vpn-routes.png)
As you can see, we have routes for both V4 and V6. Now for the fun part, let's try to ping it.
![Ping](/images/content/srv6-vpn/ping.png)
And success!!! We now have a working L3VPN over internet.

But how does that look on the wire?

As you can see, matching on Ipv6's next header 43 (source routing) field, we are seeing both the v4 and v6 pings.
But as you can also see it's unencrypted, In theory this should be solvable with IPsec, you probably just want to make sure the SRH isn't being encrypted.
![Wireshark overview](/images/content/srv6-vpn/wireshark-1.png)

Well traffic is flowing from in this case VPN-SITE-B's Linknet address to the SID we saw VPN-SITE-A had reserved for the L3VPN. Inside the packet we can see the following:
- We have a routing header of type segment routing (type 4)
- we can see there are 0 segments left, in our case we only have 1 segment, but if you added in traffic engineering, more segments could exist.
- We can see our current segment is 2a0e:97c0:ae6:1001:1:: this matches our destination addess. This is exactly how it should be.
- The next header is IPIP this indicates the next packet is an IPv4 packet, if we had looked at one of the IPv6 pings, the next header would have been IPv6.
- We can see the inner IP header is just a normal header we would expect to see between our two hosts inside the VPN.
![Wireshark packet](/images/content/srv6-vpn/wireshark-2.png)

## How can this be used?
The setup described above with only two sites isn't all that interesting from a usecase perspective. What if we had more sites? What if we wanted to route traffic between all the sites? What if we wanted to steer traffic around the internet in special ways? What if we where using hosts instead of routers?

Those are the kind of questions where i think Srv6 becomes very interesting. I might explore how to use SRv6 to create a "poor mans SD-WAN" solution or something like that in the future. 

SRv6 is also very intersting from a host/server perspective, the setup above could also be implemted in a container enviorment like K8S to provide a very flexible k8s overlay network. Infact the Cillium project is already kinda doing that,

# Conclusion
SRv6 is a very powerful technology, while this simple setup didn't acchive anything you couldn't do in a simpler way, i hope it showed what could be posible, and started some thoughts of how we could use SRv6.