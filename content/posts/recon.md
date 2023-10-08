+++
title = "Recon + Bug Bounty tools"
date = "2023-01-19"
description = "Because starting a blog it's not difficult anymore"
draft = true

[taxonomies]
tags = ["internet", "projects"]

+++

```
I just found an unbelievable number of unauthorized API endpoints using this 1 liner.

katana -u $url -hl -nos -jc -silent -aff -kf all,robotstxt,sitemapxml -c 150 -fs fqdn |subjs | python3 /opt/JSA/jsa.py |goverview probe -N -c 500 |sort -u -t';' -k2,14 |cut -d ';' -f1
```

```
1) Download this
https://gist.github.com/nullenc0de/2981930386c439af08c031622440bc2e

2) python3 http://365doms.py -d http://tesla.com |grep -v http://onmicrosoft.com |subfinder |dnsx| httpx |nuclei
```

FortiNAC:

```
subfinder -d ubisoft.com -silent -all | httpx -silent -ports 443,8443 --path "/configWizard/keyUpload.jsp" --mc 200
```

S3 buckets:
```
subfinder -d disney.com -all -silent | httpx -silent -web-server -threads 100 | grep -i AmazonS3

subfinder -d disney.com -all -silent | httpx -silent -threads 100 -ms "AccessDenied"
```

https://github.com/arkadiyt/bounty-targets-data

https://github.com/dionach/CMSmap


XSS hunter: https://infosecwriteups.com/easy-xsshunter-express-setup-script-d5a66039f7b6


katana -u http://test.com -headless -jc -aff -kf -c 50 -fs dn