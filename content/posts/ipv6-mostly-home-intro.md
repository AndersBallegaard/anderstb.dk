---
date: '2025-07-05T22:56:45+02:00'
draft: false
title: 'A Glimpse into the Future: An introduction to IPv6 in your homelab'
author: "Anders Ballegaard"
tags:
  - IPv6
  - NAT64
  - Homelab
  - Home network
image: /images/content/intro-homelab-v6-hero.png
description: "The start of a series of posts related to building an ipv6 mostly home network and lab"
toc: true
---
## Why Should IPv6 be a part of a homelab?
I have been a long-time advocate for IPv6. It has been a crucial part of my homelab for years, and through my work at a major Danish ISP, I've have among other things contributed to enabling and improving IPv6 for many Danish broadband customers.

As I'm currently updating and fine-tuning some aspects of my homelab, I thought it would be a good idea to document the process here. This will serve as not only personal documentation but also an introduction for anyone interested in setting up their own IPv6 homelab.

But why should you care about IPv6? Let's take a look at its current usage.

Firstly, almost half of all internet traffic is now IPv6. The numbers may vary slightly, but according to reports from Google and Meta, the trend is clear:

![google ipv6 stats](/images/content/ipv6-series/google-stats.png)
[Source](https://www.google.com/intl/en/ipv6/statistics.html)
![meta ipv6 stats](/images/content/ipv6-series/meta-stats.png)
[Source](https://www.facebook.com/ipv6/?tab=ipv6_total_adoption)

Besides the fact that a large portion of the internet is already using IPv6, there are also pushes from both companies and goverments to move to ipv6. Some of those major pushes include:

- Apple requires all app store apps to support working in IPv6-only networks. They have required this since 2016.
- Several mobile operators have deployed IPv6-only mobile networks, with 464XLAT being the only way of accessing IPv4 sites. In the West, the most notable example is probably T-Mobile in the US. However, to my knowledge, this approach is also common in developing countries due to IPv4 scarcity.
- The US Office of Management and Budget has implemented an IPv6 mandate. In 2023, the US federal government presented a quite ambitious plan for moving to IPv6.
- China has mandated that Chinese router manufacturers must enable IPv6 by default in all new routers they sell.
- Most major cloud providers have started not including public IPv4 addresses for free, thus adding an extra cost for still running IPv4 directly on servers. While this does not force organizations to change, it is a nudge that can be used as a motivator.

Ofcourse companies and goverments isn't just pusing for ipv6 for no reason at all. It takes a lot of effort to change, so there needs to be some good reasons behind the change. So here are some of the reasons:
- We are running out of IPv4 address space. Part of this problem is related to the fact that early IPv4 allocation was made in a very shortsighted way; unfortunately, there isn't really a way to change this. (And no Class E or redefining 127.0.0.0/8 won't work.) Unlike many IPv6 supporters, I don't like to say we have run out, but instead say we are running out. While it is true that getting new IPv4 space directly from your RIR is impossible (or close to it), there is still a healthy resale market. So you can get IPv4 space, but supply and demand makes a pure IPv4-only internet an impossibility now due to the amount of things we want connected.
- Simpler routing and network operations are two benefits of IPv6. This might sound counterintuitive for anyone who has grown up with IPv4 networks, and I do admit it takes some time getting used to. But once you see the beauty in always using /64 netmasks without having to worry about exhaustion, or when you start to appreciate the simplicity of not dealing with NAT when troubleshooting, or realize the simplicity of the (base) IPv6 header compared to IPv4's, you'll understand what I mean. Like all things, there is a learning curve, and the more time you have spent with IPv4, the harder it probably is; but the more you use IPv6, the easier it becomes, and the more you will love it.
- Decreased latency is another benefit of IPv6. Removing NAT on the internet does decrease latency, especially if your ISP forces you through CGNAT routers placed outside the optimal network path. In some cases, we also see a decreased latency due to cutting out legacy infrastructure that only supports IPv4.
- Energy efficiency is also a benefit of IPv6. Kinda the same as latency, removing NAT removes compute cycles to do NAT and decreases power consumption.
- The use of extension headers enables several key protocol improvements, including:
    - Routing header: This allows the source device to specify the path it wants to take through the network. A very cool application of this is SRv6 routing.
    - IPsec header: This allows for encryption and authentication of packets built directly into the IP protocol, instead of as an additional layer like it is in IPv4.

So now that you have a glimpse into why you should care about ipv6, I want to encourage you all to start experimenting with ipv6. Whether you're building networks or developing apps, understanding how to work with ipv6 is essential for the future of networking and computing. With ipv6, we can expect simpler routing, decreased latency, improved energy efficiency, and more. By starting to experiment with ipv6 today, you'll be better equipped to handle the challenges and opportunities that come with it.


## IPv6 Mostly vs IPv6 Only
It's probably important to start out defining what I am trying to achieve and what some common terms mean.

### IPv6 Only
This is straightforward; it means that you have access only to an IPv6 network. Unless you understand your devices and applications very well, this might not be a good idea right now.

IPv6 only is the ultimate goal, but we aren't there yet. So instead of IPv6 only, most networks are targeting IPv6 mostly as a stepping stone.

Ipv4 connectivity might still be provided for backwards compatibility through NAT64.

### IPv6 Mostly
This is a defined term; see [IETF draft-ietf-v6ops-6mops-01](https://datatracker.ietf.org/doc/draft-ietf-v6ops-6mops/) for the full version, but here's the short version:
- The network must work for IPv6 only clients, dual-stack clients, and IPv4 only clients. The goal is to provide a space for migrating clients towards IPv6 only.
- The network must provide a NAT64 solution to the clients; there is no requirement for providing a DNS64 solution.
- The network's DHCPv4 server(s) must include DHCP option 108 in responses to clients, indicating to hosts that support IPv6 only that the network also supports IPv6 only. Option 108 essentially lets a device skip getting an IPv4 address.

### My target
My target for now is IPv6 Mostly, and here's why:
- I own devices that don't support IPv6 or don't support IPv6 only operations.
- This is the most common deployment method.
- It doesn't limit me from running some devices as IPv6 only for testing purposes.

I have chosen IPv6 mostly because it provides a good balance between being forward-thinking and still supporting backwards compatibility with IPv4 networks. While IPv6 only might be the ultimate goal, IPv6 mostly is a more achievable target that can help pave the way for widespread adoption of IPv6 in the future.

## So how do i access ipv4 only sites?
The short answer is NAT64 + either DNS64 or CLAT. I will dedicate a blog post in the future to NAT64, but here's the short version of what it does. Due to IPv6 having more bits than IPv4, we can cram an ipv4 address into an ipv6 address. We traditionally use 64:ff9b::/96 for this, but there are other options. So let's say you wanted to access 1.1.1.1 via NAT64, instead of sending your packet to 1.1.1.1, you would send it to 64:ff9b::101:101 given that is what the address would be if you took the first 96 bits from 64:ff9b:: and added the 32 bits of 1.1.1.1.

But we are (mostly) not accessing services directly by ipv4 address, so we need to map DNS to this mess, somehow. There are two ways this is done
- DNS64 - This is essentially the DNS server lying to the client, by creating a fake AAAA record though the NAT64 device if no AAAAs exist for that domain. But given the DNS server is lying to the client, DNSSEC doesn't like DNS64. The advantage is that it works on any device that supports IPv6. But it only works for DNS, so any IPv4 literals won't be saved by this. Another indirect consequence of this approach is that sites with AAAA records, but broken ipv6 doesn't have any way to fall back to the ipv4 connectivity.
- CLAT aka 464XLAT - This works by having code on the device doing the translation, it's typically implemented as a new ip on an existing interface, or new interface entirely. This is very common in mobile devices, and it is (very slowly) getting implemented on desktop devices. The advantage is that this works for both DNS and IPv4 literals, and it doesn't involve changing DNS responses.


## A short introduction to my home network, and what i want to do.
To say that my home network is unusual would be an understatement. Like a lot of people working in IT, I have a sizable homelab, but unlike most others, I have decided to somewhat separate my lab from the rest of the network. Oh and then there is the small detail that I am running my own publicly routed ASN (AS201911), and though that has a /44 IPv6 allocation.

The following is a diagram from earlier this year, of how I wanted the network to look logically. Some of this isn't implemented, but it gives a picture of the direction I have been going
![Network diagram](/images/content/ipv6-series/logical-network-diagram-2025.svg)

I will fully acknowledge that best practice is an unknown concept in this rat's nest of a network. But my goals have never been to create something that made sense; it has been to create something that gave me the flexibility I wanted to do whatever I want with limited impact on other parts of the network. Besides that, I just like BGP, and wanted more BGP in my home network.

I don't have a public IPv4 address for my home network, so everything I expose is exposed through IPv6 only, mostly with Cloudflare proxy in front of the service, both to protect the service, and to enable dual-stack access through Cloudflare's proxy service.

All routers you see in the diagram are either OpnSense firewalls or VYOS routers.

So what do I want to do with the network?
- Create a centralized NAT64 service. Right now, the DKNIM-LFW cluster, and DKNIM-HFW clusters are both running NAT64; I would like to centralize this.
- Enable option 108 on all networks with DHCP. A lot should already have it, but it's not enabled everywhere.
- Explore running CLAT on Linux servers.
- Explore options for a permanent IPv6 only or dual-stacked container platform.


## Expected challenges
If you are starting an IPv6 mostly journey, here are some things to be aware of.
- Firstly, there are a few popular services using ipv4 literals, most notably Discord. So if you enable option 108 on a device without CLAT, don't be surprised when parts of Discord stops working.
- You might also find that your ISP doesn't support ipv6, you can of course solve this in the crazy person way and start your own ISP like network, or you could be more sensible, and use something like HE tunnels.
- IOT devices generally don't have great ipv6 support.
- If you are used to doing music streaming from your phone to maybe a Sonos speaker, that might break with option 108, given that Sonos doesn't support ipv6, and your phone most likely won't have an ipv4 address.
- Containers and ipv6 - Generally not a good time, although it can be in some cases.
- Some applications you host might listen to 0.0.0.0 instead of [::] (this supports both v4 and v6), if it's an open source project, and you have the ability, please fix it in the project, and try to get it merged.


## What is next?
My plan is to start looking into diffrent NAT64 options given i have been out of that game for a bit. So look forward to a post comparing different options, and detailing what i will end up doing.
