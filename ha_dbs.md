# Set up HA in the 3scale DBs

As mentioned in [this other section](ha_3scale.md) of this repo, in order to set
up HA for the 3scale DBs, we need to tell the operator that we are going to
manage them ourselves. This applies to both the Backend and the System Redis,
and also, the relational databases used by System, and starting from v2.10, also
Zync.

We can choose between different options when setting up the DBs. One of the
first questions that we need to answer is whether we want to deploy them on
premises or if we'd like to use a managed service provided by a cloud provider.
Some users might be required by law to have everything on-premises. For users
who are not, and are already using services provided by Amazon, for example,
using their services also for the 3scale DBs might be the most convenient
option.


## Backend Redis and System Redis

Redis offers replication. You need to set up multiple instances of Redis. One of
them will be the leader and the others the replicas. Writes go to the leader and
are propagated to the replicas. When the leader fails, a replica becomes the new
leader. When the old leader comes back online it will become a replica of the
new leader.

In Redis, the replication mechanism is asynchronous. This means that when
something is written to the leader, Redis answers to the client without waiting
for the writes to be effective in the replicas. This is fast, but when there is
a failover, we'll lose the data that has not been replicated yet.

In the case of Backend, that trade-off makes sense. In the event of a failover,
Backend might lose some reports. This means that some rate-limits could not be
accurate and we could let more calls than configured for a brief period of time.
Also, some statistics might not be exact. This is a trade-off that makes sense
in the case of Backend. It needs to be fast because it is called on every API
call (unless there's some caching policy enabled in APIcast), and failovers
should be pretty rare, plus the amount of data not in the replicas should be
pretty low. Apart from that, the Backend DB contains some information
synchronized from System that could also be lost, but that can always be
recovered by executing a rake task from System.

Here are some options available to use as the Backend and System Redis:

- [Redis with sentinels](https://redis.io/topics/sentinel). This is the option
you need to choose if you want to deploy everything on-premises.
- [AWS
Elasticache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/WhatIs.html).
Check the
[docs](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Replication.Redis.Groups.html).
It's [multi
AZ](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/AutoFailover.html),
and can also be multi-region using the ["Global Datastore"
feature](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Redis-Global-Datastore.html).
Keep in mind that in the multi-region case failovers are not automatic.
- [Redis Enterprise](https://redislabs.com/redis-enterprise-software/overview/).
Check the
[docs](https://redislabs.com/redis-enterprise/technology/highly-available-redis/).
It's multi AZ, and can also be configured in [multiple
regions](https://redislabs.com/redis-enterprise/technology/active-passive-geo-distribution/).

When choosing and configuring any of those or other options, keep in mind that
Backend does not support the [Redis Cluster
mode](https://redis.io/topics/cluster-tutorial).

In the case of Backend you can deploy its databases in a single Redis instance
using two different DB indexes, but you can also use two instances if the option
that you choose does not support this.

Take into account that using the same Redis instance both for Backend and System
is not supported.

## System and Zync DBs

Here are some options for the relational database used by System and Zync. Keep
in mind that they need to be two separate DBs:
- [Crunchy](https://www.crunchydata.com/). Read more about it
[here](https://access.crunchydata.com/documentation/postgres-operator/4.6.1/architecture/high-availability/multi-cluster-kubernetes/)
and
[here](https://access.crunchydata.com/documentation/postgres-operator/4.6.1/advanced/multi-zone-design-considerations/).
It can be deployed using a Kubernetes operator. By default, the replication is
asynchronous, but it can be [configured to be
synchronous](https://access.crunchydata.com/documentation/postgres-operator/4.6.1/architecture/high-availability/).
It can be configured to be multi-cluster, but in that case, the failover is not
automatic. Also, the replication in that scenario is synchronous and uses S3 as
an intermediate storage.
- [AWS RDS](https://aws.amazon.com/es/rds/ha/). The replication is synchronous.
It supports [multi-AZ and multi-region
replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html).
