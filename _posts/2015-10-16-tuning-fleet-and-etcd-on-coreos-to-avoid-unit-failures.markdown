---
layout: post
title:  "Tuning fleet and etcd on CoreOS to avoid unit failures"
date:   2015-10-16 10:12:51
author: Tadas Šubonis
tags:
- feedpresso
- fleet
- etcd
- coreos
- linux
- devops
---

Here at Feedpresso we are relying on [CoreOS](https://coreos.com/) to run our services. We have ran Feedpresso 
on various clouds (Vultr, Google Cloud Compute, Microsoft Azure) and we have learned quite a 
few things along the way. One of the most frustrating issues that we had to face almost every 
time was the [etcd](https://github.com/coreos/etcd) and [fleet](https://github.com/coreos/fleet) runtime behaviour.

In most of theses cases, we had just to appropriately tune them to make everything work again.

In this blog post we won't be claiming that here is everything that you need to know about etcd and fleet tuning or that we
know what's exactly happening there and how each of these parameters could impact you.

Consider this post as introduction to things you might want to try out or check first.

# VM load and latency

Most of these issues arise when there is considerable latency between node communicatons and usually this latency comes
from high Virtual Machine load. We've seen quite often etcd and fleet 
failing when CPU was around 100% for a few minutes. Most of these parameters will be coping with that latency.

# etcd

As etcd is used by fleet, it is required to get *etcd* right first before getting to *fleet*.

On etcd there are basically two properties that you might want to change: _heartbeat-interval_ and _election-timeout_ . The
_election-timeout_ should be **at least** 5 times bigger than  _heartbeat-interval_ (5x - 10x bigger).

We've seen quite often in our setups that _heartbeat-interval_ was set to a too low value. When there is a higher load on the 
machine it started responding slower and it might cause a heartbeat failure. This in turn might cause leader reelection which
would impact fleet.

If you seeing something like this:

```
06:40:30 host fleetd[23463]: INFO client.go:292: Failed getting response from http://localhost:2379/: cancelled
06:40:30 host fleetd[23463]: INFO client.go:292: Failed getting response from http://localhost:2379/: cancelled
06:40:30 host fleetd[23463]: INFO client.go:292: Failed getting response from http://localhost:2379/: cancelled
06:40:45 host fleetd[23463]: INFO client.go:292: Failed getting response from http://localhost:2379/: cancelled
06:40:45 host fleetd[23463]: INFO client.go:292: Failed getting response from http://localhost:2379/: cancelled
```

or

```
googleapi: got HTTP response code 500 with body: {"error":{"code":500,"message":""}}
```

when you are using fleetctl or etcdctrl it is quote likely that you need to bump your _heartbeat-interval_value.

This can be done like this in your cloud-config file:

```
#cloud-config

coreos:
  etcd2:
    heartbeat-interval: 600
    election-timeout: 6000
```

Obviously, you need to finetune them to your set up. If those values are left to be to high, you might not
be able to cope with cluster failures very well.


# Fleet

Fleet is quite sensitive to response times from etcd as well. If it loses connection to etcd and then
recovers it, it would restart all units on that machine and this might not be the thing you would want.

For example, the high database usage would cause a latency increase and that would make fleet lose connection
to etcd. Then when connection is restarted all units are going to be restarted together with the DB that is 
obviously in use at that moment.

In fleet logs it usually looks like this:

```
Apr 09 21:20:31 machine1 fleetd[21543]: INFO engine.go:208: Waiting 25s for previous lease to expire before continuing reconciliation
Apr 09 21:20:31 machine1 fleetd[21543]: INFO engine.go:205: Stole engine leadership from Machine(9c33fade6ddf46448fcbacd8ed8495a5)
Apr 09 21:20:30 machine1 fleetd[21543]: ERROR engine.go:218: Engine leadership lost, renewal failed: 101: Compare failed ([13675908 != 1
```

or

```
fleetd[615]: ERROR reconcile.go:120: Failed fetching Units from Registry: timeout reached
fleetd[615]: ERROR reconcile.go:73: Unable to determine agent's desired state: timeout reached
fleetd[615]: INFO client.go:292: Failed getting response from http://etcd-b1.<OUR_CLUSTER_DOMAIN>:4001/: cancelled
fleetd[615]: ERROR engine.go:149: Unable to determine cluster engine version
fleetd[615]: INFO client.go:292: Failed getting response from http://etcd-b1.<OUR_CLUSTER_DOMAIN>:4001/: cancelled
fleetd[615]: INFO engine.go:81: Engine leadership changed from e9fc4f64602545aca6fbab041d4abae9 to 4ddd4e13813f4e11bbf96622bf3aefe0
fleetd[615]: ERROR reconcile.go:120: Failed fetching Units from Registry: timeout reached
fleetd[615]: ERROR reconcile.go:73: Unable to determine agent's desired state: timeout reached
fleetd[615]: INFO client.go:292: Failed getting response from http://etcd-a1.<OUR_CLUSTER_DOMAIN>:4001/: cancelled
fleetd[615]: INFO client.go:292: Failed getting response from http://etcd-a1.<OUR_CLUSTER_DOMAIN>:4001/: cancelled
fleetd[615]: INFO client.go:292: Failed getting response from http://etcd-a1.<OUR_CLUSTER_DOMAIN>:4001/: cancelled
fleetd[615]: INFO client.go:292: Failed getting response from http://etcd-a1.<OUR_CLUSTER_DOMAIN>:4001/: cancelled
fleetd[615]: INFO client.go:292: Failed getting response from http://etcd-a1.<OUR_CLUSTER_DOMAIN>:4001/: cancelled
fleetd[615]: WARN job.go:272: No Unit found in Registry for Job(spark-tmp.mount)
fleetd[615]: ERROR job.go:109: Failed to parse Unit from etcd: unable to parse Unit in Registry at key /_coreos.com/fleet/job/spark-tmp.mount/object
fleetd[615]: INFO client.go:292: Failed getting response from http://etcd-a1.<OUR_CLUSTER_DOMAIN>:4001/: cancelled
fleetd[615]: INFO manager.go:138: Triggered systemd unit spark-tmp.mount stop: job=1723248
fleetd[615]: INFO manager.go:259: Removing systemd unit spark-tmp.mount
fleetd[615]: INFO manager.go:182: Instructing systemd to reload units
```

In cases like this, you would need to tune _engine-reconcile-interval_, _etcd-request-timeout_ and _agent-ttl_. Most likely you will want to bump _etcd-request-timeout_.
_engine-reconcile-interval_ determines how often your fleetd consults with etcd for unit changes in etcd. It might be good to increase this value to reduce the load on etcd 
at the expense of how fast changes are picked up. 

It is not completely clear what _agent-ttl_ does, but since agent is responsible for actual running and stopping of units 
on the system, this value is used to determine if it is
still possible to do that. In case the agent is dead, it would probably start a new one.



Cloud config could look like this:

```
coreos:
  fleet:
    public-ip: $private_ipv4
    metadata: 
    engine-reconcile-interval: 10
    etcd-request-timeout: 5
    agent-ttl: 120s
```


# Documentation

For more information it would be worth reading these articles:

1. <https://github.com/coreos/fleet/blob/master/Documentation/architecture.md>
2. <https://github.com/coreos/fleet/blob/master/Documentation/deployment-and-configuration.md>
3. <https://github.com/coreos/etcd/blob/master/Documentation/configuration.md>
4. <https://github.com/coreos/etcd/blob/master/Documentation/tuning.md>


Also, it is worth taking a look into these issues:

1. <https://github.com/coreos/fleet/issues/1289>
2. <https://github.com/coreos/fleet/issues/1181>
3. <https://github.com/coreos/fleet/issues/1119>
4. <https://github.com/coreos/fleet/issues/1250>



