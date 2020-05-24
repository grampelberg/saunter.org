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
clusters running windows as well as linux. The [full list][topologykey] can give
you some ideas of ways to distribute pods across zones, architectures or
arbitrary keys.

```yaml
labelSelector:
  matchExpressions:
    - key: app.kubernetes.io/name
      operator: In
      values:
        - frontend
```

By matching the name we're running, the weight of 100 will be applied to
scheduling decisions if a pod is already running on a node. This is flexible and
you could make it select across a whole application (`guestbook`) either
co-scheduling resources that work in concert or keeping hungry pods separate for
performance reasons.

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
    type: RollingUpdatea;sldfk
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

- Don't add `name` or `namespace`. These will be inherited from the parent.
- Match the selector as closely as possible. There should be as much overlap
  between the selector and the pod's labels as possible.
- Don't use label values that can change. Add these to annotations instead. You
  can't change the selector after creation, so having values between metadata
  and the selector that change will cause update issues.

## RBAC

Kubernetes [`ServiceAccount`][serviceaccount] resources are credentials that can
be combined with `Role` and `RoleBinding` resorces to grant access to the API.
These do not restrict access to the pod itself, there are other resources such
as `NetworkPolicy` to do that. Even if a pod does not need to utilize the API at
all, it is a best practice to specify a unique service account for each
deployment. This guarantees that if there is a misconfigured `RoleBinding`, a
pod does not have access it should not to the API's data. The default is to use
the `default` service account in the namespace that they pod is running in. Make
sure you don't modify the default `RoleBinding`!

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

## Priority

TODO

## Security Context

TODO

##

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
              # If your cluster config does not include a dns service, then to
              # instead access environment variables to find service host
              # info, comment out the 'value: dns' line above, and uncomment the
              # line below:
              # value: env
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
