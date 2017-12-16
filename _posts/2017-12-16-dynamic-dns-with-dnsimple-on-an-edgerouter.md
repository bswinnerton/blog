---
layout: post
title: Dynamic DNS with DNSimple on an EdgeRouter
author: Brooks Swinnerton
---

A few weeks ago I caught the "homelab" bug after stumbling upon [/r/homelab](https://www.reddit.com/r/homelab/). One of my first purchases was Ubiquiti's [EdgeRouter X](https://www.ubnt.com/edgemax/edgerouter-x/). It's an incredible little device for around $50 that can take your home network to the next level.

Since purchasing the router, here's what I've been up to:

1. Purchased the domain name `brooks.network`
2. [Set up](https://help.ubnt.com/hc/en-us/articles/204950294-EdgeRouter-IPsec-L2TP-Server) a VPN back to my home network for when I'm abroad
3. [Added PXE boot support](https://gist.github.com/skottler/80a18086991367566aa54b24a11bb8d2) to my home network with netboot.xyz
4. Purchased two NUCs to host applications from, with the intentions of getting a small Kubernetes cluster running

I use Optimum as my internet service provider in New York City. Like most consumer internet plans, you're given a dynamic IP address that is subject to change. This makes DNS records tricky to keep up to date because the value you assign to an [A record](https://support.dnsimple.com/articles/a-record/) today may be different tomorrow.

The EdgeRouter has [built in](https://help.ubnt.com/hc/en-us/articles/204952234-EdgeRouter-Dynamic-DNS-commands) support for updating DNS records based on your dynamic IP address, but only with dnspark, dyndns, namecheap, zoneedit, dslreports, easydns, sitelutions, and afraid. Unfortunately, DNSimple isn't supported and that's where I host my DNS.

My solution was to use the EdgeRouter's built in scheduling system to run a script every minute to update the A record of `brooks.network`.

DNSimple has a script that they recommend using [here](https://developer.dnsimple.com/ddns/).

Start by SSH'ing into your EdgeRouter, fetching the script, and making it executable:

```
curl -o /config/scripts/ddns.sh https://developer.dnsimple.com/ddns/ddns.sh
chmod +x /config/scripts/ddns.sh
```

You'll want to start by modifying the contents of that script in your favorite editor and filling out the appropriate values¹ at the top of the file:

```bash
TOKEN="your-oauth-token"  # The API v2 OAuth token
ACCOUNT_ID="12345"        # Replace with your account ID
ZONE_ID="yourdomain.com"  # The zone ID is the name of the zone (or domain)
RECORD_ID="1234567"       # Replace with the Record ID
```

¹The appropriate values can be a little tricky to determine:

- `TOKEN`: You can create a token using [this](https://support.dnsimple.com/articles/api-access-token/) guide.
- `ACCOUNT_ID`: Your account ID can be determined by modifying any DNS record in DNSimple and looking in the URL. It's the value that comes after `/a/`.
- `ZONE_ID`: The zone ID is the domain name (or "zone") that you're updating
- `RECORD_ID`: The record ID is the actual DNS record ID that you would like to update. Similar to the account ID, this can be determined by looking at the URL when modifying the record in question.

Once you've entered in the appropriate values, it's a good idea to double check that the script is working. I'd recommend changing the IP address of the DNS record in question to something you know is incorrect like `8.8.8.8`, and then running `/config/scripts/ddns.sh` and verifying the correct value comes in.

Now you can configure the EdgeRouter to execute this script every minute:

```
configure
set system task-scheduler task ddns executable path /config/scripts/ddns.sh
set system task-scheduler task ddns interval 1m
```

And to double check your work before enabling it, you can check to see what the new configuration values will be:

```
show system task-scheduler
```

You should see something like:

```diff
+task ddns {
+    executable {
+        path /config/scripts/ddns.sh
+    }
+    interval 1m
+}
```

Then enable and save the configuration:

```
commit
save
```

Enjoy!
