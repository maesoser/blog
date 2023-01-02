+++
title = "Creating a site 2 site VPN with WireGuard"
date = "2023-01-01"
description = "How to create a site to site tunnel between two VPN gateways"

[taxonomies]
tags = ["networks", "homelab"]

[extra]
author = { name = "Maesoser", social= "https://github.com/maesoser" }
+++

My life is balanced between London, the city in which I work and live, and Madrid, where my family and an important part of my friends live. For that reason, my homelab is also split between these two cities and some of the services I am running in it need secure and private connectivity between them. I've been using ad-hoc solutions like SSH tunnels thanks to [mole](https://github.com/davrodpin/mole) or writing specific pieces of software like promrelay[^1] but in the end it was too difficult to keep this configuration up and running smoothly.

I decided to standardize it a little bit more and configure a site to site VPN. I've been running two VPN gateways (one in Madrid and the other one in London) for quite a while. These gateways allow me not only to connect to my lab in each country but also to access the Internet using an IP address located in each specific country, which is pretty handy when you are consuming geofenced Internet services.

I didn't want to open any additional ports or create new interfaces. The gateways already in place should allow me not only to connect to the Internet from each country using my laptop or my phone, but they should also be connected to each other and give me seamless connectivity.

![infra](/images/site-to-site-wireguard/infra.png)

There are also some tiny details that add a little bit of complexity to the mix like that each gateway uses different software (MicroTik in London and OpenWrt in Madrid) or that the gateway and the router are different boxes in Madrid.

## A little bit of theory about WireGuard 

Although [Wireguard's documentation](https://www.wireguard.com/#conceptual-overview) is probably way better than anything I could write I would like to do a quick summary about a couple of important concepts that WireGuard implements. 

The first is the type of encryption WireGuard implements. This protocol is similar to any other software or protocol using asymmetric encryption, like SSH or HTTPS for instance. In this type of cryptography, each device has a unique pair of keys: a private key and a public key. The private key is kept secret (so it should not appear on any configuration except the one present in the device itself) and is used to decrypt messages that have been encrypted with the device's public key. The public key, on the other hand, is shared with other devices and is used to encrypt messages that are intended for the device. Understanding this is way better than trying to remember by heart which key goes where.

The commands needed to create a new key pair plus a pre-shared key are the following:

```bash
umask 077; wg genkey | tee "${device}.key" | wg pubkey > "${device}.pub"
wg genpsk > ${device}.psk
```

The second concept is the meaning and importance of the "Allowed IPs" parameter that each peer has, which specifies a list of IP addresses or subnets that are allowed to establish a connection with the WireGuard interface. This parameter can be used to restrict access to the WireGuard interface to only authorized parties. This is important because in our case we will configure there the networks that are exposed through the specific peers we will configure.

## Configuration

In this section we will go through the four main steps needed to configure a site to site encrypted tunnel. Conceptually the steps are the following:

1. Create the key pair needed in both Madrid and London.
2. Configure the VPN Gateways to be able to receive traffic.
3. Configure the VPN Peer in each gateway to be able to send traffic to each other.
4. Add the routes needed to forward the traffic through the tunnel.

### 1. Create the key pairs for the VPN Gateways

The first thing we need to do is to create the keys we will configure on our routers. Besides the public and private keys, we will also generate an additional key called the 'preshared key' which is an important part of the WireGuard security model and helps to protect against man-in-the-middle attacks and other types of security threats.

```bash
$ umask 077; wg genkey | tee madrid-gw.key | wg pubkey > madrid-gw.pub
$ wg genpsk > madrid-gw.psk
$ umask 077; wg genkey | tee london-gw.key | wg pubkey > london-gw.pub
$ wg genpsk > london-gw.psk
$ ls -1a | grep gw
london-gw.key
london-gw.pub
london-gw.psk
madrid-gw.key
madrid-gw.pub
madrid-gw.psk
```

### 2. Configure the VPN gateways in Madrid and London

The next thing we will do is to configure the WireGuard interface in Madrid. For that purpose, we will add these lines to `/etc/config/network` and reload OpenWrt's network service by running `service network reload`. It should be straight forward to translate this plain text configuration into the 'clicks' needed in the [LuCI web interface](https://openwrt.org/docs/guide-user/luci/start) that OpenWrt offers:

```bash
config device
	option name 'wg0'
	option ipv6 '0'

config interface 'wg0'
	option proto 'wireguard'
	option listen_port '50501'
	option private_key '(content of madrid-gw.key)'
	list addresses '10.0.0.2/24'
```

We will do something very similar in the London router, just access to the web dashboard, go to the WireGuard section and create a new WireGuard interface with the following configuration:

![MicroTik Wireguard iface config](/images/site-to-site-wireguard/microtik-1.jpg)

We will also need to perform another small step in MicroTik in order to configure an specific IP address for the wg0 interface. For that purpose we will go to **IP > Addresses** and assign an IP address to the interface wg0 which in this example it is 10.0.0.1/24.

### 3. Configure the peers in each gateway

The next thing we will do is to configure the Madrid peer in our London router and otherwise. Therefore in our OpenWrt VPN gateway in Madrid we will open the `/etc/config/network` file and add a new peer with some information about London like these three important details:

   - The **Public Key** of the London gateway.
   - The **Endpoint host** with the hostname of the London gateway.
   - The networks available through this peer by using the **Allowed IPs** setting.

```bash
config wireguard_wg0
	option description 'london-gw'
	option public_key '(content of london-gw.pub)'
	option endpoint_host 'london-gw.domain.com'
	option endpoint_port '50501'
	option persistent_keepalive '25'
	list allowed_ips '10.0.0.1/32'
	list allowed_ips '192.168.1.0/24'
```

In the London router we will do something extremely similar but using the MicroTik's dashboard, specifically in **Wireguard > Peers** section:

![MicroTik Wireguard peer config](/images/site-to-site-wireguard/microtik-2.png)

### 4. Routing configuration

As the number of routers is just two and the number of networks is also low, we will just install some static routes instead of using more complex solutions like OSPF or BGP.

To that effect, in Madrid, we will instruct our VPN gateway to route the packets that have 192.168.1.0/24 as destination through the WireGuard interface. This is the equivalent to `ip route add 192.168.1.0/24 dev wg0`:

```bash
config route
	option interface 'wg0'
	option netmask '255.255.255.0'
	option target '192.168.1.0/24'
```

As in Madrid my router and my VPN gateway are not the same we will also need to install some additional routes on the router itself in order to forward the traffic to the aforementioned VPN Gateway:

```bash
config route
	option interface 'lan'
	option target '192.168.1.0/24'
	option netmask '255.255.255.0'
	option gateway '192.168.2.2'
```

This last step is not needed in the MicroTik located in London. There we will only need to configure a single static route to send traffic through the WireGuard interface in **IP > Routes**:

![MicroTik Wireguard routes config](/images/site-to-site-wireguard/microtik-3.png)

After performing this last step we should already have connectivity between both network locations. We could test that by simply using the ping command:

```bash
user@madrid-gw:~# ping -c 5 192.168.1.1
PING 192.168.1.1 (192.168.1.1): 56 data bytes
64 bytes from 192.168.1.1: seq=0 ttl=64 time=42.473 ms
64 bytes from 192.168.1.1: seq=1 ttl=64 time=41.113 ms
64 bytes from 192.168.1.1: seq=2 ttl=64 time=40.865 ms
64 bytes from 192.168.1.1: seq=3 ttl=64 time=61.570 ms
64 bytes from 192.168.1.1: seq=4 ttl=64 time=41.332 ms

--- 192.168.1.1 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 40.865/45.470/61.570 ms
```

## Summary

In this article, we explained how to configure a site to site WireGuard tunnel between two locations using different routing software. We also provided a brief overview of some of the key concepts in WireGuard. By establishing this tunnel we will be able to seamlessly access networks located miles away without installing any software in the hosts connected to such networks.

[^1]: Promrelay is a piece of software I wrote to relay prometheus metrics using HTTP POST requests.