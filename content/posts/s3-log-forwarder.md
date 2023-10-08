+++
title = "Sending your network logs to S3 straight from your router"
date = "2023-09-06"
description = "Taking advantage of the extensibility options of current routers to enhance the monitoring capabilties of my network"
draft = false

[taxonomies]
tags = ["internet", "projects"]

+++

I think it is because of my previous work at Nuage, but I've always seen routers as not just routing devices. They have evolved from specialized devices to extensible network powerhouses which usually contains commodity CPUs to perform management tasks and in certain situations, it could be extremely helpful to deploy additional services in these commodity CPUs, taking advantage that network equipment is usually an infrastructure element that is resilient and runs 24/7. DNS/DoH resolvers (even with [filtering capabilities](https://pi-hole.net/), NTP or TFTP servers, [monitoring solutions](https://blog.maesoser.me/posts/openwrt-minimal-node-exporter/) or [out-of-bound management tools](https://blog.maesoser.me/posts/cloudflared-openwrt/) are some examples of services that could be offered by routers.

Nowadays, many router operative systems like VyOS or OpenWrt are based on Linux, so it is pretty straightforward to add news services to them. In the corporate landscape, companies like [Juniper](https://www.juniper.net/documentation/us/en/software/junos/overview-evo/topics/task/third-party-applications-deploying.html), [Cisco](https://www.cisco.com/c/en/us/products/collateral/switches/catalyst-9300-series-switches/white-paper-c11-742415.html) or [Mikrotik](https://help.mikrotik.com/docs/display/ROS/Container) also provide to their customers the possibility of running containers inside some of their devices.

As a network enthusiast (and an immigrant), my current network infrastructure spans across different countries and is composed of Mikrotik and OpenWrt devices. To establish seamless connectivity between the networks, I've implemented a VPN mesh using the WireGuard protocol. Although this architecture is more advanced than the one present at a common household, it is still built on top of the ISP offering for standard customers, and thus it has a bigger chance of experiencing downtime.

For that reason, it is important to have a centralized logging system that collects logs from all the network devices in order to streamline the resolution of network disruptions. Given the specific characteristics of my network, the logging solution should fulfil some specific requirements:

- **It should be out-of-bound**, meaning that it should not rely on the WireGuard tunnels that compose my network. I'd like to receive logs in the event that the WireGuard tunnels are down, but the connectivity to the Internet remains intact.
- **It should be encrypted** as we don't fully control the amount of sensitive information that could be contained in the logs
- **It should provide the means for receiving logs from "dumb devices"** that do not allow the installation of additional software

Given these requirements I decided to create a small syslog-to-s3 daemon. The daemon will sit in my spine network devices as a syslog service, it will receive syslog messages from localhost or other devices in the network, turn the syslog format into JSON format and if the number of messages reaches a certain threshold or the messages hasn't been sent for a certain amount of time, it will gzip the messages and sent them to an S3 compatible location like an S3 bucket, R2 or a remote instance with Minio installed for instance.

![diagram](/images/s3-log-forwarder/diagram.png)

I built the daemon using Golang because it allows me to implement my logic in a few lines of code, and it is easy to cross-compile the code to the CPU architectures needed to be deployed in my routers like ARM, ARM64 or even the more exotic MIPS. This is an example of the message format that is being sent by my daemon:

```json
{
    "client":"192.168.4.7:39722",
    "hostname":"crisipo",
    "facility":10,
    "priority":85,
    "severity":5,
    "tag":"dropbear",
    "timestamp":"2023-10-04T00:24:15Z",
    "content":"Pubkey auth succeeded for 'user1' with key sha1!! 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 from 192.168.4.226:60334",
}
```

In conclusion, the ability to extend the functionalities of routers offers immense possibilities for enhancing network capabilities. By developing a syslog-to-S3 daemon, I have significantly increased the visibility of my network. This enhanced visibility allows me to quickly identify and diagnose any issues or security threats that may arise. Moreover, it was an enjoyable experience to work on! I plan to write a future post discussing the software stack that enables me to ingest and analyze these logs.