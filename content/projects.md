---
title: Projects
draft: false
date: 2020-09-18
---

## Global Traffic Management

With multicluster support in Linkerd, I had the opportunity to do a little
discovery one what you can do with that. I implemented a
[global traffic management](https://drive.google.com/file/d/1FQoOfoph9IAVfT9bkfNVGMrk0ZHabCVa/view?usp=drivesdk)
solution. You can control the routing between services in any cluster and manage
the policy that sends requests to the right location.

To do this, I had to implement a
[custom HTTP client](https://github.com/grampelberg/fetch-stream). As a solution
for multiple clusters, I needed a way to stream responses from multiple requests
to each cluster back to users. The alternative would be to wait for every
cluster to respond, but that would end up negatively impacting a user's
experience. Additionally, I needed a way to watch Kubernetes resources and
events. This API streams results back over a single request and requires working
with fetch at a low level.

Did you know that browsers have a max number of concurrent requests? I didn't!
If you're watching a couple different resources on a few clusters, you can very
quickly run into this limit. While the limit is pretty low on a per-hostname
basis (foo.com), it is actually pretty high on a per-browser basis (256 for
chrome). So, instead of having all requests go to a single domain (foo.com) I
started dynamically generating domain names (1.foo.com) to get around the
browser's limitations.

## Service Mesh Interface

The service mesh space has many implementations and no common API. This is a
little frustrating for end users looking for a common solution. It is extremely
frustrating for those looking to build on top of service meshes. How do you
support multiple implementations without going crazy?

I authored the [Service Mesh Interface](https://smi-spec.io/) to solve these
problems. By distilling the common functionality of a service mesh into an API,
I worked closely with Microsoft and HashiCorp to help bring a common interface
for popular service meshes into reality. This allows users to work with a tool
such as [Flagger](https://github.com/weaveworks/flagger) to implement canary
rollouts for their applications without worrying which mesh they've implemented.

One of the first APIs that I built out was the
[metrics](https://github.com/servicemeshinterface/smi-metrics) API. This is
implemented as an
[APIService](https://kubernetes.io/docs/tasks/extend-kubernetes/setup-extension-api-server/)
and works very similarly to how the
[metrics server](https://github.com/kubernetes-sigs/metrics-server) for
Kubernetes itself operates. Instead of abusing the Kubernetes API as a database,
requests are proxied to an abstraction layer that converts from Kubernetes
primitives (pods, deployments) into Prometheus queries. Standard
[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) can be
used to restrict access to sensitive metrics and service mesh implementations
don't need to standardize their Prometheus labels/metrics.

## Autoscaling on what matters

Enabling autoscaling on CPU and memory doesn't work for every workload. In fact,
it takes a metric that may be a second or third level concern and tries to
automate sizing based off that. Many organizations have complex, sophisticated
solutions for AWS VMs. These solutions don't translate directly to Kubernetes.

Instead of scaling based off CPU, I propse using the
[golden signals](https://landing.google.com/sre/sre-book/chapters/monitoring-distributed-systems/)
in my [Kubecon](https://www.youtube.com/watch?v=gSiGFH4ZnS8) talk. In addition
to explaining why you'd want to do this, I show an example of how to do it with
Linkerd. This solution is general and works for anyone who has Prometheus
metrics to query.

## Multicluster Service Mesh

Kubernetes provides an abstraction layer that takes care of many tasks we used
to do all the time. I remember having to go find physical disk drives and plug
them into servers if I wanted more space! The abstractions that Kubernetes
provides has provided teams with more time to implement patterns that required
extremely large teams, even 5 years ago. Working with Linkerd, one of the most
common requests we receive is around multi-cluster support. Service meshes
shouldn't stop at the cluster boundary, they should make it easy for traffic to
communicate wherever a service is running.

Introducing moving parts to an already complex system, like Kubernetes, needs to
be done thoughtfully. To architect a solution that works in most customer
environments and fails in a reliable manner, I created a list of
[requirements](https://linkerd.io/2020/02/17/architecting-for-multicluster-kubernetes/).
These could then be used to understand if specific architectures would work in
the real world or not. I was also able to take these and work with the community
to verify that any solution we built would work for them.

Taking the requirements and working extremely closely with Kubernetes
primitives, I was able to
[design a solution](https://linkerd.io/2020/02/25/multicluster-kubernetes-with-service-mirroring/)
that would work for most Linkerd users. By leveraging as many primitives as
possible, I was able to keep the implementation small and easy to reason about.
Check out the
[community discussion](https://docs.google.com/document/d/1uzD90l1BAX06za_yie8VroGcoCB8F2wCzN0SUeA3ucw/edit#heading=h.bw0xc1qalza2)
for a glimpse into how we pulled everything together.

## Ksync

I like to pay attention to common tasks and think about ways to automate those,
so that I can write code more quickly. As I started building applications for
Kubernetes, I spent a lot of time building new containers, deploying them and
waiting for the containers to start up. Surely there is a better way to do this?

When doing development, it is rare for an entire container to need to be
rebuilt. If you're working with an interpreted language, like Python, there's no
need to even do a compile step. Locally, we just restart the server. For
compiled languages, subsequent compilation is actually pretty quick. So, instead
of rebuilding containers from scratch every time, I decided to sync the files I
was working on locally to the remote container. This is all
[ksync](https://github.com/ksync/ksync) does at a high level. The details are
pretty interesting though! Check out
[TGIK8S](https://tanzu.vmware.com/developer/tv/tgik/0029/) to hear Joe Beda talk
about ksync and what problems it solves.

To modify a container's running filesystem, you've got a couple options. One
would be to exec a process such as `rsync` inside the container and receive
files. This assumes that the container is setup in a specific way and fails
anytime someone uses a container based on `scratch`.

Another option is to directly modify the container's FS from the host.
OverlayFS, the default backend for Docker, extracts a container's separate files
onto the host's FS. It then joins the different layers together so that they
look like one FS to the running container. You can sync files directly into
these overlays and have them show up inside the running container. Using this
method, the container doesn't require any outside knowledge of updates.

Kubernetes provides all the tools required to get this up and running. Ksync
adds a daemonset to the cluster. This runs the agent (as a privileged container)
on every host. The agent connects directly to the Docker daemon and discovers
everything required to sync files for application containers. The agent also
includes a bidirectional syncing application that can move files from a
developer's laptop into the container itself. With hot reload, interpreted
languages can have containers fully updated and restarted in under a second.
This is massive for developer's productivity!

## Synse

Vapor places datacenters at the bottom of cell towers. This provides carriers
with a place for their equipment and latency sensitive applications the best
possible location. It also presents a unique set of constraints. There's
thousands of towers, just in the US. Unlike a traditional colocation facility or
datacenter, there's no way to have humans on location to fix things if anything
goes wrong. More importantly, controlling the temperature and humidity becomes
more difficult in small installations around cell towers.

Needing a way to automate the control and management of physical devices such as
fans, temperature sensors and power management, we created
[Synse](https://github.com/vapor-ware/synse). It serves as a bridge from HTTP
protocols to various low level protocols like RS-485 and IPMI. Using this API,
we could then automate controllers for the actual physical devices.

I created a GraphQL version of this API so that we could implement visualization
of what was happening in every location for customers. The first customer of
this also happened to be me as I built out the management dashboards for Vapor's
customers.

## Package Management

The cornerstone of DC/OS was the package management system. We needed a way to
bundle and apply common software across clusters. This was important to users as
it allowed them to get started with complex software, such as Hadoop or Spark,
quickly. It was also important to expand Mesosphere's market by providing
vendors, such as Databricks, a fast way to get their customers up and running on
the latest technology.

We settled on a very naive implementation early on. The first version didn't
even allow you to modify settings. After a little time, we settled on JSON
templates that allowed users to change specific configuration. I hold the
[patent](http://patft.uspto.gov/netacgi/nph-Parser?Sect1=PTO1&Sect2=HITOFF&d=PALL&p=1&u=%2Fnetahtml%2FPTO%2Fsrchnum.htm&r=1&f=G&l=50&s1=9880825.PN.&OS=PN/9880825&RS=PN/9880825)
on package management for distributed systems. The
["universe"](https://github.com/mesosphere/universe) as we called it to mirror
Debian's channel naming eventually grew to quite a few packages! Many of the
design decisions we made here can be seen in [Helm](https://helm.sh/) when
working with Kubernetes.

## Mesos Performance

A defining quality of Mesos is the scale it can operate at. At Twitter, it
orchestrated clusters of over 60k nodes and ran all of Siri for Apple. While it
could orchestrate many containers across a whole fleet of nodes, there were some
performance issues.

To show off what you could do with Mesos, we set a goal of launching 50k
containers in under 90 seconds. The first time we tested doing this, it didn't
quite work out. In fact, we never got there. The whole cluster stalled out after
launching maybe 10k containers with Mesos' CPU usage completely pegged.

After doing some debugging into this problem, I realized that Mesos was using a
naive algorithm for port assignment. To start a container, a random port would
be chosen. This port would then be reserved on specific nodes. Because the port
was random, each started container would increase the number of port ranges
Mesos would have to track (100-200 would become 100-149, 151-200). Picking
random ports was the most optimal way to manage reservations on that side, so
the agorithm for distributing port ranges needed to be updated. With a few
simple changes to that algorithm, we were able to launch containers at quite the
clip!

Check out the [keynote](https://youtu.be/_tfneEN0uPs?t=372) to see the
containers launch live. The team that build out the visualization itself did a
small
[writeup](https://d2iq.com/blog/visualizing-50000-live-containers-how-we-built-our-mesoscon-15-demo)
on how they pulled that side off. I'd highly recommend taking a read!

## Installation

The first customer request we received for DC/OS was to be able to install it
with something other than CloudFormation. Come to find out, most folks didn't
have the infrastructure in place to use a CloudFormation like tool and wanted to
use DC/OS on their existing datacenters. We had customers excited about using
DC/OS, we just needed to make it work for them now.

I started working with the team to build out an installer for DC/OS. This needed
to be something that could replicate all the required software for DC/OS across
clusters over 1k nodes in any customer's environment. With customers to work
with directly, I became the interface between the engineering team and the
customer. At one point, I think I was spending more time on video calls with the
customer than at home.

Taking the initial DC/OS packages, we ran into some unique problems on the
customer's site. From memory, one of the more interesting ones was related to
systemd in CoreOS. Systemd was still a pretty new init system and had some bugs.
To allow for us to upgrade DC/OS in place, we used symlinks for all the units
which defined how to start the DC/OS components running on the host OS. When the
customer would start 100 nodes at once, only about 75 would come up. The other
25 would start some services, not start others and generally be difficult. Come
to find out, the version of systemd that shipped with the latest CoreOS release
at the time couldn't handle symlinks to unit files. This was 100% a timing issue
though, if certain processes would start up in the right order everything ws
fine. Altering the DC/OS installer to only copy unit files themselves ended up
clearing the whole problem up.

Between direct feedback from customers and the sales team, we continued to
iterate on the DC/OS installer to get it into a state that could run in
production anywhere. This definitely took some time and a ton of amazing effort
from the team. To this day I think the installer itself for DC/OS is better than
many other projects.

## DC/OS

Mesosphere started out on the solid foundation of Mesos. Early on we did some
experiments to learn what the best business model was. We added plugins for
Mesos, created some products to help with incident management in a scheduled
world and had some limited success with
[Elastic Mesos](https://d2iq.com/blog/mesosphere-for-digitalocean). None of
these were particularly sustainable business models for the company,
unfortunately.

Taking some of the learnings from the Mesos CLI, I started to ask folks what
makes a platform feel like an operating system. My hypothesis coming into this
was the GUI. Imagine my shock when the unanimous answer was the package
management system. Installing complex applications is such a common task in
modern operating systems, it was the base requirement for day to day usage.
Taking Mesos, building a package management system for it and bundling the tools
required to actually run in production seemed like a good idea.

To test whether DC/OS was a good idea or not, I built the original version out
myself. This took the form of a React SPA running on top of a RoR app for all
the visualization. A [basic CLI](https://github.com/mesosphere-backup/dcos-cli)
provided write access to the Mesos API. I then automated the whole thing on top
of GCE by using
[startup scripts](https://cloud.google.com/compute/docs/startupscript). After
asking for a couple increases in quota, it was possible to create 1k+ node
clusters in under 5 minutes.Finally, there was no way I could create all the
packages required to get users thinking about what they could do. So, instead of
writing frameworks and packages for everything, I created a chameleon package.
It could pretend to be different workload patterns and names to present how
workloads would work on your mesos cluster. Humorously,
[this package](https://hub.docker.com/r/thomasr/fserv) has actually been
downloaded more than 1m times! To get a feel of everything we'd put together,
you can watch the
[original demo](https://www.dropbox.com/s/3vq4gxf6wq43t7w/flo-intel-demo.mp4?dl=0).

This all happened at just the right time. The idea of container scheduling was
starting to catch on and I was one of the first to show _how_ it worked, leading
users to understand what it could do for them. We decided to make this all a
reality and I started to define what DC/OS as a product would look like.

As I worked with the team to build something everyone could use, we continued to
do some marketing efforts to show the larger story. One of these was through a
partnership with Microsoft. I built out the script, demo application and
infrastructure for Corey Sanders to go on stage at //Build and show
[mesos containers launched](https://www.dropbox.com/s/tfmh4u5anqmekq0/container-demo-initial.mp4?dl=0)
on Azure infrastructure.

For a take on how I saw scheduling changing how engineers build things, take a
peek at my [MesosCon talk](https://www.youtube.com/watch?v=M_KjGMImOmA). I try
to explain why schedulers are empowering and break down the barriers that we'd
built up. They provide the tools for engineers and operations to work together
to get more reliable software out faster.

The initial version of DC/OS made some interesting assumptions. The largest of
these was that users would like to treat clusters the same way they wanted to
treat containers - as cattle, not pets. I architected this version to use AWS'
[CloudFormation](https://aws.amazon.com/cloudformation/) to provision the entire
datacenter at once. Hosts used [CoreOS](http://coreos.com/) to make even the
base OS something that could be replaced at any time. When we launched the first
version, we ended up getting a call from AWS. Apparently it isn't true that you
can get all the infrastructure you want from the cloud! We'd caused them to run
out of specific instance sizes in most of the popular regions at the time.

## Mesos CLI

One of the most difficult tasks when working with many of the early container
schedulers was debugging your application. It could end up on any host in the
cluster and would have its own location that you'd need to discover. At the
time, this required looking up your process, hoping you have SSH access to the
target host, finding the correct directory a log resided in and finally hoping
that the correct tools were installed to work with the logs or process itself.

I thought this whole thing could be optimized a little bit. More importantly,
we'd been talking about what it meant to have an operating system that was the
size of a datacenter. If every host in a datacenter is just part of an operating
system, why weren't there standard unix tools to work with it all?

My first try at getting this idea to work out was the
[Mesos CLI](https://github.com/mesosphere-backup/mesos-cli). Instead of
requiring a ton of steps to find a specific log for a process, why not just take
the name of the process and tail the logs for the users? The first sub-command
that was implemented, `tail` allowed users to interact with the log stream using
local tools. Just `grep` for what you needed or use your favorite environment
tooling to take care of it.

From an implementation perspective, Mesos actually made this whole thing pretty
easy. There was a single `state.json` endpoint that did a dump of the entire
cluster's state. The CLI could then use this to discover which node to talk to,
contact the agent on that node and then setup a standard stream from either
specific files or stdout/stderr.

As Mesos was used on very large clusters (Twitter ran one that had more than 60k
nodes), it was important to build out some of the UX tooling to make the CLI a
little easier to work with. This resulted in some features that we know and love
with modern CLI projects today, such as autocompletion and server side
filtering.

My favorite issue with the Mesos CLI came from TwoSigma. They are a large hedge
fund that used Mesos for data analysis and prediction. Researchers would create
jobs that'd run on the Mesos cluster. These researchers would then need to fetch
the results of their jobs and either debug the issues or feed the results into
new jobs. To keep memory under control in Mesos, there was a limit to the size
of `state.json`. Old jobs would be bumped and no longer be available. The Two
Sigma cluster was doing so much work that researchers who'd go to lunch would no
longer be able to get their logs through the CLI. The files would still be there
on the nodes, but Mesos would have no history of them anymore. We implemented a
special history store to allow the CLI to rely on more than just Mesos to fetch
history. This moved the onus onto external infrastructure that was easier to
scale, providing researchers with data even after they'd gotten lunch.

Following in the footsteps of the CLI, was there anything else that an operating
system could do that we could with Mesos? I created
[DCSH](https://www.dropbox.com/s/r4wgnbfmjb8hgrc/dcsh-demo.mp4?dl=0) to test
this theory. Instead of requiring a complex manifest and container to run things
on the cluster, why couldn't we run scripts just like `#!/bin/bash`? Using a
very small python client, the shebang ends up loading all the script's content
into a container. The script is then executed and stdin/stdout/stderr is hooked
up for the user. There's no way to know that you're running entirely on a large,
remote cluster!

## uTorrent App Platform

We had a problem at BitTorrent. Everyone loved the protocol and clients
(BitTorrent and uTorrent), unfortunately the only way we could come up with to
make money off the ~150m users we had at the time was through toolbar revenue.
Basing a business off a single revenue stream that was advertisement based and
somewhat shady made us all pretty uncomfortable. It was time to experiment!

At this point in history, app stores were a big deal. The iPhone had been pretty
successful and everyone wanted a piece of that. So, we asked ourselves, why not
a torrent client? Well, we
[built it](https://readwrite.com/2010/05/14/even_utorrent_has_an_app_store_now/).
As always, there were a couple interesting challenges that arose.

uTorrent was written in c++ and this had been awesome for building a small, fast
bittorrent client. Finding developers to write all kinds of apps in c++,
however, might be a little bit more difficult. So, we decided to build something
out with javascript. This presented the second problem. At the time, the
uTorrent client clocked in at about 300kb in total. Adding any kind of
javascript engine or browser to the client would increase the size to something
well over 75mb. As a large majority of bittorrent users were on slow internet
connections, we had data that showed a steep dropoff of user installation by
increasing the client even over 1mb.

To keep the client light and still provide a great app development experience,
we decided to embed Internet Explorer. Using W32 APIs and COM, we could package
javascript into "apps" that eventually became called Single Page Apps and
interact with the local client via an API that would do things like add/remove
torrents or show progress. I took care of everything on the javascript side of
this project from designing the API to creating the first applications and
building out the partner portal that allowed third parties to create
applications as well.

After launching all this, we learned a couple interesting things. Perhaps the
most surpising to me though was around Internet Explorer itself. While IE6 has
been long EOL'd and was rarely seen around the internet, our apps ended up
running inside an IE6 window almost 75% of the time. Come to find out, most of
the uTorrent users had pirated Windows and were therefore unable to upgrade
their base Windows XP version. This wasn't an issue for these users as they'd
just use Firefox or Chrome. Our use of the built-in windows APIs forced us to
use the default version of IE, causing some interesting problems when trying to
design and build out apps that work generally.

The more important learning was what eventually killed the project. BitTorrent
users don't want apps, they just want to download their torrents and be done
with the client. Even more critically, most vendors are uninterested in being
associated with the BitTorrent brand and were uninterested in having their
content distributed that way.

So, we decided to do something a little bit different! We had a great API and
could do some pretty fancy things with the web, maybe there was another
integration we could try? This eventually became a product called
[SoShare](https://www.bittorrent.com/blog/2013/02/15/soshare-public-beta-sending-unlimited/).
Instead of requiring everything to happen in a client experience, we created a
browser plugin. The plugin was just uTorrent and the API we'd designed for the
app platform. Pulling up a website was all that was required. Now, you could
package up large files and send them to less technical users by visiting a
website and starting the transfer. We'd keep everything running in the
background and do the incremental uploads that everyone loves with BitTorrent.
