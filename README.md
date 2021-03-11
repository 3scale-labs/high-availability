# Highly-available 3scale

This repo contains instructions on how to configure a 3scale installation to be
highly-available (HA).

At the time of writing, the latest release of 3scale is 2.9, but most of the
aspects discussed here apply to any 3scale release.

This document assumes familiarity with the main components of the 3scale
architecture. If you'd like to learn more about them, please check their repos:

- [apicast](https://github.com/3scale/apicast)
- [apisonator (also known as "backend")](https://github.com/3scale/apisonator)
- [porta (also known as "system")](https://github.com/3scale/porta)
- [zync](https://github.com/3scale/zync)

We also assume that you're installing 3scale on [Red Hat
OpenShift](https://www.openshift.com/) using the
[3scale-operator](https://github.com/3scale/3scale-operator).

This repo is divided into the following sections:
- [What happens when 3scale components fail](components.md).
- [Configure HA for 3scale](ha_3scale.md).
- [Set up HA in the DBs used by 3scale](ha_dbs.md).
- [Disaster recovery](disaster_recovery.md).
