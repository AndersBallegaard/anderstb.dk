---
date: '2025-07-06T09:05:08+02:00'
draft: false
title: 'NAT64 for the Homelab'
author: "Anders Ballegaard"
tags:
  - IPv6
  - NAT64
  - Homelab
  - Home network
  - Vyos
image: /images/content/ipv6-series/DNS64_flow.png
description: "An comparison of diffrent NAT64 options, and an introduction to NAT64 related concepts"
toc: true
---

As discussed in ***[the previous post](/posts/ipv6-mostly-home-intro/)***, I am currently making some modifications to my homelab. As a part of this process, I am looking at NAT64 solutions again. I am currently running Tayga on OpnSense, but want to move to NAT64 with a dedicated VM. This post will be going through what NAT64 is, how clients interact with it, a comparison of different implementations and finally setting up my chosen implementation.

## What is NAT64 and why do we need it?
We need NAT64 in IPv6 mostly and IPv6 only networks because there are still many sites and services on the internet that don't support IPv6. NAT64 solves this problem by mapping every single IPv4 address to a unique IPv6 address, which can be used for communication with those addresses.

This doesn't magically fix client devices that don't support IPv6, but it enables devices with IPv6 support to start going IPv6 only. Mobile devices, and some desktop operating systems (primarily macOS) support IPv6 only operations particularly well, due to having built-in CLAT implementations. However, we will dive deeper into this later.

## How does NAT64 work?
All NAT64 implementations map an IPv6 address into a /96 IPv6 prefix by taking every single bit of the IPv4 address and adding it to the end of the IPv6 address. This means that for example, an IPv4 address '1.1.1.1' could become '64:ff9b::101:101', or '96.7.128.175' becomes '64:ff9b::6007:80af'.

But where does the 64:ff9b:: come from? Well, you can technically use any /96 IPv6 prefix, but 64:ff9b::/96 is reserved to NAT64. Using 64:ff9b::/96 does have some pros and cons:
- If you want to use publicly avalible DNS64 services, this is the prefix they assume your NAT64 implementation will be using.
- It is obvious that traffic is going through NAT64 if you see an 64:ff9b::/96 address.
- Some NAT64 implementations might not allow translating traffic to RFC1918 destinations, if you are using 64:ff9b::/96

There can be some diffrences between NAT64 implementations, but we will look more at that in the comparison section below. For homelab purposes i would also argue it makes quite a diffrence if you are managing the NAT64 software directly, or if you are using it as part of an intigrated solution like running NAT64 in OpnSense.

## Client interaction with NAT64
It might be worth briefly looking at how clients interact with NAT64 before looking at the solutions themself. The two main ways are DNS64 and CLAT (also known as 464XLAT), which are not mutually exclusive, but can be used in combination.

### DNS64
DNS64 essentially works by lying to the client, The DNS server sends A and AAAA queries for a given domain. If no AAAA record is found, it maps the A record address into a NAT64 address, for this reason it is very important that the DNS64 server knows the correct NAT64 prefix.
![DNS64](/images/content/ipv6-series/DNS64_flow.png)

The advantage of using DNS64 is quite clear, it doesn't require any changes to your clients. But there are unfortunately a few drawbacks:
- If used standalone without CLAT on the clients, it doesn't offer any fallback in case a service has a AAAA record, but the IPv6 implementation of the site for some reason doesn't work. To be fair, this is not a flaw in DNS64 itself, but just a consequence of purely relying on DNS64.
- It doesn't offer any way of translating IPv4 littrals. While generally not a huge problem, it is a problem in some cases, most notably Discord voice chat.
- If your endpoints are doing DNSSEC validation, it will detect that the DNS server is lying to you and reject the response.

### 464XLAT
464XLAT introduces a new component, a Customer site translater called CLAT. The CLAT is most often located on the endpoint device itself, but it doesn't have to be. If as an example you have 5G router on an IPv6 Only mobile network, you probably have a CLAT function built into your router. CLAT essentially just allows the translation of IPv4 packets into IPv6 packets using the NAT64 prefix.
![464XLAT](/images/content/ipv6-series/464xlat.png)

The pros of this is that IPv4 works no matter if you have DNSSEC, IPv4 littrals, or whatever else. The cons are that it requires a new component usually located on the endpoint device itself.
Mobile devices generally have very good CLAT implementations, apple have also included the Iphones CLAT implementation in macOS. Microsoft have commited to CLAT for all network types in Windows 11, but they commited to that over a year ago, and we haven't heard anything since. 

But how do CLAT implementations even know what NAT64 prefix to use? There are generally two ways of doing this.
- The first and preferred way is to use PREF64 router advertisements. This option needs to be implemented per endpoint network, but it enables the router to inform the client about the NAT64 prefix when announcing the IPv6 router information. 
- Another way is using DNS64. This requires the client to lookup a AAAA record for ipv4only.arpa. Per RFC7050 the response for ipv4only.arpa should be 192.0.0.170/192.0.0.171. So AAAA response would indicate NAT64 is implemted. The NAT64 prefix is found by taking the first 96 bits of the IPv6 address in the response, and using that as the NAT64 prefix. It is worth noting that the IETF is working on deprecating this method, recormending the use of PREF64 instead.

### Comparing NAT64 implementations
I will focusing mostly on NAT64 implementations that are free, and easy to implement. So yes you could ask Cisco/F5/Juniper/etc for a NAT64 implementation, solution. But not everyone has access to that.

I do however have a cisco router in my homelab, so i will include that just because i could use it.
#### Tayga
I am currently using Tayga inside OpnSense and it has worked fine for me. From what i remember this was generally the recormended solution back when i last researched NAT64. It seems like it's not the best option for performance, and that it has had some problems with lacking maintence. 

Earlier in 2025 some new life was given to Tayga, in the form of Andrew Palardy being the new maintainer (Checkout his ***[youtube channel](https://www.youtube.com/@apalrdsadventures)*** if you like this kind of content)

It is ofcourse posible to setup a VM, and just run Tayga on any Linux server, but tayga is also the NAT64 option for OpnSense, and PfSense. 

#### Jool
Jool seems to be a newer better performing option, development seems to be slow but still existing. 
Unlike Tayga, it runs as a kernel module. This could be why the performance is much better. 

I haven't done any performance testing but Nico Schottelius did a ***[presentation at RIPE85](https://ripe85.ripe.net/presentations/78-ripe85-open-source-nat64.pdf)*** and found Jool to perform more than twice as fast as Tayga, but I haven't tested it myself yet.

If you want an out of the box solution using Jool, it seems like Jool is the built in NAT64 option for VYOS. 

#### Cisco IOS XE
I happen to have a fairly modern Cisco router in my lab, so I wanted to look at if i could use that. I would probably not recormend going out to buy a physical router just to use it for NAT64.

The main pro for me is that it is something that is more likely to see in a production network. Obivoiusly when running a production network, vendor support is a very important component. It also seems very easy to configure, and i am sure it would work fine. 
A drawback for me is power consumption. I currently don't have any other reason to run that router 24/7, so locating NAT64 on it, would add a new source of power draw to my homelab.

## My setup
Based on above mentioned options, i have decided to use Jool but inside VYOS. I have also decided to use DNS64 given that the majority of my devices doesn't have a built in CLAT implementation.

### VYOS NAT64 configuration
Even though i have sevral diffrent VYOS routers in my network, i have decided to setup a new router for this purpose. I am mainly doing this for seperation of functions, and because any excuse to complicate my home networks routing is a good one.

To start out with, i am only building one router, but i might add redundancy in the future

I will be using the following configuration:
```
# Set the IPv4 external address, in my case i just use DHCP
set interfaces ethernet eth0 address dhcp

# Setup a loopback IP for mgmt
set interfaces loopback lo address 3fff::64:a/128

# Set the IPv6 address
set interfaces ethernet eth0 address 3fff:64:ff9b::b/64

# Setup routing, in my case this router will be part of my ASN AS201911. Using a private ASN, or static routing is absolutly also an option.
set policy prefix-list6 ANY6 rule 1 prefix ::/0
set policy prefix-list6 ANY6 rule 1 ge 0
set policy prefix-list6 ANY6 rule 1 action permit

set policy prefix-list6 EXPORT rule 1 action permit
set policy prefix-list6 EXPORT rule 1 prefix 3fff::64:a/128
set policy prefix-list6 EXPORT rule 2 action permit
set policy prefix-list6 EXPORT rule 2 prefix 64:ff9b::/96

set protocols bgp system-as 201911
set protocols bgp peer-group INTERNAL remote-as 201911
set protocols bgp peer-group INTERNAL address-family ipv6-unicast
set protocols bgp peer-group INTERNAL address-family ipv6-unicast prefix-list export EXPORT
set protocols bgp peer-group INTERNAL address-family ipv6-unicast prefix-list import ANY6
set protocols bgp address-family ipv6-unicast redistribute connected
set protocols bgp address-family ipv6-unicast redistribute static
set protocols bgp address-family ipv6-unicast network 64:ff9b::/96

set protocols bgp neighbor 3fff:64:ff9b::a peer-group INTERNAL

# Configure NAT64
set nat64 source rule 100 source prefix '64:ff9b::/96'
set nat64 source rule 100 translation pool 1 address 100.127.255.1
set nat64 source rule 100 translation pool 1 port '2000-65000'

# NAT64 really wants a static ip, but since i want to configure my interface as DHCP, i am creating an internal interface and NAT44'ing that interface.
set interfaces dummy dum0 description VIRTUAL_NAT64_OUTSIDE
set interfaces dummy dum0 address 100.127.255.1/24

set nat source rule 100 outbound-interface name 'eth0'
set nat source rule 100 source address '100.127.255.0/24'
set nat source rule 100 translation address 'masquerade'
```