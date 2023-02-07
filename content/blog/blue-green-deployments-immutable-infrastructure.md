---
title: Blue-Green Deployments and Immutable Infrastructure
date: "2020-01-11"
---

I've blogged in the past about [why I'm running my own DNS infrastructure](/blog/run-your-own-dns-servers/), and [how I monitor those machines](/blog/low-cost-infrastructure-monitoring/). This post rounds out the trilogy with a discussion of how and why I automated the deployment of this infrastructure.

## Why immutable infrastructure?

Immutable infrastructure is the idea that once a server is configured and serving production traffic, it will never be re-configured. When you want to roll out an update, rather than mutating the existing server, the idea is to create a new server with the desired configuration to replace the old server.

Blue-green deploys are a deployment process where you spin up a new server before taking down your old server. Once you've confirmed the new server is healthy, you update your load balancer to point production traffic at the new server. This deployment process allows deploying immutable infrastructure without downtime (which you might otherwise have in the period between the old server shutting down and the new server turning on).

So now we know what immutable infrastructure is, and how we can deploy it, but what are the benefits?

My favorite feature of immutable infrastructure is that you can be confident that you are always capable of deploying production infrastructure from scratch. This sounds obvious, but if you setup your servers once and re-configure (mutate) them as your needs change, over time the configuration of these machines will drift. Given enough time (and mutations), it is likely that the configuration would drift to such an extent that you wouldn't immediately know how to re-provision these machines from scratch if that became necessary.

With immutable infrastructure, any time you want to make a configuration change the entire server is deployed from scratch. The ensures configuration drift cannot happen, and it is always possible to re-provision a failed machine.

## Infrastructure overview

Now that we've motivated immutable infrastructure and blue-green deploys, how exactly did I implement it in my personal DNS infrastructure?

If you read the previous infrastructure posts, you'll know my goal was to host two globally distributed authoritative DNS servers. The machines are hosted on [Linode](https://www.linode.com), and I use BIND as the DNS server.

BIND is capable of handling configuration changes without dropping traffic, but I wasn't sure how I'd be able to update BIND itself without downtime (much less apply things like kernel updates). This lead to deploying a load balancer (or reverse proxy) in front of each DNS server, which allows for blue-green deploys as described above.

The load balancers are themselves linux virtual machines, also hosted on Linode, running nginx.

## Automated deployment

All of my machine configuration is [managed with Ansible](https://github.com/JoshMcguigan/infra/tree/master/ansible). But Ansible is really a tool to provision machines, and doesn't provide features for blue-green deployment.

To fully automate the deploy of nameserver updates, I wrote a [python script](https://github.com/JoshMcguigan/infra/blob/master/scripts/deploy-infra.py) which interacts with the Linode API, as well as Ansible.

Now if I want to update my nameservers I just change the Ansible configuration, then run `./scripts/deploy-infra.py`, and I can be sure the changes are deployed safely and without downtime.

An outline of the script behavior follows:

1. Create two new virtual machines (ns1-next, ns2-next)
1. Provision the new machines with Ansible
1. Ensure both machines respond to a DNS request
1. Re-configure the load balancers to point to the new DNS servers
1. Ensure DNS health checks pass when run against the load balancer public IP addresses
1. Delete the old DNS servers
1. Rename ns1-next/ns2-next to ns1/ns2

## But the load balancers are mutable..

Yes, unfortunately something has to be mutable. I would have preferred to use a managed load balancer (Linode's managed load balancer doesn't support UDP) or some other network trickery (I did experiment with Linode's IP swap feature) to make this work, but ultimately I found I needed to run my own load balancer to get exactly what I was looking for.

Although this is not ideal, for me it is acceptable under the assumption that the machines behind the load balancers would change more often and more substantially than the load balancers themselves. Further, nginx can support both configuration changes as well as application updates without downtime.

## Blue-Green load balancer deploys

For kernel updates, or any other updates to the load balancers which require a system restart, they still can be re-deployed in a blue-green fashion, but the process will be a bit more involved (and not nearly as fast).

The process is as follows:

1. Deploy new load balancer instances (create machine, run Ansible configuration)
1. Ensure new load balancer instances are healthy
1. Log into my domain registrar and update DNS configuration to point to the IP addresses of the new load balancers
1. Delete the old load balancers once they are no longer receiving traffic (being cautious of long DNS client TTL settings)

## Conclusion

Is this setup overkill for my current needs? Almost certainly yes. But I've enjoyed building it. And it allows me to have great confidence in my ability to maintain this system, with a high level of uptime, over the long term.

As of this writing, this is the DNS infrastructure serving requests for this blog. Try it out with a direct request like `dig www.joshmcguigan.com @ns1.rhiyo.com`, or use `dig www.joshmcguigan.com +trace` to recursively resolve the domain from the root DNS servers.
