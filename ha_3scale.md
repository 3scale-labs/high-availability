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

## Two clusters in different regions (Active-Passive mode)

This setup consists of having 2 clusters in 2 different regions and deploying
3scale in active-passive mode. This means that one of the clusters will be
operating normally, whereas the other will be in "stand-by" without receiving
traffic, but prepared to receive it in case there's a failure in the other
cluster.

This section focuses on setting an active-passive deployment using Amazon
services, but setting it up in a different cloud platform should be possible as
long as the managed DB services offered can be configured with replicas across
regions.

Notice that the failover procedure described here involves a few manual steps.
Also, there will be some downtime.

### Pre-requisites and installation

- Install Openshift in 2 different regions. Inside the same region, set up
multiple compute nodes in different availability zones.
- Configure a cross-region ElasticCache (using the ["Global Datastore"
feature](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Redis-Global-Datastore.html))
for Backend and another one for System.
- Configure RDS with a [multi-region
replica](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)
for the System DB. From 3scale version 2.10 you can also set it up for Zync.
- Configure a cross-region S3 bucket for the System assets.
- Create all the secrets that contain the credentials (tokens, passwords, etc.)
manually and deploy them with `oc apply` in both clusters. By default, the
3scale operator creates them with some secure random values. However, in this
case, we need to have the same credentials in both clusters. That's why we need
to create them manually. The list of secrets and how to configure them can be
found in the [3scale-operator
docs](https://github.com/3scale/3scale-operator/blob/master/doc/apimanager-reference.md#apimanager-secrets).
In this case, the ones that need to be created are:
    - backend-internal-api
    - system-app
    - system-events-hook
    - system-master-apicast
    - system-seed
- Depending on your setup, you might have different URLs for the DB endpoints in
different regions or a unique one. If the URL does not change, you need to
configure the URLs in these secrets: `backend-redis`, `system-database`,
`system-redis`, and `zync`, and deploy them to the 2 clusters. If the URL is
different, you'll need to create a different version of those secrets for each
cluster.
- Create a custom domain in route53 and point it to the Openshift router of the
cluster that you want to use as the active one.
- Install 3scale in the active cluster using the 3scale-operator as defined
above. The `wildcardDomain` attribute of the spec needs to resolve to the custom
domain created in the step above.
- Install 3scale in the passive cluster. The APIManager custom resource should
be identical to the one used in the step above. After all the pods are running,
change the APIManager to deploy 0 replicas for all the pods of apicast, backend,
system, and zync. We want to have 0 replicas to avoid consuming jobs from the
active DB, etc. We can't tell the operator to deploy 0 replicas of each directly
because the deployment will fail due to some pod dependencies that cannot be met
(some pods check that others are running). That's why, as a workaround, first we
deploy normally, and then, with 0 replicas. This is how it's specified in the
APIManager spec:
```YAML
spec:
  # Other attributes might be here
  apicast:
    stagingSpec:
      replicas: 0
    productionSpec:
      replicas: 0
  backend:
    listenerSpec:
      replicas: 0
    workerSpec:
      replicas: 0
    cronSpec:
      replicas: 0
  zync:
    appSpec:
      replicas: 0
    queSpec:
      replicas: 0
  system:
    appSpec:
      replicas: 0
    sidekiqSpec:
      replicas: 0
  # Other attributes might be here
```

### Manual Failover

- Manual failover of the databases (order does not matter):
  - RDS: System and Zync.
  - Elasticaches: Backend and System.
- In the fallback cluster, edit the APIManager to scale the replicas of the
Backend, System, Zync and Apicast pods that were set to 0.
- In the fallback cluster, recreate the Openshift routes created by Zync. To do
that, run `bundle exec rake zync:resync:domains` from the `system-master`
container of the `system-app` pod. In 3scale version 2.9, this command fails
sometimes. You can retry until all the routes are generated.
- Point the custom domain created in route53 to the Openshift router of the
fallback cluster. From this moment, the fallback cluster will start to receive
traffic, and it becomes the "active" one.
- Scale the replicas of the Backend, System, Zync, and Apicast pods to 0 in the
cluster that started as the active one.
