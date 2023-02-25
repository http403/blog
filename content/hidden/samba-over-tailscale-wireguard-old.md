---
title: "Samba over Tailscale/Wireguard [Old]"
date: 2023-01-04T18:14:39Z
draft: false
tags: [server, samba, tailscale, wireguard]
author: ["Hartman"]
searchHidden: true
cover.image: ""
cover.caption: ""
cover.alt: ""
cover.relative: true
---
I recently found out about Tailscale, a control plane for WireGuard. I decided to revisit an old project of mine and use it to set up a Samba server on an long ignored dedicated server so that I could access my videos remotely. However, the process proved to be more challenging than I had anticipated.

## Goal
The goal was to access my videos from the server via my Windows PC and Android phone, while also ensuring that the data was secure in transit, and not accessed by others. On my PC, I planned to use Windows' native SMB support. On my Android phone, I intended to use the MX Player app to access the videos.

## Problem
One of the main reasons I initially chose not to serve SMB over the internet was due to security concerns. While SMB is commonly used in LAN, it is not designed for use over the internet and may not provide adequate security when used in this manner. Additionally, the Android app I'm using only supports SMB, so I had no other choice but to use it.

## Solution
The solution I came up with was to tunnel all Samba traffic through Tailnet, the WireGuard network managed by Tailscale. This way, I could benefit from the security provided by WireGuard and the authentication provided by Tailscale. Completing the goal of secure data in transit and authenticated access.

### Alternate Solution
Another solution would have been to use a traditional VPN like OpenVPN, but this introduces a major drawback: all of my traffic would have to go through the VPN. While it is possible to use advanced configurations to avoid this issue, I did not have sufficient knowledge of networking to do so. When WireGuard first came out, I considered using it, but ultimately decided not to because I was unfamiliar with overlay networks and the work required to manage such a network.

## Implementation
I won't go into detail on the process of setting up Samba or Tailscale, as I am not an expert in Samba configuration and Tailscale have a easy to follow quick start.

### Initial Configuration
After consulting the documentation, I believed the following configuration would be sufficient:

```
[global]
    
    interfaces = lo tailnet0        # Listens on Tailscale and loopback interface
    bind interfaces only = yes      # Only listen

[share]
    path = /home/user/share
    read only = no
    valid user = user               # Defense in depth, more layer protection is always better
```

### Debugging the Issue
Unfortunately, this configuration did not work as expected. The Samba server did not bind to the Tailscale interface. Upon further investigation, I read a [post on Unix StackExchange](https://unix.stackexchange.com/a/613409) that Samba will not listen Wireguard interface unless IP and netmask are given. As a result, I modified the configuration as follows:

```
[global]
    interfaces = lo 100.64.0.0/10   # 100.64.0.0/10 is the IP range that Tailscale assigns
    bind interfaces only = yes

[share]
    path = /home/user/share
    read only = no
    valid user = user
```

Sadly, this modification did not resolve the issue and the Samba server still did not listen. Despite spending two days trying to resolve the problem, I reread the documentation and discovered the following information regarding the `bind interface only` parameter:

> ...  Note that you should not use this parameter for machines that are serving PPP or other intermittent or non-broadcast network interfaces as it will not cope with non-permanent interfaces. ...

## Final Configuration

This led me to realize that the `bind interfaces only` parameter was causing the issue. However, simply disabling it would cause the server to listen on every broadcast-capable interface in addition to the Tailscale interface.

To minimize the effect, I decided to use the `hosts allow` parameter to restrict access to only clients on the same Tailscale network and enable hidden share. However, this is not a perfect solution. Upon port scan, one can still see the server have Samba installed. People can still visit the root directory, although they can't see any share. Still, I deemed it should be sufficient for meeting my goal.

The final configuration I used is shown below:

```
[global]
    interfaces = lo 100.64.0.0/10

[share$]                            # The trailing dollar sign makes it hidden
    path = /home/user/share
    read only = no
    valid user = user
    hosts allow = 100.64.0.0/10     # Clients on the same Tailnet will have the same IP range
```

## Conclusion
In conclusion, it is not advisable to serve SMB over the internet due to security concerns. However, by using Tailscale, the hosts allow parameter, and hidden share, it is possible to securely serve SMB to clients within a specific network. While Tailscale's access control can be used to implement more stringent access control lists (ACLs), it was not necessary in my case as I was the only user.