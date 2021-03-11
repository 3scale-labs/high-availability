# What happens when the 3scale components fail

This section explains what features stop working when each of the pods of the
3scale installation are not available.

## apicast-production

APIcast proxies calls to the APIs configured in 3scale. If `apicast-production`
goes down, production traffic stops working.

## apicast-staging

If `apicast-staging` goes down, staging traffic stops working.

## backend-cron

When a `backend-worker` job fails, it is pushed to a "failed jobs" queue so that
it can be retried later. Jobs can fail, for example, when there's a Redis
timeout. `backend-cron` is responsible for rescheduling those failed jobs. That
means that while `backend-cron` is down, those jobs will not be rescheduled. If
the 3scale installation is working correctly, the failed jobs queues will be
empty almost at all times, so `backend-cron` being down is not critical.

`backend-cron` is also responsible for deleting the stats of services that have
been removed. This is run every 24h. If `backend-cron` crashes in the middle of
the delete process, it will just continue the next time it runs.

## backend-listener

This is the component responsible for authorizing and rate-limiting requests.
When it goes down, APIcast is not be able to tell whether a request should be
authorized or not and simply denies everything. That means that when
`backend-listener` is down, the APIs configured using 3scale are down too. This
can be mitigated by using the APIcast caching policy, but before using it, make
sure to check whether the trade-offs it makes suit your use case.

## backend-redis

It's the DB used by `backend-listener` and `backend-worker`, when it goes down,
those 2 pod groups stop working.

## backend-worker

Processes the background jobs created by `backend-listener`. Those jobs contain
the reported metrics. When this component goes down, the reported metrics can't
be made effective. That means that the rate-limiting functionality loses
accuracy because those pending reports are not taken into account. Also,
statistics will not be up to date and the alerts and errors shown in the admin
portal will not be triggered.

## system-app

If this pod goes down, many things will stop working including: admin portal UI,
3scale APIs (accounts, analytics, etc.), developer portals, etc. Also, APIcast
will not be able to retrieve the gateway configuration. APIcasts that are
already running will still be able to serve traffic with the latest
configuration that they downloaded, but new APIcast deployments will not work.

## system-memcache

It's an ephemeral cache used by `system-app`. If it's down, `system-app` will
need to perform all the queries against its DB making things slower.

## system-mysql

It's the DB used by `system-app`. If it's down, `system-app` stops working.

## system-redis

Used by `system-app` as a queue for sidekiq jobs. Some of the most important
jobs perform tasks such as: synchronize info with backend and zync, send emails,
trigger alerts, etc. If `system-redis` goes down, those jobs will not be
enqueued.

## system-sidekiq

It's the job manager used by `system-app` to process jobs in the background. If
it goes down, jobs will not be processed and will start accumulating in
`system-redis`.

## system-sphinx

If it goes down, the search functionality on the `system-app`admin portal
(accounts and proxy rules search bars) stops working. Similarly, on the dev
portal, templates and forum searches stop working too.

## zync

It receives requests from `system-app` to create Openshift routes for the admin
portals, developer portals, and APIcasts. It also receives requests to
synchronize information with the IDP if configured. When `zync` is down,
`system-app` will retry the failed requests for some time.

## zync-database

The DB used by `zync`. It contains job queues and also some data synchronized
from `system-app`. When it's down `zync` will not be able to enqueue jobs.

## zync-que

Job manager used to process the jobs created by `zync`. When it's down, the jobs
are not processed.
