---
layout: post
title: Validating BGP Routes with RPKI in BIRD
author: Brooks Swinnerton
---

## Introduction to BGP

For some time now, I’ve been fascinated by the way that networks interconnect with one another. What is happening when I type `google.com` into my browser? How does my ISP send my request to Google? What does that interconnection look like?

I’ve come to learn that the answer is BGP: the Border Gateway Protocol. BGP is a routing protocol in which two networks (known as “autonomous systems”) exchange information about how to get to their slice of the internet. We can visualize the boundaries of autonomous systems with a `traceroute`. As I write this article from a coffee shop, I start a trace route to Google:

```
$ traceroute 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 64 hops max, 52 byte packets
 1  nyclnyrga1r.wifi.rr.com (24.29.97.185)  15.426 ms  14.597 ms  17.935 ms
 2  agg210.nyclnyrg01r.nyc.rr.com (24.29.97.176)  22.046 ms  22.827 ms  15.998 ms
 3  bu-ether29.nwrknjmd67w-bcr00.tbone.rr.com (107.14.19.24)  21.914 ms  21.526 ms  11.968 ms
 4  66.109.5.138 (66.109.5.138)  21.063 ms
    bu-ether12.nycmny837aw-bcr00.tbone.rr.com (66.109.6.27)  21.389 ms
    66.109.5.138 (66.109.5.138)  22.444 ms
 5  0.ae1.pr0.nyc20.tbone.rr.com (66.109.6.163)  14.863 ms  60.652 ms  20.229 ms
 6  ix-ae-6-0.tcore1.n75-new-york.as6453.net (66.110.96.53)  19.951 ms
    ix-ae-10-0.tcore1.n75-new-york.as6453.net (66.110.96.13)  18.836 ms
    ix-ae-6-0.tcore1.n75-new-york.as6453.net (66.110.96.53)  14.319 ms
 7  72.14.195.232 (72.14.195.232)  18.199 ms  15.604 ms  17.883 ms
 8  * * *
 9  dns.google (8.8.8.8)  20.173 ms  22.622 ms  16.466 ms
```

Inspecting the traceroute output, we can see the first seven hops all include the `rr.com` domain. The “rr” domain is a reference to Road Runner, an ISP which was acquired by Time Warner some years ago. Great, so we know that my coffee shop’s ISP is Time Warner and my request starts by flowing through a series of their routers until eventually it gets to `ix-ae-6-0.tcore1.n75-new-york.as6453.net`. This is the first autonomous system boundary on our way to Google. After some research, we see that [AS6453](https://www.peeringdb.com/asn/6453) is Tata Communications, a “tier 1” ISP. This seems to suggest that Time Warner doesn’t have a direct route to Google, instead, they pass along my request to _their_ ISP that does know how to get to Google. Tata then takes my request and routes it through a few of their own hops until eventually we enter the next autonomous system boundary and my request lands on a Google DNS server.

At each of these autonomous system boundaries, the routers establish a BGP session with one another and exchange routes that they know how to get to.

## The dangers of BGP

BGP is a naive protocol that blindly trusts the routes being passed between autonomous systems. What happens if an AS accidentally shares a route that doesn’t belong to it? Could a bad actor maliciously originate a route and intercept that traffic? The answer is yes, and has been the cause of numerous outages across the internet like [this one](https://blog.cloudflare.com/how-verizon-and-a-bgp-optimizer-knocked-large-parts-of-the-internet-offline-today/).

One solution to this is RPKI, in which the network operators can cryptographically sign which autonomous systems are allowed to originate their routes (also known as prefixes). Regional internet registries (RIRs) then host these signed certificates, which are known as Route Origin Authorizations.

This is great, but requires that autonomous systems only accept RPKI valid routes, which requires changes to their BGP-facing routers.

## RPKI validation with BIRD

This article details how to drop invalid RPKI routes using [BIRD](https://bird.network.cz/), a software based router on Linux.

It’s worth noting that RPKI validation is not done on the router itself, but instead on a validator. The valid routes are then sent to the router using a protocol called [“RPKI to RTR"](https://tools.ietf.org/html/rfc6810). The reason the validation doesn’t take place on the router is largely because this cryptographic verification can be expensive, and most routers aren’t well equipped to do it.

There are many different options out there for validators. They range from Cloudflare’s [gortr](https://github.com/cloudflare/gortr) (written in Go), NLnet Labs’ [routinator](https://github.com/NLnetLabs/routinator) (written in Rust), and [RTRLib](http://rtrlib.realmv6.org/).

I’ve chose to run [gortr](https://github.com/cloudflare/gortr), mostly because it can run inside a simple Docker container and because Cloudflare caches the aggregated prefix data sources at its edges, which means it should be relatively quick.

I started by setting up gortr on a Docker host with the following `docker-compose.yml` file:

```
version: '3'

services:
  routinator:
    image: cloudflare/gortr
    container_name: gortr
    restart: unless-stopped
    ports:
      - 8282:8282
```

And then started it with `docker-compose up -d`. Tailing the logs we can see it starts up successfully `docker-compose logs -f`:

```
gortr    | time="2019-11-11T16:21:23Z" level=info msg="New update (114706 uniques, 114706 total prefixes). 0 bytes. Updating sha256 hash  -> a294dbee9bae169a5d007fa15b084d122ecbb202f2039e4862d75b4a783c9ca7"
gortr    | time="2019-11-11T16:21:24Z" level=info msg="Updated added, new serial 1"
```

Great, now gortr has a list of RPKI valid prefixes, we just have to get them over to BIRD. As of version 2.0.7, BIRD supports [two ways](https://bird.network.cz/?get_doc&v=20&f=bird-6.html#ss6.13) of transporting this data: unencrypted TCP and SSH. For today’s post let’s use TCP.

Despite not using SSH as the transport, we need to ensure that BIRD is compiled with support for libssh ([source](https://bird.network.cz/pipermail/bird-users/2019-November/013962.html). The libssh package varies by distribution, but if you’re on a Debian-based host you can use `libssh-dexv`. Start by installing some of BIRD’s dependencies, downloading, and compiling it:

```
apt install build-essential flex bison libncurses5-dev libreadline-dev libssh-dev
cd /tmp
wget ftp://bird.network.cz/pub/bird/bird-2.0.7.tar.gz
tar -xzf bird-2.0.7.tar.gz
cd bird-2.0.7
./configure --enable-libssh
make
make install
```

Once you have BIRD up and running, we need to configure it to point to our validator. This can be done with the following configuration:

```
roa4 table r4;
roa6 table r6;

protocol rpki {
  roa4 { table r4; };
  roa6 { table r6; };

  remote “docker.fqdn.com” port 8282;

  retry keep 90;
  refresh keep 900;
  expire keep 172800;
}

function is_v4_rpki_invalid () {
  return roa_check(r4, net, bgp_path.last_nonaggregated) = ROA_INVALID;
}

function is_v6_rpki_invalid () {
  return roa_check(r6, net, bgp_path.last_nonaggregated) = ROA_INVALID;
}
```

_(Be sure to swap out `docker.fqdn.com` for the hostname or IP address of your validator)._

This configuration does a few things:
1. Creates two new routing table for ROA routes (IPv4 and IPV6)
2. Configures the rpki protocol with the routing tables, address of the validator, and a few sensible values for how long to hold onto the RPKI validated routes if the validator goes down
3. Creates two new functions that can be used in `import`s to check if a route is invalid

Once you’ve saved your configuration, open a BIRD console with `birdc`, reload the configuration, and check to see if it can communicate with the validator:

```
bird> configure soft
Reading configuration from /etc/bird.conf
Reconfigured
bird> show protocols rpki1
Name       Proto      Table      State  Since         Info
rpki1      RPKI       ---        up     2019-11-11 16:22:42  Established
```

Now we have two new functions that we can use to check if prefixes are invalid: `is_v4_rpki_invalid` and `is_v6_rpki_invalid` . You can use these functions in a peering configuration like this:

```
protocol bgp AS6939v4 {
  description "Hurricane Electric";
  local 193.189.82.40 as 397143;
  neighbor 193.189.82.134 as 6939;

  hold time 90;
  keepalive time 30;
  graceful restart;

  ipv4 {
    next hop self;

    import keep filtered;
    import filter {
      if is_v4_rpki_invalid() then reject;
      accept;
    }
  };
}
```

Because we have `import keep filtered;`, we can check to see if the routes were filtered out:

```
bird> show route filtered count
5 of 5 routes for 5 networks in table master4
1 of 1 routes for 1 networks in table master6
0 of 0 routes for 0 networks in table r4
0 of 0 routes for 0 networks in table r6
Total: 6 of 6 routes for 6 networks in 4 tables
```

🤔 interesting, I would have expected more than 5 routes filtered from that session. After more research, it looks like BIRD does not automatically reject routes that were imported before the RPKI session came up. To resolve this, we need to `reload in all`:

Let’s look at those filtered routes again:

```
bird> show route filtered count
4612 of 4612 routes for 4612 networks in table master4
0 of 0 routes for 0 networks in table master6
0 of 0 routes for 0 networks in table r4
0 of 0 routes for 0 networks in table r6
Total: 4612 of 4612 routes for 4612 networks in 4 tables
```

That’s better. And terrifying. It looks like we’re filtering 4,612 invalid routes from this session. This `reload in all` quirk is something that BIRD has not solved yet, but is expected to be fixed in a subsequent release.

## Making sure it all worked

You can verify that your autonomous system isn’t accepting invalid RPKI routes by browsing to https://www.ripe.net/s/rpki-test. If all is well, you’ll see the following:

![](https://i.imgur.com/zFL0HZ7.png)

🎉
