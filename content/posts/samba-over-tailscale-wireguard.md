---
title: "Samba over Tailscale/Wireguard"
date: 2023-01-04T18:14:39Z
draft: false
tags: [server, samba, tailscale, wireguard]
---
***Edit:** I rewritten my old article as that was too long and too many things are irrelevant. As my instructor said, you don't need to write irrelevant things in technical articles. Those're just lard. The old version is still readable [here](../hidden/samba-over-tailscale-wireguard-old.md).*

I will show how to serve Samba share over Tailscale in a secure and (some what) hidden manner. I don't imply that SMB over Internet is not secure, but I'm more confident with Tailscale, or actually Wireguard that running under the hood.

# The usual way
As a person that have some experience with Linux server and setting up Internet facing applications, it's normal to assume I only need to force Samba to listen only the Wireguard interface. So specify `interface` to `tailnet0`, and enable `bind interface only` in the global section of `samba.conf`. However, that's the trap. Wireguard interface is Point-to-Point, which Samba `bind interface only` won't cope with.

# Solution
The replacement option is `host allow` in the share configuration. By limiting the IP ranges to CGNAT (100.64.0.0/10), which Tailscale uses, only clients on Tailscale will able to connect. Still, the share is viewable from the public. To hide it, use make it into a hidden share by appending a dollar sign ($) after the share name. For example, a share call `secret` becomes `secret$`. This is not perfect, but acceptable.

## Demo config
```
[share$]                            
    path = /home/user/share
    read only = no
    valid user = user
    hosts allow = 100.64.0.0/10
```