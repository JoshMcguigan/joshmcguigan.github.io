---
title: Low-cost Infrastructure Monitoring
date: "2019-12-28"
---

In my last post I [described how I'm running my own authoritative DNS servers](/blog/run-your-own-dns-servers/), and I left off with a note about how I wasn't ready to use these servers in production because I didn't have any monitoring configured. 

In this post, I'll explain why I decided to build a custom monitoring solution, and how you could create your own low-cost (free) monitoring tool.

## Goals for the system

Generally speaking, I wanted a monitoring solution which could confirm that each of my DNS servers was responding to queries. This is different from checking that my domains can be resolved, as DNS has automatic fail-over built into the client, and I want to know when even one of my servers is down.

There are many monitoring-as-a-service vendors, but adding an additional vendor to your stack has some cost. Even if I would qualify for the free tier, there is the mental cost of learning a new platform.

Additionally, I have some future infrastructure provisioning automation plans which I think could benefit from building on top of a monitoring tool that I fully control.

## The architecture

I built a custom command line application in Rust, which acts as a DNS client and checks that my DNS servers respond correctly. If there are any failures, it uses the GitHub API to create an issue on my [infrastructure repo](https://github.com/JoshMcguigan/infra).

I then deploy this application to Heroku, using the `Heroku Scheduler` to automatically run it every 10 minutes.

The full source of the monitoring application at the time of writing is available [here](https://github.com/JoshMcguigan/infra-tools/blob/ed341b2abe0816089adceed92e8af3789a63fdcb/src/main.rs) and an example of an automatically generated issue created by the tool is [here](https://github.com/JoshMcguigan/infra/issues/4).

## Deploying the monitoring service

While this monitoring tool is certainly the kind of thing you could run on a Raspberry Pi (in particular since it doesn't need a static IP), I chose to deploy it to Heroku. This is because I already have a couple other projects running on Heroku, including one very similar to this (a scheduled app rather than a long running server). Also, since my infrastructure is hosted with [Linode](https://www.linode.com), hosting the monitoring service elsewhere prevents common mode failures.

Deploying a service like this to Heroku is fairly easy:

1. Create a new Heroku app
1. Install the `Heroku Scheduler` add-on
1. Connect the new Heroku app to the git repo for your application
1. If you are using Rust (as I am), configure Heroku to use a [Rust buildpack](https://github.com/emk/heroku-buildpack-rust.git)
1. Set your environment variables (config variables) in Heroku (for me, this is `GITHUB_API_KEY`)
1. Configure the `Heroku Scheduler` to run your binary (for me, `./target/release/monitoring`)

## Future work

For now this tool is hard-coded to check DNS authoritative name servers, but the ideas presented are general enough that they could easily be used to monitor web servers or really any internet-accessible service. In the future, I'll likely expand this service to act as an HTTP(S) client so I can be alerted if my web servers go down.

As it sits I believe this tool has satisfied my immediate goals. I now have monitoring configured for the DNS servers, it was developed/deployed without bringing a new vendor into my stack (I was already using GitHub and Heroku), and since I have full control of the code I've kept open the door for many future improvements.
