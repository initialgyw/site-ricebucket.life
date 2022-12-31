---
title: Hello World
description: This site will mostly be used to document my homelab changes.
date: 2022-03-07
authors:
  - konri
toc: true
draft: false
---

In order to understand how new technologies work, I need a hands-on practice environment to implement the latest technological changes. My homelab will be the pillar of my knowledge acquisitions. Maybe a career portfolio?

<!--more-->

## Homelab Philosophy

The idea for my homelab will be an hybrid on-premise and cloud environment. I will try to utilize free/affordable cloud services and incorporate it into my homelab.

### Existing Services

I already have some existing services that I need to note.

- **Github for code respository**
  
  Github provides a free service to store my code publicly and privately. It was a hard choice between Github and GitLab but I'm sticking with Github due to popularity and maybe integration with Azure (if any) for future projects.

- **Cloudflare for DNS**

  I bought my domain using [porkbun.com](https://porkbun.com) registrar but decided to use Cloudflare's DNS service due to a lot of application support (Terraform, Ansible, etc). Cloudflare also provides email routing and web page caching, which I do not get with my domain registrar. I looked into [NS1](https://ns1.com) which has geographic filtering, which is pretty fancy for split horizen, but I'm going to stick with Cloudflare for now. I'll write a blog if I ever migrate to use NS1.

  I will also be using [Cloudflare Pages](https://pages.cloudflare.com/) to deploy this site.

- **Lucidcharts**

  This SaaS will be used for editing my homelab topology.

## Why Cloudflare Pages with Hugo

A fresh start on my homelab means I don't have any servers to host contents, as these servers are not yet configured. [Cloudflare Pages](https://pages.cloudflare.com/) does not require me to run any servers or database backend and thinking about the future, maybe it's time to start embracing PaaS.

Cloudflare Pages aligns with my homelab philosophy; *utilize free/affordable cloud services*. It has a lot of free features, such as caching and DDoS protection for site hosting. This website are static pages so I don't need to care about security as much; I don't need to worry about site performance or network bandwidth usage. I am also currently heavily vested in NET, so it would be a good idea to understand the products of Cloudflare.

*Since the contents will be stored on a Github repository, why not use [Github Pages](https://pages.github.com/) to centralize the location of storage and deployment?* Github Pages does not have traffic analytics. I need to add in a third-party code to get traffic data, such as Google Analytics. Cloudflare has built-in analytics through their proxies so embracing that would make my site more miminalist.

I picked Hugo (over other static site generators, Jekyll, Pelican) due to popularity (more community support). Cloudlfare allows me to Continuously Develop and Continuously Integrate to automatically generate static pages and push it publicly just by committing the changes to the Github repository.

## What else am I using this site for?

- improve my technical writing
- journal my stock market ~~gambling~~ ideas.
- post photos that I took in which I think are good in my *PERSONAL* opinion.
