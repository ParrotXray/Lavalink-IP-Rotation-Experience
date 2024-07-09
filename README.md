# Lavalink-IP-Rotation-Experience

## Preface

If your robot is hosted across many servers and by many people, you may need an IPv6 tunnel broker to prevent YouTube from rate limiting your IP. Having a large pool of available IPv6 addresses can be beneficial. Assigning a large prefix of addresses to computers can be challenging, as it depends on the configuration of IP addresses, which is often beyond your control if you're renting servers.

We can obtain an IPv6 prefix for routing using Hurricane Electric's free Tunnelbroker service. This can also work if your server initially lacks an IPv6 address. If you intend to use Tunnelbroker, remember it's a free service and may experience interruptions.

Important Note: Exercise caution when making changes to real-time production systems. Errors here could disrupt your network and leave your server inaccessible. Consider testing on a disposable VPS.

## setup.1
- **Obtain Tunnel Broker**  
> Register an Account at https://tunnelbroker.net/  
> Create a new (regular) tunnel  
> Enter your IPv4 IP address and select a nearby region. Your IPv4 address must be pingable.  
> Click on "Assign /48" to request a new /48. We can use a /64, but /64s are more likely to be blocked. Assigning a /48 may take several days to process.  

## setup.2
- **Configuring TunnelBroker**
> bind to local address
```sh=
sysctl -w net.ipv6.ip_nonlocal_bind=1
echo 'net.ipv6.ip_nonlocal_bind = 1' >> /etc/sysctl.conf

sysctl -w net.ipv6.conf.all.autoconf=0
echo 'net.ipv6.conf.all.autoconf=0' >> /etc/sysctl.conf

sysctl -w sysctl -w net.ipv6.conf.all.accept_ra=0
echo 'net.ipv6.conf.all.accept_ra=0' >> /etc/sysctl.conf
```
- **Configure Network**
> if use interfaces 
```sh=
vim /etc/network/interfaces
```
```sh=
auto he-ipv6
iface he-ipv6 inet6 v4tunnel
        address 0:0:0::2 # Routed ex. 2001:354:1df4::2
        netmask 48
        endpoint 0.0.0.0 # Tunnel Server IPv4 Address
        local 0.0.0.0 # Tunnel Client IPv4 Address
        ttl 255
        gateway 0:0:0::1 # Routed ex. 2001:354:1df4::1
 post-up /sbin/ip -6 route replace 0:0:0::/48 dev lo # ex. 2001:354:1df4::/48
```
> else use Netplan
```sh=
vim /etc/netplan/99-he-tunnel.yaml
```
```sh=
network:
  version: 2
  tunnels:
    he-ipv6:
      mode: sit
      remote: 0.0.0.0 # Tunnel Server IPv4 Address
      local: 0.0.0.0 # Tunnel Client IPv4 Address
      addresses:
        - '0:0:0::/48' # ex. 2001:354:1df4::/48
      gateway6: '0:0:0::1' # ex. 2001:354:1df4::1
```

## setup.3
- **Test your configuration**
```sh=
# Test that IPv6 works in the first place
ping6 google.com

# Test your tunnel with
ping6 -I he-ipv6 google.com

# If you have the IPv6 block 2001:354:1df4::/48
# You should be able to use any of the IPs within that block
ping6 -I 2001:354:1df4:: google.com
ping6 -I 2001:354:1df4::1 google.com
ping6 -I 2001:354:1df4:dead::beef google.com

# Make sure your /48 block appears when running this command
ip -6 route
## it should look something like this
#::1 dev lo proto kernel metric 256 pref medium
#2001:354:1df4::/48 dev he-ipv6 proto kernel metric 256 pref medium
#fe80::/64 dev eth0 proto kernel metric 256 pref medium
#default via 2001:470:cc7b::1 dev he-ipv6 proto static metric 1024 pref medium
```

## setup.4
- **setting ipv6 route**
```sh=
ip add add local <ipBlock ex. 2001:354:1df4::/48> dev lo
ip -6 route replace local <ipBlock ex. 2001:354:1df4::/48> dev lo
```

## setup.5
- **Add the ratelimit block to your config**
```yml= 
    ratelimit:
      ipBlocks: ["2001:470:fc39::/48"] # list of ip blocks
      #excludedIps: ["...", "..."] # ips which should be explicit excluded from usage by lavalink
      strategy: "RotatingNanoSwitch" # RotateOnBan | LoadBalance | NanoSwitch | RotatingNanoSwitch
      searchTriggersFail: true # Whether a search 429 should trigger marking the ip as failing
      #retryLimit: -1 # -1 = use default lavaplayer value | 0 = infinity | >0 = retry will happen this numbers times
```
## END
- **Reference materials**

> 1. https://blog.arbjerg.dev/2020/3/tunnelbroker-with-lavalink#:~:text=Replace%20the%20/64%20prefix%20with%20the%20/48%20one%20you%20were%20allocated
> 2. https://blog.darrennathanael.com/series/ipv6/
> 3. https://github.com/AceAsin/Lavalink
