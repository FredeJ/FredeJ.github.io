Title: Multicast service discovery is a mess
Date: 08-01-2022
Category: Networking
Status: Draft

The state of network discovery is a mess.

Let's start from the top.

Multicast Domain Name Service was proposed as an RFC in 2000 by Bill Woodcock and Bill Manning[1]. This RFC was picked up by Microsoft and Apple and turned into LLMNR and "Bonjour / Rendevouz" respectively. Apples implementation later turned into what's today known as mDNS [2] and DNS-SD[3].

The purpose of the service is achieving what is known as zero configuration networking (zeroconf). This was all the rage back in the day, where a ton of manual work had to be done to properly set up a network. Zeroconf includes DHCP for IPs and DNS for hostnames and service discovery - basically everything we consider ubiqutious for a functioning network today.

While DHCP and DNS are both massively successful at their primary usecase, the service discovery part has always felt like the ugly duckling. 

Service Discovery on the local network can be broken into two seperate pieces: mDNS and DNS-SD. mDNS is the ability for DNS Records to be distributed over the local network, while DNS SRV Records (Service Records) allows services to be defined and distributed over DNS. Together, they allow services to be discovered on the local network.

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

For a WLAN, every time a multicast or broadcast is sent, the entire wireless network slows down. Much like TCP, the protocol used in WLAN is acknowleding messages to improve reliability. With multicast it cannot do that. What happens instead is that it slows down to the very slowest speed it can go and multicasts the message as that has the highest likelyhood of the message being received by everyone. This takes forever by Wi-Fi standards, and this is time where other messages cannot be sent, slowing down the throughput of the entire network. And again: This issue scales with numbers of users. 

Beyond that, if the TXT Record contains information necessary to the solution you're designing you're forced to do resolutions (and for some implementations, full on connections) to get some basic information that could've just been provided to you in the first place! Not to mention the extra complexity of having to handle services at different stages of resolution in your software.

# Implementations

The implementation side of mDNS is even more of a mess. Linux has 1, MacOS and iOS have 2 and Windows has like 3, all of which are seriously flawed in different ways. Android has one which is significantly different from the iOS one, which makes it hard to built a cross platform Flutter implementation. Beyond that, there's a number of implementations in different languages, which is really a reflection of the poor OS support for something that really should be an OS feature. You don't see that many language specific implementations for things like the TCP or IP protocol.

I intend to do a bit of a deep dive into the different options available here.

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