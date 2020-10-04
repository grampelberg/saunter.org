---
title: Resume
draft: false
date: 2020-10-04
---

An engineer with experience taking goals and building products that can achieve
the desired outcomes, I am constantly asking "what problem are you solving?".
Facilitating a discussion around problems instead of implementation details
helps me [gather requirements](#requirements) that narrowly define what must be
done to either fix a user's problem or move the business in a specific
direction. Narrrow scope has benefits that cascade through the entire process,
making [architecture](#architecture) simpler, [implementation](#implementation)
quicker and [iteration](#iteration) more meaningful.

Instead of simply building products and then expecting users to come, I leverage
the lessons that come from building the product to
[launch](https://www.zdnet.com/article/mesosphere-launches-its-mesos-based-datacenter-os-plus-a-free-version-on-aws/)
solutions and educate users on why my solutions are the best for their problems.
The feedback from users gathered by supporting the product allows me to
iteratively build out the required functionality that
[grows the product](https://techcrunch.com/2016/04/19/mesosphere-open-sources-its-data-center-os/)
and solves even more problems for more users. I love listening to the
interesting ways diverse voices - such as sales, marketing or engineering use
the product. There's always a new use case or detail that was originally missed
which can be added to the product and make the experience even better!

Throughout my career my passion for building focused, highly technical products
has taken me on an exciting journey. I created [DC/OS](#dcos), one of the
earliest container orchestration solutions. Being part of the orchestration
space as it grew and matured, I was able to watch how developers had the
opportunity to write applications in a totally new way and build [ksync](#ksync)
to help improve their productivity. As more organizations adopted orchestration
platorms, it became apparent that there needed to be a new layer of abstraction
and so I authored the [Service Mesh Interface](#service-mesh-interface) which
provided observability, security and reliability to every application
automatically. With common tasks becoming more automated, operators had the time
to start looking at more advanced patterns for running their clusters, I used
this to focus on [multi-cluster](#multi-cluster) patterns and solutions that
allow for [global traffic management](/projects/#global-traffic-management).

This is a selection of some of the things I've worked on and shipped. For a more
complete list, either check out my [projects](/projects/) or
[LinkedIn](https://www.linkedin.com/in/grampelberg/).

## Buoyant

### Multi-Cluster

Kubernetes provides an abstraction layer that takes care of many tasks we used
to do all the time. I remember having to go find physical disk drives and plug
them into servers if I wanted more space! The abstractions that Kubernetes
provides has provided teams with more time to implement patterns that required
extremely large teams, even 5 years ago. Working with Linkerd, one of the most
common requests we receive is around multi-cluster support. Service meshes
shouldn't stop at the cluster boundary, they should make it easy for traffic to
communicate wherever a service is running.

<a name="requirements"></a>

Multi-cluster was one of the most popular requests by users, unfortunately the
details around what it should do and what problems were solved was a little
fuzzy. Most folks interested in operating multiple Kubernetes clusters were
looking for direction on how to get started with it. This presented quite the
challenge! The only way to design a solution that would work for most users
while maintaining Linkerd's focus on lightweight, simple implementations was to
really focus on gathering requirements from the community and iterating on
those.

After going through many iterations with users inside and outside of the Linkerd
community, I created a list of
[requirements](https://linkerd.io/2020/02/17/architecting-for-multicluster-kubernetes/).
By focusing on a solution that would work in most user's environments and fail
in a reliable manner, these requirements minimized the addition of moving parts
to an already complex system. Users end up with a pattern to implement their own
multi-cluster solutions. Making requirements explicit allowed us to understand
which specific architectures would work in the real world and actually solve the
problem users needed a solution for.

Taking the requirements and working extremely closely with Kubernetes
primitives, I was able to
[design a solution](https://linkerd.io/2020/02/25/multicluster-kubernetes-with-service-mirroring/)
that would work for most Linkerd users. By leveraging as many primitives as
possible, I was able to keep the implementation small and easy to reason about.

Check out the
[community discussion](https://docs.google.com/document/d/1uzD90l1BAX06za_yie8VroGcoCB8F2wCzN0SUeA3ucw/edit#heading=h.bw0xc1qalza2)
for a glimpse into how we pulled everything together.

### Service Mesh Interface

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

<a name="implementation"></a>

To validate the specification, I implemented the
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

## Vapor IO

### ksync

I like to pay attention to common tasks and think about ways to automate those,
so that I can write code more quickly. As I started building applications for
Kubernetes, I spent a lot of time building new containers, deploying them and
waiting for the containers to start up. Surely there is a better way to do this?

When doing development, it is rare for an entire container to need to be
rebuilt. If you're working with an interpreted language, like Python, there's no
need to even do a compile step. Locally, we just restart the server. For
compiled languages, subsequent compilation is actually pretty quick. Instead of
rebuilding containers from scratch every time, I decided to sync the files I was
working on locally to the remote container. This is all
[ksync](https://github.com/ksync/ksync) does at a high level. The details are
pretty interesting though! Check out
[TGIK8S](https://tanzu.vmware.com/developer/tv/tgik/0029/) to hear Joe Beda talk
about ksync and what problems it solves.

<a name="architecture"></a>

Syncing files into a container is not particularly hard, in fact `kubectl cp`
works as long as you can control the environment. For ksync to be generally
usable, more architecture was required. Two requirements scoped this problem and
introduced unique complexity:

- The target container cannot be assumed to have binaries residing in it. For a
  worst case scenario, containers should work with `scratch` images.
- Deployment manifiests cannot change. Adding or removing containers from a pod
  requires a restart and that adds maintenance overhead as well as complexity.

Given these constraints, you must first look at how to modify a container's
running filesystem from outside the container. By working outside the container
there are no assumed dependencies or binaries. Additionally, deployment
manifests won't have to change and it becomes possible to sync files
bidirectionally for totally arbitrary containers.

Working outside the container means that you need to directly modify the
container's FS from the host. OverlayFS, the default backend for Docker,
extracts a container's separate files onto the host's FS. It then joins the
different layers (base image vs working copy) together so that they look like
one FS to the running container. You can sync files directly into the working
copy's overlay and have them show up inside the running container. Using this
method, the container doesn't require any outside knowledge of updates.

Kubernetes provides all the tools required to get this up and running. Ksync
adds a daemonset to the cluster. This runs the agent (as a privileged container)
on every host. The agent connects directly to the Docker daemon and discovers
the path for a container's working copy. The agent also runs a bidirectional
syncing application that can move files from a developer's laptop into the
container itself. With hot reload, interpreted languages can have containers
fully updated and restarted in under a second. This is massive for developer's
productivity! Code updates can go from a process that takes minutes to seconds.

## Mesosphere

### DC/OS

Mesosphere started out with a solid foundation from Mesos. Early on we did some
experiments to learn what the best business model was. We added plugins for
Mesos, created some products to help with incident management in a scheduled
world and had some limited success with
[Elastic Mesos](https://d2iq.com/blog/mesosphere-for-digitalocean). None of
these were particularly sustainable business models for the company,
unfortunately.

<a href="#iteration"></a>

Taking some of the learnings from the [Mesos CLI](/projects/#mesos-cli), I
started to ask folks what makes a platform feel like an operating system. My
hypothesis coming into this was the GUI. Imagine my shock when the unanimous
answer was the [package management system](/projects/#package-management).
Installing complex applications is such a common task in modern operating
systems, it was the base requirement for day to day usage. Taking Mesos,
building a package management system for it and bundling the tools required to
actually run in production seemed like a good idea.

To test whether DC/OS was a good idea or not, I built the original version out
myself. This took the form of a React SPA running on top of a RoR app for all
the visualization. A [basic CLI](https://github.com/mesosphere-backup/dcos-cli)
provided write access to the Mesos API. I then automated the whole thing on top
of GCE by using
[startup scripts](https://cloud.google.com/compute/docs/startupscript). After
asking for a couple increases in quota, it was possible to create 1k+ node
clusters in under 5 minutes. Finally, there was no way I could create all the
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
