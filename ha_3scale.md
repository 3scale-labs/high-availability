# Configure HA

This section explains how to install 3scale using the
[3scale-operator](https://github.com/3scale/3scale-operator) and how to
configure it to have a highly-available deployment.

The first thing that you need to do is install the operator. To do so, you can
follow [this
guide](https://github.com/3scale/3scale-operator/blob/3scale-2.9-stable/doc/operator-user-guide.md#installing-3scale).

After installing the operator, we need to deploy an `APIManager` custom
resource. The operator will read that resource and deploy 3scale configured
according to its spec. There's a basic example of an `APIManager`
[here](https://github.com/3scale/3scale-operator/blob/3scale-2.9-stable/doc/operator-user-guide.md#basic-installation).

In order to set up an HA deployment we need to tune a few things in the
`APIManager` spec:
- We need to tell the Operator not to deploy the critical DBs. The Operator can
deploy those DBs, but it cannot set them up to offer HA. To do that, follow
[these
instructions](https://github.com/3scale/3scale-operator/blob/3scale-2.9-stable/doc/operator-user-guide.md#external-databases-installation).
In order to configure HA for those DBs, please refer to [this other
section](ha_dbs.md) of this repo. Notice that the operator does not offer an
option to deploy `system-memcached` and `system-sphinx` externally. That's
because those two are not considered critical, so we don't need an HA deployment
of them.
- We need to deploy at least 2 replicas of each component that can scale. That
includes all the pods except the DBs. Depending on your cluster size and
throughput and latency requirements you might want to deploy more than 2
replicas, at least for the components that are called on each request to the
APIs configured in 3scale, that is, `apicast-production`, `backend-listener`,
and `backend-worker`. To configure that, you need to set the `replicas`
attribute in the `APIManager` spec. There's an example
[here](https://github.com/3scale/3scale-operator/blob/3scale-2.9-stable/doc/operator-user-guide.md#enabling-pod-disruption-budgets).
- We need to enable pod disruption budgets. In summary, this ensures that
there's at least one pod of each type running at all times. You can read more
about this in the [Kubernetes
docs](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets).
To enable this we need to follow [these
instructions](https://github.com/3scale/3scale-operator/blob/master/doc/operator-user-guide.md#setting-custom-affinity-and-tolerations).

With that we have a basic HA set up, but we might need to take some extra steps
depending on the environment where 3scale is running. Next, we present some of
the most common scenarios. We start with the most basic setups, but we will
expand this section in the future.

## Single cluster with one node

This is the most basic scenario. We just need to follow the steps presented in
the above section. Notice that this setup tolerates individual pods failures
because there are multiple replicas, but we'll need to restore a backup of
3scale using the operator if the node fails.

## Single cluster with multiple nodes

With this setup, we'd like 3scale to continue working if a node fails. In order
to achieve that, we just need to add an extra attribute when defining the spec
of the `APIManager`. We need to set affinities in the operator so that when
there are several replicas of a pod they are distributed across the hosts of the
cluster. Check [these
examples](https://github.com/3scale/3scale-operator/blob/master/doc/operator-user-guide.md#setting-custom-affinity-and-tolerations).
You can read more about affinities in the [Kubernetes
docs](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity).
