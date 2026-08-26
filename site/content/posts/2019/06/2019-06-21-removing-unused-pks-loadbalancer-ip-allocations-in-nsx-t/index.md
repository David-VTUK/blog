---
title: "Removing Unused PKS LoadBalancer IP allocations in NSX-T"
date: 2019-06-21
slug: "removing-unused-pks-loadbalancer-ip-allocations-in-nsx-t"
categories: 
  - "nsxsddc"
---

I'm constantly building, removing, destroying, breaking and constantly tinkering in my lab, which unfortunately can leave behind a bit of technical mess. Most recently I've been doing some messing around in PKS and consequently had a load of load balancer VIPs no longer in use due to clusters being destroyed and / or recreated.

Unfortunately, there's no way to remove these in the UI (for now, at least). But you can remove them using the API.:

\[code language="bash"\]

curl -k -u <NSX-USERNAME>:<NSX-PASSWORD> -X POST 'https://<NSX-IP-OR-FQDN>/api/v1/pools/ip-pools/<POOL-ID>?action=RELEASE' -H "X-Allow-Overwrite: true" -d {"allocation\_id":"10.10.10.2"}' -H "Content-Type: application/json"

\[/code\]

This is great for the odd IP, but for me, I had 10's of IP addresses I needed releasing. A quick and easy Bash script to the rescue!

\[code language="bash"\]

for i in {2..254} do curl -k -u <NSX-USERNAME>:<NSX-PASSWORD> -X POST 'https://<NSX-IP-OR-FQDN>/api/v1/pools/ip-pools/<POOL-ID>?action=RELEASE' -H "X-Allow-Overwrite: true" -d '{"allocation\_id":"10.10.10.'"$i"'"}' -H "Content-Type: application/json" done \[/code\]

Modify the for loop for your environment. I typically have .1 as my default gateway.

Note : After releasing an IP, it seems to take a while before it's reflected in the UI, therefore give it a few minutes.
