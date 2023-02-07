---
title: Run Your Own Authoritative DNS Servers
date: "2019-12-21"
---

The domain name system is a critical part of the internet, but it is often overlooked. In this post I'll explain my rationale for running my own DNS authoritative name servers, and then share the automated deploy and configuration scripts I created along the way, which would enable you to setup DNS servers of your own in under an hour.

## What is an authoritative name server?

Broadly speaking, there are two types of DNS servers. Recursive resolvers are the type users are more likely to be familiar with. Examples of recursive resolvers are [Cloudflare's 1.1.1.1](https://www.cloudflare.com/learning/dns/what-is-1.1.1.1/) and [Google's 8.8.8.8](https://developers.google.com/speed/public-dns). When your computer needs to perform a DNS lookup, it will ask a recursive resolver.

Authoritative name servers are the source of truth in the domain name system. When recursive resolvers get a request for a particular domain name, they will get that information from the authoritative name servers (assuming they don't already have it stored in their cache).

When you buy a domain name, in the most technical sense the thing you are buying is the right to act as the authoritative name server for that DNS zone.

If you are interesting in learning more about DNS, [this webcomic](https://howdns.works/ep1/) serves as a great introduction.

## Why run your own DNS servers?

It is possible to own a domain and run a website without giving much of a thought at all to DNS. This is because nearly every domain registrar offers free DNS hosting as a benefit to their customers. So given this, why would I want to run my own name servers?

My answer here is entirely non-technical. I am doing it to learn. And not just about DNS, but also about running a highly available, globally distributed service.

## Why NOT run your own DNS servers?

If not for learning, you almost certainly should NOT run your own DNS servers. As mentioned above, for smaller sites, your domain registrar probably provides DNS hosting for free. For users that need more control, greater uptime, or improved performance, there are paid DNS hosting providers that do a great job.

One particularly interesting reason that quality DNS providers are able to provide such high levels of performance and resiliency is that they use [Anycast](https://www.cloudflare.com/learning/dns/what-is-anycast-dns/) to route traffic to the closest available server. Unfortunately, [setting up an Anycast network](https://labs.ripe.net/Members/samir_jafferali/build-your-own-anycast-network-in-nine-steps) is out of reach for most, making it nearly impossible to provide top-tier DNS performance and reliability.

## Automated deploy

If after all that you still want to run your own authoritative DNS servers, the good news is you can get setup in less than an hour. I like to write scripts to automate things like this, even if I don't plan to run them very often, as I find short scripts to be the most trustworthy source of documentation.

Having the whole process automated also allows quick iteration which makes experimentation easy when you are first getting started.

#### Spinning up cloud server instances

I chose to host my DNS servers with [Linode](https://www.linode.com). Of course you can use any VPS provider, but the script in this section only works to provision instances on Linode. If you are following along and want to use a different VPS provider, provision two linux (I used CentOS 8) machines in different data centers, then jump to the next section.

Why do we need two servers? The DNS specification actually requires at least two authoritative name servers for each zone. See section 4.1 of [RFC 1034](https://tools.ietf.org/html/rfc1034) for details. 

```python
for linode_label, linode_region in [
            ("ns1", "us-west"),  # Fremont
            ("ns2", "eu-west"),  # Frankfurt
        ]:
    new_linode, _password = client.linode.instance_create(
        "g6-nanode-1",
        linode_region,
        label=linode_label,
        image="linode/centos8",
        authorized_keys="~/.ssh/id_ed25519.pub")

    print(f"Created {linode_label} with at {new_linode.ips.ipv4.public[0]}")
```

The python snippet above creates two cloud servers using the Linode API, configured with my local SSH key to allow remote access. See the [full source](https://github.com/JoshMcguigan/infra/blob/fe24c52c77257723ebd575c47dfc48359e0c15a7/scripts/bootstrap-linode-infra.py) for more details. The only setup needed before this is creating a Linode account and getting an API key for the script to use.

#### Provisioning DNS software and configuration

I use Ansible to script the server configuration.

```yaml
- hosts: all
  roles:
    - common

- hosts: nameservers
  roles:
    - nameserver
```

The `common` role just disables password based login via SSH for improved security. You can review the [role definition](https://github.com/JoshMcguigan/infra/tree/fe24c52c77257723ebd575c47dfc48359e0c15a7/ansible/roles/common) for details.

More relevant is the [nameserver role](https://github.com/JoshMcguigan/infra/tree/fe24c52c77257723ebd575c47dfc48359e0c15a7/ansible/roles/nameserver).

```yaml
- name: install bind
  package:
    name: bind
    state: present
  notify: bind enable

- name: configure firewall
  firewalld:
    service: dns
    immediate: yes
    permanent: yes
    state: enabled

- name: bind config
  copy:
    src: named.conf
    dest: /etc/named.conf
  notify: bind update

- name: rhiyo zone config
  template:
    src: master.rhiyo.com.j2
    dest: /var/named/master.rhiyo.com
    owner: root
    group: named
  notify: bind update
```

There are many authoritative name servers, but I chose to use [BIND](https://www.isc.org/bind/) due to its popularity. The first step in this role installs BIND and enables the service to start when the machine starts. The second step allows DNS through the system firewall, which by default blocks this traffic.

The third step provisions the BIND configuration file.

```
options {
	directory "/var/named";

	// version statement - inhibited for security
	version "not available";

	// disable all recursion - authoritative only
	recursion no;

	// disables all zone transfer requests
	allow-transfer{ none; };
};

zone "rhiyo.com" {
	type master;
	file "master.rhiyo.com";
};
```

Disabling recursion configures our server as an authoritative server only (it can't be used as a recursive resolver). I've configured my servers to respond to DNS queries for the `rhiyo.com` zone, but you'll of course want to replace that with your own domain(s).

The final step provisions the zone file.

```
$TTL	1h
@  IN  SOA ns1.rhiyo.com. hostmaster.rhiyo.com. (
			      0 ; serial
			      3H ; refresh
			      15 ; retry
			      1w ; expire
			      3h ; min ttl
			     )
       IN  NS     ns1.rhiyo.com.
       IN  NS     ns2.rhiyo.com.

ns1    IN  A      {{ hostvars.ns1.ansible_host }}
ns2    IN  A      {{ hostvars.ns2.ansible_host }}

# additional DNS records go here
```

This file is templated so I don't have to duplicate the public IP addresses of my name servers, since they are already specified in the [inventory file](https://github.com/JoshMcguigan/infra/blob/fe24c52c77257723ebd575c47dfc48359e0c15a7/ansible/inventory).

## What's next?

You may have noticed that the configuration above only serves DNS records for the nameservers, and not any other hosts. This is because I'm not quite ready to switch over my real sites (this blog, for example) to this DNS setup.

Before using this "in production" (quotes because I'll only be using it for hobby projects) there are a couple extra things I want to have.

First on my list is monitoring. While failover happens automatically in the DNS protocol (in the case one name server is down, clients will automatically try another server), I still want some way of being notified if one of the machines goes down. I also want to have some plan in place for performing configuration, software, and server updates without downtime. Keep an eye out for future blog posts on these topics.
