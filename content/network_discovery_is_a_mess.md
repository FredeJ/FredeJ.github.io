Title: Multicast service discovery is a mess
Date: 08-01-2022
Category: Networking
Status: Draft

The state of network discovery is a mess.

Let's start from the top.

Multicast Domain Name Service was proposed as an RFC in 2000 by Bill Woodcock and Bill Manning[1]. This RFC was picked up by Microsoft and Apple and turned into LLMNR and "Bonjour / Rendevouz" respectively. Apples implementation later turned into what's today known as mDNS [2] and DNS-SD[3].

The purpose of the service is achieving what is known as zero configuration networking (zeroconf). This was all the rage back in the day, where a ton of manual work had to be done to properly set up a network. Zeroconf includes DHCP for IPs and DNS for hostnames and service discovery - basically everything we consider ubiqutious for a functioning network today.

While DHCP and DNS are both massively successful at their primary usecase, the service discovery part has always felt like the ugly duckling. 

Service Discovery on the local network can be broken into two seperate pieces: mDNS and DNS-SD. mDNS is the ability for DNS Records to be distributed over the local network, while DNS-SD defines DNS SRV Records (Service Records) to be distributed over DNS. Together, they allow services to be discovered on the local network.

So what's the issue? Well, as I'll demonstrate in this article, the strengths of the solution are unimportant in its most widely used form, while the drawbacks are actively hurting.

# The mDNS Protocol

So, through DNS A Records, we have a way of pairing hostnames to IPs. By introducing mDNS and distributing A records, we have now solved the issue of naming PCs on the local network without relying on a centralized location. Great! This, alongside DHCP takes care of 2/3s of the goals of zeroconf!

So lets say we have a server with the hostname `server`. When trying to access `server`, the domain name must be resolved, as is the case with any domain name. For the regular unicast DNS this involves going to your favourite DNS server and asking for an A Record for the appropriate domain. So far so good.

For mDNS the situation is a little bit different. As with all DNS Solutions a cache is maintained on your PC. However, if the entry is not in your cache, you multicast over the network "Is anyone here named `server`?". Now, anyone who has knowledge (in their cache) around who is named `server` can reply. An implementation optimization ensures that (most likely) only one actually replies. This is done through multicast, so everyone can update their cache with the latest information.

Now lets take a look at how things look for service discovery.

A service registers (multicasts) an SRV record looking something like this:

```
service protocol domain ttl IN type priority weight port target
_service._tcp.local.    120 IN SRV  0        10     5000 server.local
```

This is cached by everyone who's listening. This tells everyone that a service of type `_service._tcp.local` can be found at `server.local:5000`. Additionally, a PTR Record is created with a given instance name like `MyHomeServer`, pointing to `server.local`, which allows naming of specific services.

Another node comes online and wants to connect to a service of type `_service._tcp.local`. It multicasts a request to find all services of this type. The same optimization as above happens, resulting in a single response being multicast back, containing the cached services of the type on the network. If a node on the network determines the response to be inaccurate, the node multicasts the contents of its cache. The result is that every node on the network eventually agrees on what services are available on the network.

As you may notice, the service record contains the hostname rather than the IP. This is good, as the IP might change but the hostname can stay the same. Additionally, this means that there's no duplicate information stored, as the conversion from hostname to IP is handled by an A record. More on this later.

In addition to SRV Records, we're also able to store additional information on a service using TXT Records. TXT Records can contain anything, but are usually used as key-value stores for extra information. This information can be useful in informing others about the current state of the service (if it's a stateful service) or informing other things such as paths on webservers or similar.

# The issues

Okay, so I explained how SRV Records store hostnames rather than IPs, since the hostname to IP conversion is handled by an A record. I strongly believe this is a bad thing. Let's explore why:

So first of, honestly, in a good old regular LAN, how often does your IP really change? I fully understand that it theoreticaly could when the DHCP lease is over, but come on.. a DHCP lease is usually like 24 hours, where the regular TTL for a service registration is 120 seconds. And usually what happens with a DHCP lease for a machine is that it's just renewed, with the same IP. In other words, for the common case I believe the extra flexibility is not necessary.

Where the flexibility of having changing IPs might be necessary is in an enterprise setting, but if you're ever relying on mDNS for handling DNS in your enterprise setting, you're doing something wrong. Go do good old regular DNS instead.

And we pay for that flexibility. That flexibility costs us two multicasts every time a connection is made to any service, from any PC on the network - one request and one response. But can't we just cache the A Record and avoid it? Yes! But that's equivalent to the SRV record just providing the IP in the first place! And if we cache it, we lose that very same flexibility that we've just paid for!

Additionally, TXT Records are provided at resolution time, rather than discovery. So that's another two multicasts to get the TXT Record! Luckily, I believe that the TXT Records and A records can be squished together into a single response, but that still means that if I just want to know the state of a stateful service (such as a device) I need to resolve it. In this scenario the number of multicasts scale with the product of the number of services and the number of service-consumers! And again, this information cannot / should not be cached because it is designed around this information changing at any moment, so the response you just got before requesting your own may no longer be valid.

This might all seem trivial, but multicasts are expensive.

For a WLAN, every time a multicast or broadcast is sent, the entire wireless network slows down. Much like TCP, the protocol used in WLAN is acknowleding messages to improve reliability. With multicast it cannot do that. What happens instead is that it slows down to the very slowest speed it can go and multicasts the message as that has the highest likelyhood of the message being received by everyone. This takes forever by Wi-Fi standards, and this is time where other messages cannot be sent, slowing down the throughput of the entire network. And again: This issue scales with numbers of users and services.

Beyond that, if the TXT Record contains information necessary to the solution you're designing you're forced to do resolutions (and for some implementations, full on connections) to get some basic information that could've just been provided to you in the first place! Not to mention the extra complexity of having to handle services at different stages of resolution in your software.

# Implementations

The implementation side of mDNS is even more of a mess. Linux has 1, MacOS and iOS have 2 and Windows has like 3, all of which are seriously flawed in different ways. iOS has one which is significantly different from the Android one, which makes it hard to built a cross platform implementation. Beyond that, there's a number of implementations in different languages, which is really a reflection of the poor OS support for something that really should be an OS feature. You don't see that many language specific implementations for things like the TCP or IP protocol.

## Linux

Linux really has one option for mDNS: Avahi.

### Avahi

Avahi [4] is pretty good though seems slightly complicated, partly due to the requirements of the protocol. It is (in my opinion) a little heavy on the knowledge required to interface with it using its C interface: There are a lot of allocations of different things to keep track of, but once you have done that (or wrapped it in a way that does it for you) it's decently easy to work with. To register a service you must: Create an eventloop, create an entrygroup and a callback handler, create a service with callback handler, commit service to event loop and keep track of when everything should be allocated and freed. It feels a little heavy-handed for what is in essence "Register Service". And it means that your relatively simple app now includes atleast an event loop and likely also thread management.

On the plus side, Avahi fits neatly in the linux world, is available basically everywhere and is successful as the standard implementation for Linux. Where other OSes have multiple standards Avahi has succeeded in being the defacto standard and I applaud them for it.

## MacOS / iOS

As mentioned earlier, Apple has been heavily involved in the design and standardization of mDNS and it's not surprising that they have very good support for it. A lot of the things that make up the Apple ecosystem are powered by mDNS and it seems that mdNS support is a first class citizen in the Apple world.

It is therefore also not surprising that Apple has iterated further on what it means to have and broadcast the availability of a service on the network.

### DNS-SD / Bonjour / mdnsresponder

The most widespread implementation of the standard is likely Apples DNS-SD. It provides a service (mdnsresponder), a CLI for interfacing with the service (DNS-SD) and a library. The implementation also exists on Windows (more on that later), likely to further the goal of having DNS-SD be the standard. This implementation is however deprecated now, in favor of..

### NWBrowser

NWBrowser is Apples latest API for interfacing with service discovery. It takes the approach that a socket (an NWListener) is fundamentally required for a service to be present. So, in this scenario, a service is a property of the socket that can be exposed / enabled. Similarly, on the receiver side a discovered service directly represents a socket that can be connected to. This means that it's not possible, through this API, to get port information, for example. Because you're directly connecting to the service, you don't need to know the specific port it's connecting to - you just feed the discovered service to your connect function and it will do the resolving and connecting for you.

While I somewhat like this opinionated approach it also definitely has its drawbacks. It's impossible to retrofit into an existing application - the application must be completely rewritten to use this paradigm. It also means that TXT information is available after having *connected* to the service, as opposed to after resolving, as connecting and resolving are now part of the same step.


## Windows

Okay, so this is where the madness starts.

### Device Enumeration

The initial way of supporting mDNS in Windows was part of the Device Enumeration API. Basically, it considers a service on the network to be similar to a device attached to the PC. This concept works out pretty well for some cases, like cameras. In general, the concept is sounds, but the implementation is sorely lacking.

The way to do Device Enumeration is to activate device enumeration through an arcane filter like this `$"System.Devices.AepService.ProtocolId:={4526e8c1-8aac-4153-9b16-55e86ada0e54} AND System.Devices.Dnssd.ServiceName:=\"{proto}\" AND System.Devices.Dnssd.Domain:=\"local\""` (which is, by the way, *NOT* documented). You then start a `Watcher` to look for devices that match this querystring. However, the implementation does not respect TTL, meaning even if you deregister your service using `ttl=0` the `Remove` callback will not be fired. The most reliable way I've found of actually removing devices when they disappear, is to cache all discovered services and restart device enumeration every time the `DeviceEnumerationFinished` event fires. Then you can check which services were not found again and do your own "Remove" callbacks. This means extra multicast traffic on the network and a ton of extra software complexity.

### WinDNS

Eventually, mDNS support got rolled into the DNS System on Windows, which seems to be a much saner place to include it. It uses the `DnsServiceBrowse` function and returns a `DNS_RECORDW` [5] which can be a little hard to parse as it's basically all DNS Records rolled into a single struct, with a type field denoting which type to parse it as and a union holding the data. That's fair, though a little convoluted. However the big issue here is that when you get a SRV record *it does not reliably contain a port*. In other words, the SRV record is useless and it can be literally impossible to use this implementation.

### mdnsresponder

Finally, I think Apples mdnsresponder deserves a place here, as it's what a lot of people end up using on Windows. It works, most of the time. However, to distribute it you need to sign their Windows Bundling Agreement [6]. This includes language that Apple can immediately terminte that agreement, that Apple will be provided with usage reports and that Apple be provided with samples of the products wherein the Bonjour is included. Additionally, material related to the product must be marked with the Bonjour logo.

These additional legal requirements can be a pretty high bar to reach for some products and is - in my opinion - too much to ask for, beyond the obvious liability with giving someone the ability to remove such a foundational feature from your product. Again, would you ever sign a contract that enabled someone to disallow the usage of TCP in your application? No, obviously not.

I think one way to get around this agreement is to compile it yourself. You're allowed to do so - it's only if you distribute Apples binaries that you are beholden to their agreement. But trying to figure your way around Apples open source site is a bit of a mess. Seriously, try to figure out which one here [7] is the newest? (Hint! It's not the one with the largest number!)

# Conclusion

mDNS is currently the best widely available tool for network discovery. However the desire to fit into the existing models make it less performant and more resource demanding than it needs to be.

A number of other 



- Why is it a mess?
  - https://datatracker.ietf.org/doc/html/rfc6762
  - Attempts to provide full DNS support in a multicast scenario
  - Who is it for? Regular users does not require full DNS support. Enterprise users would never use mDNS.
  - What's really necessary? Hostnames resolution, service name, type, port, details.
- How does it work?
  - Multicast everywhere
  - Poor WLAN Performance
  - 3 multicasts to connect to service
- What different implementations are there?
  - Linux
    - Avahi
  - MacOS / iOS
    - Bonjour
    - NWBrowser
  - Windows
    - Enumeration
      - Needlessly convoluted
      - Doesn't respect TTL
      - 
    - WinDNS
    - Bonjour
  - Libraries
    - Go
    - Python
- Other protocols
  - SSDP
  - uPNP
- What's next?
https://tools.ietf.org/id/draft-ietf-dnssd-srp-00.html

[1]: https://datatracker.ietf.org/doc/html/draft-manning-dnsext-mdns-00.txt
[2]: https://datatracker.ietf.org/doc/html/rfc6762
[3]: https://datatracker.ietf.org/doc/html/rfc6763
[4]: https://avahi.org/
[5]: https://docs.microsoft.com/en-us/windows/win32/api/windns/ns-windns-dns_recordw
[6]: https://developer.apple.com/licensing-trademarks/bonjour/
[7]: https://opensource.apple.com/tarballs/mDNSResponder/