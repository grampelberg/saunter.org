---
title: Kubernetes line by line
draft: true
date: 2020-05-24
---

## Metadata

```yaml
metadata:
  name: frontend
```

All resources have a metadata section that includes labels and annotations.
Labels are supposed to be mostly static and used for filtering queries and
results. You'll see `selector` on many resources, that's what the controllers
use to figure out what resource they should be targeting.

Labels are simply key/value pairs. While it is tempting to use something
arbitrary, much of the ecosystem has standardized on a common naming scheme that
many would recognize:

```yaml
my.domain.com/name: "foobar"
```

In addition to the naming scheme, it is recommended to have a couple labels on
every resource. Take a look at the [common labels][labels] documentation for
more information there.

```yaml
app.kubernetes.io/name: frontend
app.kubernetes.io/instance: frontend-demo
app.kubernetes.io/version: v4
app.kubernetes.io/component: frontend
app.kubernetes.io/part-of: guestbook
app.kubernetes.io/managed-by: manual
```

It might feel like you're duplicating data, such as the name, but it is worth
it! These all allow you to query the API. Instead of
`kubectl get deployment frontend`, you can do something like
`kubectl get deployment -l app.kubernetes.io/name=frontend`. Each label is
especially useful when using a templating system such as [helm][helm] or
[kustomize][kustomize]. Templated with helm, these labels would look like:

```yaml
app.kubernetes.io/name: "{{ .Chart.Name }}"
app.kubernetes.io/instance: "{{ .Release.Name }}"
app.kubernetes.io/version: "{{ .Chart.AppVersion }}"
app.kubernetes.io/component: frontend
app.kubernetes.io/part-of: guestbook
app.kubernetes.io/managed-by: "{{ .Release.Service }}"
```

Annotations are slightly different than labels. They're a way to add arbitrary
data to resources. It isn't possible to filter on annotations, but you can put
anything in there you'd like. Imagine anything from timestamps to nested YAML!
When trying to decide if something should be a label or an annotation, consider
if you want to query on it or whether it will change. As they're meant to be
queried, labels should be mostly static.

So, what is a good use of annotations? How about:

- Runbooks - ever want to know what to do with a service in the middle of an
  incident? By adding your runbook as an annotation in markdown format, it will
  always be attached to the resource that needs to be debugged. This ends up
  treating the running state of Kubernetes as a single source of truth! Everyone
  interacting with the system can just look at what's defined in Kubernetes to
  understand the next steps. After all, [Kubernetes is a database][a-database].
- Version information - wouldn't it be nice to know what actual code is running?
  Put a commit hash in as an annotation, then you can just link back to exactly
  what is running to see what problems are happening.
- Template configuration - instead of having to guess what configuration was
  used by the templating system (like helm), stick it into the objects as
  annotations. Debugging why a specific configuration is running becomes
  infinitely easier.
- User information - ever wondered who updated something last? Put their
  username in as an annotation!
- Modeling dependencies - by default, changes to `ConfigMap` resources does not
  trigger a redeployment. If you're using environment variables or need your app
  to restart when a `ConfigMap` changes, you'll want to do something like add an
  annotation that is the SHA of the dependent `ConfigMap`. That way, when the
  config updates, the deployment will redeploy.

Many annotations end up being multiline. There are bunch of [different
ways][yaml-multiline] to do multiline YAML, but here's a simple example to get
started with:

```yaml
metadata:
  annotations:
    guestbook.com/runbook: |
      # Guestbook

      This is the frontend of an app that consists of many components. See
      the full repo for more information.

      # Who do you call?

      [Ghostbusters](https://en.wikipedia.org/wiki/Ghostbusters)
```

Remember, this is just a blob of text! You can put anything in there such as
JSON, YAML or as in this example, markdown itself.

A completely populated `metadata` section for this app would end up looking
something like this:

```yaml
metadata:
  name: frontend
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/instance: frontend-demo
    app.kubernetes.io/version: v4
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: guestbook
    app.kubernetes.io/managed-by: manual
  annotations:
    guestbook.com/user: thomas
    guestbook.com/commit: 0112841
    guestbook.com/repo: https://github.com/kubernetes/examples
    guestbook.com/branch: master
    guestbook.com/runbook: |
      # Guestbook

      This is the frontend of an app that consists of many components. See
      the full repo for more information.

      # Who do you call?

      [Ghostbusters](https://en.wikipedia.org/wiki/Ghostbusters)
```

## Selector

```yaml
spec:
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
```

The `spec` here tells a deployment controller what it should do. This controller
uses the selector to discover what it is managing and what needs to happen next.
While it is normally a best practice to simply match the labels between the
`template` and `matchLabels`, the selector here is actually [super
flexible][selector].

In this example, the only real thing to change would be to update the label
names to the recommended format like:

```yaml
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: frontend
      app.kubernetes.io/instance: frontend-demo
      app.kubernetes.io/component: frontend
      app.kubernetes.io/part-of: guestbook
      app.kubernetes.io/managed-by: manual
```

You'll notice that `version` is not part of these. The selector must match both
the old version and new version during updates. This requirement means that
you'll not actually want to select _all_ labels.

Something fun to do is change the labels on a specific pod's definition. If the
pod's labels are changed and no longer match, the deployment controller will
spin up a new pod and leave the one with changed labels alone. This allows you
to remove buggy pods from a service and do debugging without impacting
production traffic. It also allows you to create test pods manually and add them
to a service/deployment while testing. There's a way to do things incrementally
instead of requiring an entire deployment's rolling update!

## Placement

This example does not specify placement. While that's likely fine for
development or a demo environment, moving into production should have a little
bit of extra configuration. The Kubernetes scheduler is a little naive. It will
look for nodes that can fit a pod and prioritize nodes that have already pulled
the required image. If you do not use [anit-affinity][anti-affinity], it is
likely that your pods will bunch up on a single node. Adding an anti-affinity to
the pod's `spec` will end up distributing your workloads across the cluster
unless it doesn't fit otherwise. This will increase your resilience to node
failures and theoretically improve uptime. To add this kind of distribution in,
you can add:

```yaml
spec:
  template:
    spec:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/name
                    operator: In
                    values:
                      - frontend
              topologyKey: kubernetes.io/hostname
        weight: 100
```

Using `preferredDuringSchedulingIgnoredDuringExecution` tells the scheduler to
try and follow this policy, but if it isn't possible to schedule the pod
anyways. If you were interested in requiring placement, you'd use
`requiredDuringSchedulingIgnoredDuringExecution` instead. The
`IgnoredDuringExecution` part of this is important. Once your pod has been
placed, the cluster won't optimize by moving things around.

```yaml
topologyKey: kubernetes.io/hostname
```

`topologyKey` tells Kubernetes to look for unique node hostnames. There is quite
a list of options here, such as `kubernetes.io/os` for workloads running in
clusters running windows as well as linux. You could also use
`failure-domain.beta.kubernetes.io/zone` to optimistically schedule pods across
zones. The [full list][topologykey] can give you some ideas of ways to
distribute pods across zones, architectures or arbitrary keys.

```yaml
labelSelector:
  matchExpressions:
    - key: app.kubernetes.io/name
      operator: In
      values:
        - frontend
```

By matching the `app.kubernetes.io/name` label, the specified weight of 100 will
be applied to scheduling decisions if a pod is already running on a node. This
is flexible and you could make it select across a whole application
(`guestbook`) either co-scheduling resources that work in concert or keeping
hungry pods separate for performance reasons.

Note: if you're running 1.18, there is a new field
[`topologySpreadConstraints`][spread-constraints] that provides more
configuration around this type of placement policy. In fact, if you enable the
alpha flag, it is possible to set this at the [cluster level][cluster-spread]!

## Updates

At its core, Kubernetes is a system that takes specs and tries to make the
running system match this. Controllers notice a change that happened, look at
the current state of the system and plan how to get to the desired state. This
has a couple implications that are important to pay attention to:

- Running resources won't change unless the spec changes. If you're using
  `latest` as an image tag and run `kubectl apply -f my-deployment.yaml`,
  nothing will happen! The spec hasn't changed, so the deployment controller
  doesn't know that it needs to do something.
- Changing top level metadata on resources doesn't do anything. Adding a label
  or removing an annotation from a deployment does not trigger any other
  changes. As there is no concrete resource specified by this metadata, there is
  nothing for Kubernetes to update once the resource has been saved.
- Changing metadata for concrete resources triggers updates. Imagine leaving
  everything in a pod's spec identical and just changing an annotation. Because
  the spec has changed, a rolling deployment will be kicked off. Sometimes this
  is desired behavior, such as linking a `ConfigMap` having an update with a
  deployment. Other times, when an annotation is time specific, this can cause
  some downtime and unnecesary churn on the cluster.

When changes do occur, by default, Kubernetes will use the `RollingUpdate`
strategy. The YAML below simply enumerates the default settings, so you don't
need to include it in the spec unless you'd like to make it explicit.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

This configuration will kick off a rolling deployment. There will be a new
`ReplicaSet` with its replicas set to 25% of the number configured. This comes
from `maxSurge` and can either be a percentage or a number. The old `ReplicaSet`
will be simultaneously scalled down by 25%. This comes from `maxUnavailable` and
can have the same values as `maxSurge`. As the new pods startup and become
ready, the deployment controller will scale the old `ReplicaSet` down until it
is at zero replicas.

If you have modified the affinity/anti-affinity configuration or have specific
uptime requirements, the `maxSurge` and `maxUnavailable` values are what you'll
want to change. The [documentation][rollingupdate] has more details.

Note that without some extra work, this strategy will not result in zero
downtime deploys. For most situations, this is fine. If you are concerned about
having 100% zero downtime deploys, you'll want to dig in some more.

There is pretty much no reason to change the update strategy type for a
deployment. While it is possible to do other update strategies such as
blue/green or canary, they are not built into vanilla Kubernetes and require
extra tools such as [flagger][flagger] that automate the other updates
strategies for you.

## Pod Metdata

There are two levels of metadata on a deployment resource. One is the top level
metadata, used by the deployment itself. The other is the metadata associated
with a template, used to create the pod. The pod's metadata has slightly
different requirements:

- Don't add `name` or `namespace`. These will be inherited from the parent. Any
  value you add here will be accepted but not actually do anything. A pod's name
  is based off the deployment's name (`frontend`), the hash of the pod's
  template and and a random string for the pod specifically. If you take a look
  at the created pod, you'll see that the name is being set by `generateName`
  which is a metadata field that anyone can use!
- Match the selector as closely as possible. There should be as much overlap
  between the selector and the pod's labels as possible.
- Don't use label values that can change. Add these to annotations or don't add
  the pod labels with these values to the selector. You can't change the
  selector after creation, so having values between metadata and the selector
  that change will cause update issues.

## RBAC

Kubernetes [`ServiceAccount`][serviceaccount] resources are credentials that can
be combined with `Role` and `RoleBinding` resorces to grant access to the API.
These do not restrict access to the pod itself, there are other resources such
as `NetworkPolicy` to do that. Even if a pod does not need to utilize the API at
all, it is a best practice to specify a unique service account for each
deployment. This guarantees that if there is a misconfigured `RoleBinding`, a
pod does not have accidental access to the API's data. When unspecified (as in
this example), the `default` service account is used from the namespace the pod
is running in. Make sure you don't modify the default `RoleBinding`!

We can specify the service account by adding:

```yaml
spec:
  template:
    spec:
      serviceAccountName: frontend
```

This will use the service account in the same namespace. A token is
automatically mounted and running `kubectl` from inside a pod will automatically
use the correct credentials, no other configuration required! This is
particularly useful when combined with an `initContainer` to wait for dependent
resources to becomes healthy before starting.

## Service Links

Replicating a feature that was originally part of Docker, Kubernetes adds
environment variables to each running container for every service in the system.
While this sounds useful, it is actually harmful. It is not possible to update
environment variables after a container has started. This has a couple important
implications:

- Services either added or removed will not be reflected by the environment
  variables. The container will have no way to know changes to the system and
  either miss services added after startup or try to send traffic to
  non-existent services.
- Changes to the service are not reflected. If an IP address changes for the
  service, the environment variable will not update. Even though the service
  still exists, the container will be sending requests to the wrong location.
  This is particularly subtle because you could have some pods working just fine
  while others are unable to address the right service.

As this feature shouldn't be used, it is worth disabling entirely:

```yaml
spec:
  template:
    spec:
      enableServiceLinks: false
```

## Security Context

A pod's security context configures what it can and cannot do. This can set
defaults for the containers in the pod and can be overridden by the individual
containers. [`PodSecurityPolicy`][psp] can be used to restrict what users
configure for individual pods. Allowing pods to run as root is a [bad
idea][just-say-no]. While you can (and should) set defaults for this as
`PodSecurityPolicy`, we can set some defaults for the pod in this example.

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 99
        runAsGroup: 99
```

With `runAsNonRoot` set, containers will fail to start if they're using the
default (`root` for docker containers). As it is pretty rare for containers to
specify a user and group to run as `runAsUser` and `runAsGroup` use the standard
99 for [nobody][nobody] to keep access to a minimum inside each container.
Remember, these can be overridden on a per-container basis! If you have a
container that must run as root or already specifies the user/group to run as,
you can manage it at that level.

## Advanced Concepts

So far, the focus has been primarily on best practices. There's a lot of really
interesting configuration that you can provide to a pod though! To keep
optimizing your pods, check out:

- `affinity` can be used for quite a bit more than simply spreading pods out
  across nodes. It is definitely worth [reading up][affinity] to get an idea of
  everything that is possible.
- `ephemeralContainers` can be added after a pod has started! If you've wondered
  how you can debug running containers that don't have any debugging tools,
  [ephemeral containers][ephemeral] are what you've been looking for! You can
  package up a debugging container with all your favorite tools and runtimes.
  Modify the pod you're interested in and voila, instant debugging environment.
- `priorityClassName` works hand in hand with cluster autoscaling. The priority
  provides hints to the scheduler so that it can start important pods first or
  preempt less important workloads. This is _not_ guaranteed and should not be
  used to order dependencies for an application. Instead, check out the [waiting
  pattern][waiting-pattern] to make sure applications start in the correct
  order.
- `shareProcessNamespace` allows every container in a pod to see what processes
  are running in the other containers. This is fantastic for debugging and you
  are even able to see the filesystem in all other containers because it is
  mounted as part of `/proc`. There are some obvious security issues with this,
  so it is more of a tool to use when required than something to use as a best
  practice. The [documentation][shared-process] has more details.
- [`tolerations`][taints] allow a pod to run on nodes it would not be able to
  run on otherwise. Taints added to nodes combine with tolerations on pods to
  tell the scheduler where it can put a specific pod. While using taints and
  tolerations could be used to create [pets][pets-vs-cattle] instead of cattle
  in a cluster, they come in handy!

## Command and Arguments

The interaction between `command`, `args` and Docker's
[`ENTRYPOINT`][entrypoint] can be pretty confusing. Keep a couple things in mind
and you'll be just fine! First off, `ENTRYPOINT` is the same as `command` in
Kubernetes. If you want to override `ENTRYPOINT`, use `command`.

`command` and `args` do not run in a shell by default. This improves security
but comes with a couple downsides. Shells [split words][word-splitting] by
default, taking a command like `ls -la` and converting it into `["ls", "-la"]`
when a process is executed. Instead, it is up to you to pass the arguments in
correctly. Don't fret, there's a simple one-liner that can show you what to add
in, just run:

```bash
echo "my-command --args foobar" | \
  awk '{for(i=1;i<=NF;i++){printf "\"%s\", ", $i}; printf "\n"}'
```

Additionally, there is nothing to expand environment variables. If you're
running a command like `echo $FOOBAR`, your process will receive the string
`$FOOBAR` instead of whatever that variable expands to. Ideally, you'd have your
process read the environment variable directly. Sometimes that isn't possible,
but you can set `command` and `args` to actually run a shell so that none of
this is nearly as important.

```yaml
spec:
  template:
    spec:
      containers:
        - name: php-redis
          command:
            - /bin/sh
            - -c
          args: |
            echo 'do whatever you want!' && my-process
```

`command` runs the `sh` shell and says that anything in `args` is a string that
should be interpreted as a script. With multiline YAML, args can look exactly
like the script you'd normally run.

## Container Security Context

In addition to `securityContext` at the pod level, it is possible to configure
this at the container level. In fact, there are a couple important pieces of
configuration at this level. As with `securityContext` at the pod level, it is a
best practice to define [`PodSecurityPolicy`][psp] and use those for defaults
instead of specifying it for every container and pod.

```yaml
spec:
  template:
    spec:
      containers:
        - name: php-redis
          securityContext:
            capabilities:
              drop:
                - all
            privileged: false
```

When `privileged` is `true`, a container gets full access to all the devices on
a host. A container with this kind of access could modify system configuration
or manipulate other processes running on the node, even the kubelet itself! If a
container needs to run as `privileged`, it should be treated very specially.

Linux has a list of [capabilities][capabilities] that a process can get. These
range from `CAP_SYS_ADMIN` allowing a container to mount and unmount volumes on
the host to `CAP_NET_ADMIN` allowing a container to interact with the host's
network in any way it wants. None of these are required for most workloads and
should be dropped by default. For those that do need added capabilities, it is
still valuable to drop everything first. That way, the containers run with only
what they absolutely require.

## Images

Let's get something out of the way before diving into the example - friends
don't let friends use `:latest`. It is tempting, but breaks quite a few
assumptions that Kubernetes makes:

- Kubernetes does not watch your image repository. Pushing new versions to a tag
  like `latest` will not cause updates to occur. While there is tooling in the
  ecosystem to help out here, it is definitely an anti-pattern. Isn't it nice to
  have everything explicit anyways?
- As covered in the [updates](#updates) section, applying a YAML that doesn't
  change also won't update anything. It is possible to add an annotation that
  changes, but now you're changing something! Might as well be explicit about
  what image you're running.
- By default, images are only pulled if they do not already exist on the node.
  This works great as it ends up caching your image on the nodes that it runs
  on. Any subsequent starts of the container will go faster and use less
  bandwidth. You can have the image pulled every time a container starts, but
  now you're slowing the pod's launch down, using up extra bandwidth, adding
  load to the image registry that isn't strictly required and introducing a
  single point of failure in your infrastructure.

It is a pain to modify image tags that might be spread between multiple files
every time you want to do an update. Luckily, there's a lot of great tooling
that has been created in the ecosystem to help out! Kustomize comes with
[transformers][kustomize-image] that allow you to specify the tag for any
container in a project. You can also use [helm][helm] or [jkcfg][jkcfg]. It
might seem like extra effort to start using these tools, but it will pay you
back in the future. Eventually, you're going to want to be able to configure
your application to run in different environments such as dev, staging or prod.
These tools help out with more than just image tags, they can configure
replicas, secrets and resources so that your applications are setup correctly to
run in each environment.

```yaml
image: gcr.io/google-samples/gb-frontend:v4
```

This is a pretty decent image. There's a version (`v4`) and the image is fully
qualified. By adding the commit SHA that this image was built from to the tag,
you will always be able to understand exactly what version of something is
running and then debug from there. It would be possible to use [semver][semver]
to solve the same problem. Simply increment the minor version every time a build
happens. You would then add a tag in git for every minor version bump. As this
tends to just create a ton of tags and get in the way of the information you're
actually looking for, the commit itself, it isn't usually valuable to use full
semver.

## Resources

```yaml
resources:
  requests:
    cpu: 100m
    memory: 100Mi
```

There are two configurations possible for resources - requests and limits. You
can think of requests as guaranteeing _at least_ a certian amount of resources
are available for your container. Limits define _at most_ the resources that
your container can consume.

As requests are related to the minimum amount of guaranteed resources, they are
used by the scheduler to fit your workload into existing nodes. In this
instance, 10% of a CPU has been requested. The `100m` value here refers to
millis and 1000m make up a single CPU. Check out the [resource
documentation][cpu-millis] for more units, there's a bunch of them for memory!

When scheduling, this container will only be run on a node that has at least 10%
of a CPU available. It is pretty important to remember that resource requests
are static and not related to usage. They're checked at scheduling and increase
or decrease the amount of resources available on a node when the container is
originally started. If your container doesn't use the full request, that won't
change anything.

Linux provides a little bit more to guarantee that your container has at least
the resources that you requested. This is done by setting [cpu
shares][cpu-shares]. While it is entirely possible for your container to consume
more than its request, the kernel will guarantee that the CPU is there when it
is needed. If you're not using your full request, any container on the node can
optimistically use the CPU request until it is required by the original
container again.

It is critical that every container definition have resource requests. Without
requests the scheduler will continue to add pods to one node, effectively
over-provisioning it. If anything on that node was to start consuming more CPU
than it should, this noisy neighbor could take everything else on the node down.

If that isn't scary enough for you, here's another good reason - autoscaling.
[Cluster autoscaler][cluster-autoscaler] watches for pods that are unschedulable
for resource reasons. When a pod is outstanding and cannot fit on the cluter,
the autoscaler will create a new node. Your cluster will then automatically be
able to grow or shrink based on what resource requests you've configured and how
much has been scheduled on the cluster. Remember, resource requests are static
and not related to usage, if you'd like to do autoscaling based off current
utilization check out the [horizontal pod autoscaler][hpa].

Limits are the opposite of requests. Instead of guaranteeing a minimum amount of
resources, they define the maximum amount that a pod can use. Setting a limit
for CPU will guarantee that a pod cannot consume any more than that on a node.
Alternatively, setting a limit for memory will cause your container to be OOM
killed. As it is pretty valuable to let CPU burst when the node has some
available, setting the CPU limit on a container should only happen when you've
got a specific requirement for it.

Memory limits are quite a bit more important. When nodes run out of memory, they
crash. Imagine having a container with a slow memory leak being able to take
everything down on a node. That doesn't sound like a fun incident to explain.
The value can be set pretty high and provides a little bit of insurance in case
there is a runaway memory issue.

```yaml
resources:
  requests:
    cpu: 100m
    memory: 100Mi
  limits:
    memory: 1Gi
```

With requests and limits configured this way, containers will have a minimum
amount of resources they can consume, noisy neighbors will have less opportunity
to slow sibling workloads down and runaway processes won't be able to crash the
node.

Don't worry if you don't have any idea where to set these right now. It is easy
to pick somewhat arbitrary numbers, monitor over time and iterate on the values.
After letting a workload run for awhile, take a look at the resource consumption
and make a more informed decision. Having something for requests and limits that
is too large or small is far better than not having anything at all.

## Ports

```yaml
ports:
  - containerPort: 80
```

Did you know that the ports section doesn't actually really do anything? Each
pod gets its own [network namespace][network-namespace]. The containers inside
this pod are responsible to open the ports that they listen on as well as keep
from stepping on the toes of other containers. Container port definitions are
primarily documentation for what your container is up to.

Even though the system doesn't actually do anything with container ports, the
documentation is indispensible! All you need to do is look at a pod definition
and immediately have an idea about what's going on. In fact, there should be a
name as well.

```yaml
ports:
  - containerPort: 8080
    name: http
```

Note, the container port was updated from 80 to 8080 as well. This is so that
the container does not need to run as root. If you want the container to be
accessible on port 80, simply define it that way in the `Service` and map from
80 to 8080.

## Probes

This YAML is completely missing `readinessProbe`. That's a critical mistake! The
`readinessProbe` is used to inform Kubernetes as to whether a container can
receive traffic or not. These are run constantly and not just at startup and
provide you with the tools to manage a couple important scenarios related to
lost requests and capacity.

First, it is important to understand how the pieces fit together. To reach a
pod, you must define a service. By default, services create a virtual IP address
in your cluster. When a node sees traffic destined to this virtual IP address,
it uses [iptables][iptables] redirect rules to convert the virtual IP address
into a randomly selected pod IP address. These pod IP addresses are represented
in Kubernetes as `Endpoint` resources. An `Endpoint` has an IP address added to
it when a pod passes its readinessProbe. Effectively, this means that a pod
cannot receive traffic until it passes its readinessProbe. Any time a
readinessProbe fails, the IP address will be removed from the endpoint and the
pod will no longer receive traffic.

If a `readinessProbe` is left off, the IP address for a pod is immediately added
to the endpoint. At this point, it becomes a race to see whether your containers
have started up or traffic falls on dead ears. For particularly fast servers, it
is possible that the process will have started up in time. Everything else will
cause a couple issues ranging from connection errors to the dreaded 503 error
from ingress controllers. Most processes take a little time to start up and
therefore setting a readinessProbe can eliminate what seems like flaky behavior
in a cluster.

Readiness is a tool that can be used all the time instead of just at startup.
Imagine an application that has a max queue size. If the readinessProbe would
fail when the max queue size is reached, that pod will have its IP address
removed from the endpoint. Traffic will cease to be distributed to the pod with
a failed readinessProbe allowing the pod to work through its backlog. When the
queue has shrunk in size, the readinessProbe will pass again and traffic will
flow again. As readinessProbes are not used to signal restarting processes, this
is completely safe to do! If you'd like to actually restart the container,
`livenessProbe` is the right tool for the job.

Note that because endpoint IP addresses need to be added on every node, and
iptables is extremely slow. Depending on the number of updates happening on your
cluster there are no guarantees as to how immediately an endpoint is added or
removed. There have been some reports on busy clusters of it taking over a
minute to distribute changes to endpoints. This is one of the reasons why the
`preStop` and `terminationGracePeriodSeconds` settings may need to be configured
on your cluster to ensure no requests are lost.

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 8080
```

In this example, there's basically no reason to leave a readinessProbe off.
There's already an HTTP server listening and there's no special requirements for
readiness! All it needs is a 200 response.

The `livenessProbe` has been left off here on purpose. Liveness is related to
restarting the process and comes into play when there's an opportunity for a
process to get locked up. For some applications this is a big risk and the
livenessProbe should be configured. Most applications, however, tend to not have
explicit deadlock issues and you don't get much from defining liveness.

## Advanced Concepts

- `ConfigMap` updates
- `preStop` to manage graceful shutdown

## Raw

```yaml
apiVersion: apps/v1 #  for k8s versions before 1.9.0 use apps/v1beta2  and before 1.8.0 use extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
  replicas: 3
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
        - name: php-redis
          image: gcr.io/google-samples/gb-frontend:v4
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
          env:
            - name: GET_HOSTS_FROM
              value: dns
          ports:
            - containerPort: 80
```

[labels]:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/
[helm]: https://helm.sh/
[kustomize]: https://kustomize.io/
[a-database]: TODO
[yaml-multiline]: https://yaml-multiline.info/
[selector]:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
[anti-affinity]:
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity
[topologykey]:
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#built-in-node-labels
[rollingupdate]:
  https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment
[flagger]: https://flagger.app/
[serviceaccount]:
  https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
[spread-constraints]:
  https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/
[cluster-spread]:
  https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/#cluster-level-default-constraints
[psp]: https://kubernetes.io/docs/concepts/policy/pod-security-policy/
[just-say-no]: https://opensource.com/article/18/3/just-say-no-root-containers
[nobody]: https://en.wikipedia.org/wiki/Nobody_(username)
[affinity]:
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity
[ephemeral]:
  https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/
[waiting-pattern]: TODO
[shared-namespace]:
  https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/
[taints]:
  https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
[pets-vs-cattle]:
  https://thenewstack.io/how-to-treat-your-kubernetes-clusters-like-cattle-not-pets/
[capabilities]: http://man7.org/linux/man-pages/man7/capabilities.7.html
[entrypoint]: https://docs.docker.com/engine/reference/builder/#entrypoint
[word-splitting]:
  https://www.gnu.org/software/bash/manual/html_node/Word-Splitting.html
[kustomize-image]:
  https://github.com/kubernetes-sigs/kustomize/blob/master/docs/fields.md#transformers
[jkcfg]: https://github.com/jkcfg/jk
[helm]: https://helm.sh
[semver]: https://semver.org/
[cpu-millis]:
  https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-units-in-kubernetes
[cpu-shares]:
  https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-cpu
[cluster-autoscaler]:
  https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler
[hpa]:
  https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
[network-namespace]:
  https://kubernetes.io/docs/concepts/cluster-administration/networking/#the-kubernetes-network-model
[iptables]: TODO
