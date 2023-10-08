+++
title = "Shining a Light on the Most Visited Tracking and Advertisement Domains with Cloudflare Radar"
date = "2023-05-03"
description = "Abusing Cloudflare's Radar Domain Rankings dataset to identify the most commonly accessed tracking and advertisement domains."
draft = true

[taxonomies]
tags = ["internet", "projects"]

In today's digital landscape, the use of DNS filtering has become more important than ever before. Not only does it help to block harmful content and protect users against phishing attempts, but it can also play a significant role in enhancing online privacy.

With the rise of advertising companies and tracking mechanisms, users are constantly exposed to potential threats online. This is where DNS filtering comes into play, allowing users to control what content is accessible on their network and protecting them from unwanted distractions and harmful content.

Some time ago Cloudflare [released a dataset called Radar Domain Rankings](https://blog.cloudflare.com/radar-domain-rankings/) which is based on aggregated and anonymized 1.1.1.1 resolver data and that serves as a replacement to other domain rankings like [Majestic Million](https://majestic.com/reports/majestic-million) or Alexa Ranking. The goal is to classify the most frequently used domains across the globe without infringing on individuals' privacy.

Last weekend I decided to combine these rankings with commonly used lists of tracking and advertisement domains in order to create a comprehensive yet concise list of the most frequently accessed tracking or advertisement domains. The purpose of this list is to provide users with the smallest set of domains that would block most of the potentially annoying or privacy damaging domains that are in use right now on Internet.

In order to be able to create such blocklists I download the NextDNS repository