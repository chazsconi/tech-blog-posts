---
layout: post
title:  "Creating an Erlang/Elixir cluster on Kubernetes"
date:   2019-06-08 12:00:00
published: false
---

# Deployment without an Erlang cluster

Running an Elixir application on Kubernetes is reasonably simple by using the
[Distillery](https://github.com/bitwalker/distillery) hex package and the instructions
included on [how to build a Docker image](https://hexdocs.pm/distillery/guides/working_with_docker.html).

This can then be deployed to Kubernetes using a Deployment and Service as follows:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-example-deployment
  labels:
    app: k8s-example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-example
  template:
    metadata:
      labels:
        app: k8s-example
    spec:
      containers:
      - name: k8s-example
        image: chazsconi/k8s-example
        ports:
        - containerPort: 4000
        imagePullPolicy: Always
      restartPolicy: Always
---
kind: Service
apiVersion: v1
metadata:
  name: k8s-example-service
spec:
  type: NodePort
  selector:
    app: k8s-example
  ports:
  - protocol: TCP
    port: 80
    targetPort: 4000
    # Specify a node port so we can expose this simply.  Alternatively use an Ingress
    nodePort: 30000
```

## Scaling up

By changing the number of replicas, the app can be scaled e.g.
```
kubectl scale deployment k8s-example-deployment --replicas=3
```
Doing this you will have one Erlang node on each pod, but they will not be able to communicate with each other.

For a simple Phoenix app without Phoenix channels this will suffice.  However, if you need to use Phoenix Channels,
have singleton instances of `GenServer` or any other functionality that requires inter-node connectivity
you will need an Erlang cluster.

# Creating an Erlang Cluster

To create an Erlang cluster three things are required:

1. Each node must have a name e.g. `node@my-node.my-domain`

  Here `my-node.my-name` will be the name of the pod. `node` will not change unless you plan
  to run multiple applications within the same pod in different containers (out of the scope of this post).

  This is set in `vm.args`

2. Each node must have the same Erlang cookie.

  This is set in `vm.args` also.

3. The nodes must be able to discover, resolve and communicate with each other

  The various options for this will now be discussed.

## Solution using StatefulSets

Using Deployments, your pods will be allocated generated names, which makes discovery harder.  However
by using a StatefulSet, your pods will be allocated names in sequential order, e.g. `myapp-0`, `myapp-1` etc
so you will know all their names in advance, so long as you keep the scale of your stateful set the same.

To do this, firstly you need to set the node name in `vm.args` based on the fully qualified domain name of the pod.
This can be done by setting an environment variable in the `pre_configure` hook of distillery.  This hook is executed
before the app starts.

First configure Distillery:
```elixir
# /rel/config.exs
environment :prod do
  ...
  set(pre_configure_hooks: "rel/hooks/pre_configure")
  ...
end
```

...and then you can create the hook:
```shell
# /rel/hooks/pre_configure/set_erlang_name.sh
export ERLANG_NAME=$(hostname -f)
```
...and finally you can use this in `vm.args`
```
# /rel/vm.args
...
-name node@${ERLANG_NAME}
```


The StatefulSet is then created as follows:

```yaml
---
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: k8s-example-statefulset
spec:
  selector:
    matchLabels:
      app: k8s-example
  serviceName: k8s-example-erlang-cluster
  podManagementPolicy: Parallel
  replicas: 3
  template:
    metadata:
      labels:
        app: k8s-example
    spec:
      containers:
      - name: k8s-example
        image: chazsconi/k8s-example:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 4000
        imagePullPolicy: Always
```

A service is also needed to create the DNS records for the pods backing the stateful set.
This will cause each pod to have the name: `k8s-example-N.k8s-example-erlang-cluster.default.svc.cluster.local`
where `N` is the instance number, `0`, `1`, `2` etc, and `default` is the namespace name:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: k8s-example-erlang-cluster
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  # The port is not used by the service but it is needed to create SRV records for the statefulset
  - port: 80
  clusterIP: None
  # Do not wait for the ready check
  publishNotReadyAddresses: true
  selector:
    app: k8s-example
```

The simplest solution to allow the nodes to connect is to hardcode the list of nodes within the application, rather than use
a discovery mechanism. This can be done with the [libcluster](https://github.com/bitwalker/libcluster) hex package:

```elixir
defmodule MyApp.App do
  use Application

  def start(_type, _args) do
    topologies = [
      example: [
        strategy: Cluster.Strategy.Epmd,
        config: [
          hosts: [
            :"node@k8s-example-statefulset-0.k8s-example-erlang-cluster.default.svc.cluster.local",
            :"node@k8s-example-statefulset-1.k8s-example-erlang-cluster.default.svc.cluster.local",
            :"node@k8s-example-statefulset-2.k8s-example-erlang-cluster.default.svc.cluster.local"
          ]
        ]
      ]
    ]
    children = [
      {Cluster.Supervisor, [topologies, [name: MyApp.ClusterSupervisor]]},
      # ..other children..
    ]
    Supervisor.start_link(children, strategy: :one_for_one, name: MyApp.Supervisor)
  end
end
```
The `libcluster` library has various strategies, however the one used here, `Cluster.Strategy.Epmd`, only connects on startup
and not periodically, so any connection problem will not allow nodes automatically reconnect.  Also, as the node names are hardcoded,
scaling the number of pods with `kubectl scale` will not work.

## Adding discovery to StatefulSets

When creating a k8s service for Statefulsets, in addition to `A` DNS records being created for pods, `SRV` records are also created
if there is at least one port specified for the service.  In the example above, port 80 is specified.

You can see this by executing a dns query from within one of the pods:
```console
$ kubectl exec -it k8s-example-statefulset-0 bash
bash-4.4# nslookup -q=srv k8s-example-erlang-cluster.default.svc.cluster.local
Server:   10.96.0.10
Address:  10.96.0.10#53

k8s-example-erlang-cluster.default.svc.cluster.local	service = 0 33 80 k8s-example-statefulset-0.k8s-example-erlang-cluster.default.svc.cluster.local.
k8s-example-erlang-cluster.default.svc.cluster.local	service = 0 33 80 k8s-example-statefulset-1.k8s-example-erlang-cluster.default.svc.cluster.local.
k8s-example-erlang-cluster.default.svc.cluster.local	service = 0 33 80 k8s-example-statefulset-2.k8s-example-erlang-cluster.default.svc.cluster.local.
```

By using the `libcluster` strategy `Cluster.Strategy.Kubernetes.DNSSRV` you can auto-discover the pods in the stateful set using DNS.

```elixir
topologies =
  [
    example: [
      strategy: Cluster.Strategy.Kubernetes.DNSSRV,
      config: [
        service: "k8s-example-erlang-cluster",
        namespace: "default",
        application_name: "node",
        polling_interval: 10_000
      ]
    ]
  ]
```

To avoid hardcoding the service and namespace you can pass in environment variables from the service by setting this in the `env` section of the statefulset pod spec
and then reference them with `System.get_env/1`:

```yaml
env:
- name: NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
- name: ERLANG_CLUSTER_SERVICE_NAME
  value: k8s-example-erlang-cluster
```

## Solution using Deployments

Although a Statefulset is the simplest solution to having an Erlang cluster, using a Deployment can also have advantages.  For example
when you have limited resources in your cluster and you only want one instance of the pod running normally - during a redeployment the pods
will temporarily scale up to two, and then back down again to one.  With a StatefulSet, you need to have at least two pods running to ensure
no downtime during deploys as each pod is stopped and started again in rolling restart.

With a few steps you can also set up a Kubernetes Deployment to have discoverable nodes.

Firstly having a service (in the same way for Statefulsets) a SRV DNS record will be created.

```console
bash-4.4# nslookup -q=srv k8s-example-erlang-cluster.default.svc.cluster.local
Server:		10.96.0.10
Address:	10.96.0.10#53

k8s-example-erlang-cluster.default.svc.cluster.local	service = 0 33 80 10-244-0-123.k8s-example-erlang-cluster.default.svc.cluster.local.
k8s-example-erlang-cluster.default.svc.cluster.local	service = 0 33 80 10-244-0-124.k8s-example-erlang-cluster.default.svc.cluster.local.
k8s-example-erlang-cluster.default.svc.cluster.local	service = 0 33 80 10-244-0-125.k8s-example-erlang-cluster.default.svc.cluster.local.
```

As can be seen, each pod is given a name derived from its IP address and service name.

The erlang name of each pod then needs to be set to `A-B-C-D.k8s-example-erlang-cluster.default.svc.cluster.local.` where `A-B-C-D` is
the IP address with `.` converted to `-`.  This is required so it matches the DNS name of the pod.

To get the pod IP from within the container as an environment variable the following can be used in the pod spec:
```yaml
env:
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
```

Then within the `pre_configure` hook the `ERLANG_NAME` can be set with the `.` for `-` substitution:

```shell
# /rel/hooks/pre_configure/set_erlang_name.sh
export ERLANG_NAME=$(echo $POD_IP | sed 's/\./-/g').k8s-example-erlang-cluster.default.svc.cluster.local.
```

You can then use the same strategy (`Cluster.Strategy.Kubernetes.DNSSRV`) to auto-discover the pods in the stateful set using DNS.

# Conclusion

Setting up a Erlang cluster on Elixir and Kubernetes does require a few steps but can be done quite simply with the `libcluster`
and `distillery` hex packages.

In most cases you will want to use a Statefulset and discover the pod names dynamically.  This will allow scaling, and also give your pods
nice names which makes debugging simpler.  If you do not plan to scale your Statefulset it may be more robust to hardcode the pod names
and not rely on discovery via DNS.  In the case you are resource constrained and can only afford a single pod instance,
a Deployment is the probably best option.

The complete code in the discussion can be found in github [chazsconi/k8s_erlang_cluster_example](https://github.com/chazsconi/k8s_erlang_cluster_example).
