+++
title = "Enable DoH validation in Mikrotik"
date = "2023-07-08"
description = "Installing the CAs needed in Mikrotik to avoid receiving validation errors"
draft = false

[taxonomies]
tags = ["internet", "projects"]

+++

Mikrotik DNS over HTTPS does not have the `Verify DoH Certificate` option enabled by default. If you enable it you could have an unpleasant surprise. All of your DNS queries fail and when you look in your logs you see the following message:

```
DoH connection error: SSL: handshake failed: unable to get local issuer certificate (6)
```

What happened? Apparently Mikrotik does not include by default the [Root Certificate](https://en.wikipedia.org/wiki/Root_certificate) needed to validate the certificate sent by the Cloudflare domain which is providing DNS over HTTPS on my network. First we are going to figure out which Certificate Authority is using Cloudflare:

```sh
$ openssl s_client -showcerts -connect cloudflare-gateway.com:443 | grep depth
depth=2 C = IE, O = Baltimore, OU = CyberTrust, CN = Baltimore CyberTrust Root
verify return:1
depth=1 C = US, O = "Cloudflare, Inc.", CN = Cloudflare Inc ECC CA-3
verify return:1
depth=0 C = US, ST = California, L = San Francisco, O = "Cloudflare, Inc.", CN = sni.cloudflaressl.com
verify return:1
```

Baltimore CyberTrust root certificate is available in [Digicert's official webpage](https://www.digicert.com/kb/digicert-root-certificates.htm). Download it and install it should be easy, opening a terminal in the router and running the following commands:

```sh
/tool/fetch mode=https url="https://cacerts.digicert.com/BaltimoreCyberTrustRoot.crt.pem"
/certificate/import file-name=digicert-root-ca.pem
```

This will solve the problem _right now_ but what if Cloudflare [starts using a new certificate with a different certificate authority?](). [Curl offers a .pem file with all the CA certificates that are bundled in the Mozilla Firefox Browser](https://curl.se/docs/caextract.html) so a long-standing solution would be to load these CA certificates just in case.

This will solve the problem _for now_, but what if Cloudflare decides to [use a new certificate from a different certificate authority](https://developers.cloudflare.com/ssl/reference/migration-guides/digicert-update/)? You can use the [.pem file provided by Curl](https://curl.se/docs/caextract.html), which contains all the CA certificates bundled in Mozilla Firefox Browser. Loading these CA certificates would be a recommended solution for the long term.

```sh
/tool/fetch mode=https url="https://curl.se/ca/cacert.pem"
/certificate/import file-name=cacert.pem
```

If you also wish to ensure that all the DNS traffic on your network goes through the DNS over HTTPS resolver, you can implement the following firewall rules on your Mikrotik router. By doing so, you will prevent users from bypassing the resolver in the Mikrotik by overriding the DNS settings on their devices:

```sh
/ip firewall nat

# This rule replace the IP address to the value specified in the to-address parameter
add chain=dstnat action=dst-nat to-addresses=${gateway_ip} to-ports=53 protocol=udp src-address=!${gateway_ip} dst-port=53 log=no log-prefix="" 
add chain=dstnat action=dst-nat to-addresses=${gateway_ip} to-ports=53 protocol=tcp src-address=!${gateway_ip} dst-port=53 log=no log-prefix=""

add chain=srcnat action=masquerade protocol=udp src-address=${network}/${prefix} dst-address=${gateway_ip} dst-port=53 log=no log-prefix="" 
add chain=srcnat action=masquerade protocol=tcp src-address=${network}/${prefix} dst-address=${gateway_ip} dst-port=53 log=no log-prefix=""
```