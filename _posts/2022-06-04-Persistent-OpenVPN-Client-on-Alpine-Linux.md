---
title: Persistent OpenVPN client On Alpine Linux
---
I needed to set up an OpenVPN client to connect an Alpine Linux VM (hosting containers) to a legacy environment. 
Not knowing the "old days" of OpenRC too well and being used to systemd, I struggled for a while to start an OpenVPN client
in a persistent manner through OpenRC. Unfortunately there is very little documentation about that on the Alpine wiki.

After googling for quite a while, I found the answers I needed on the [Gentoo Wiki](https://wiki.gentoo.org/wiki/OpenVPN).

#### Installation
First off, install OpenVPN:
```bash
apk add openvpn
```

#### Starting a single client
Starting a single client is easy enough:
1. Place your client configuration file in `/etc/openvpn` as `openvpn.conf`
2. `rc-service openvpn start`

This works, however, I am a firm believer in "doing things the right way", and being used to systemd instances, I found what I needed:

#### Starting a named clients/multiple instances
1. Place your client configuration file with it's desired name in  
   `/etc/openvpn/[your client name].conf`
3. Symlink an alias to the `init.d` service:  
   `ln -s /etc/init.d/openvpn /etc/init.d/openvpn.[your client name]`.  
   The service will find the file based on its invoked service name.
3. `rc-service openvpn.[your client name] start`

And now you can make it persistent:
```bash
rc-update add openvpn.[your client name]
```
