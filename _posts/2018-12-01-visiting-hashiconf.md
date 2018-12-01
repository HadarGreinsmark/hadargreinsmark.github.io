---
layout: post
title:  "Visiting HashiConf in San Francisco"
---
If you haven't heard of HashiCorp before, you might have heard of some of their open source projects such as Consul, Terraform or Vagrant that aim to simplify management of complex cloud applications. In October I visited their annual HashiConf conference -- this time in San Francisco -- along with more than 1200 other attendees. As an open source contributor to their *Consul* project, I was fortunate enough to get free tickets to the otherwise expensive conference. Something I couldn't resist to take advantage of!

![HashiConf 2018 keynote speetch]({{site.baseurl}}/assets/hashiconf-2018-keynote.jpg)

The 3 day schedule was fully packed with talks from some of the engineers behind the HashiCorp tools, as well as talks from many different companies like Amazon, Reddit and Google. In this post, I'll bring up some of their new solutions and research presented.

# Encryption within the Datacenter
One big subject that many people where talking about was how to improve internal security between services/microservices, and avoid a hacker from accessing everything if one of the services gets compromised.

A new feature presented was *Consul Connect* -- an extension to Consul for handling authorization and encryption between services. *Consul* is a Service Discovery system, that maintains an inventory of services that are alive. The *Connect* feature extends this by deciding which of these services are allowed to talk to each other based on user-defined rules. This is very similar to firewall rules. But instead of defining rules based on IP addresses and the notion of machines, Consul Connect allows to define rules based on service names and the notion of services. For example, we can specify rules that a 'web' service is only allowed to talk to a 'db' service regardless of which IP addresses they might currently have. As Consul work with the notion of service names, this allows for more flexible communication policies on a finer granularity.

![IP firewall versus Consul Connect]({{site.baseurl}}/assets/consul-connect.png)

To secure *service to service* communication further, Consul Connect adds *end to end* encryption to this. As Consul already have an agent on every machine that tracks each machine's services, this agent is used for generating and rotating encryption keys and distributing TLS certificates to all machines. Consul can in this way be used as a sidecar proxy that manages encryption for the service.

It might seem a little needless to deploy TLS encryption within the datacenter itself as TLS was designed for untrusted networks such as the Internet. But TLS offers identification of the service you're talking to, and encryption that might be necessary if one isn't pushing for very strict intermediate IP firewall policies. Something that seems like a good fit for a *service to service* approach. 


# Password Manager in the Datacenter
<img alt="Vault logo" src="{{site.baseurl}}/assets/vault-logo.svg" style="float: right;" width="200" height="200" />
For personal usage, password managers such as LastPass have become popular. These tools are generating a unique password for every website account you have, and store all these passwords in a secure and encrypted database. The user only needs to remember one master password to access this database. If a password gets leaked from one website, it's no catastrophe as that password is unique to just that website. With *Vault*, this mindset is brought to the datacenter.

Vault provides passwords to services if they can authenticate in a desired way. But there's also features like password rotation, key generation and "Encryption as a Service" to make sure general encryption is done properly without leakage. The assumption is that ordinary services aren't designed with security as a top priority. So instead, critical pieces should be managed by services that have security as top priority, like Vault.

A new feature currently being researched by HashiCorp is *Vault Advisor*, a way to auto-detect unused permissions and propose to limit these. As Vault is giving out passwords/credentials upon request from services, it's easy to spot which services never request certain credentials but are allowed to. Administrators could then get recommendations on how to limit unused permissions. One of the research questions they're trying to answer is how to find a balance between complex and restrictive permissions, versus simple and easy maintainable ones. In their talk, they were promising to release a research paper on their findings. Looking forward to read it!


# Infrastructure as Code
Even though infrastructure provisioning isn't directly connected to security, this was a major topic at the conference that would be good to mention. Their *Terraform* provisioner have gotten wide adoption, and was also the tool most conference visitors seemed the most excited about. Terraform provides a configuration language that lets you configure server resources. Instead of an operator having to e.g. spin up virtual machines in a web interface, the operator just edits his/her Terraform configuration file and deploys it as if it would have been software.

As AWS, Microsoft Azure and Google Cloud all provide official modules for configuring their infrastructure using Terraform, and the fact that all three where engaged enough to sponsor the afterparties for the conference, it seems like Terraform is very important to the major cloud providers. So this is absolutely something I have to look into more. If you want to check it out, there's lots of documentation on [their website](https://www.terraform.io/intro/index.html).


# Future of Infrastructure
It was fun to talk to other conference visitors and listen to what cloud challenges they where up to. I got lots of new insights into the trends of todays growing cloud applications. If you want to see some of the talks from the conference, they recently published [recordings on their YouTube channel](https://www.youtube.com/watch?v=o9x9agDCoNc&list=PL81sUbsFNc5ZzhVxTZJqeHG24iwzKL0ei).

![HashiConf at Fairmont Hotel]({{site.baseurl}}/assets/hashiconf-hotel.jpg)
*Mingeling area at the conference, which was held in downtown San Francisco.*