---
title: binder: DNS over ZooKeeper
markdown2extras: tables, code-friendly
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# Binder

This is the reference documentation for binder, which is a DNS server built
on top of [ZooKeeper](http://zookeeper.apache.org/), which binder uses to manage
host/service membership, and views as the authoritative source for DNS
information.

Hosts register themselves into Zookeeper using an on-zone agent called
[registrar](http://github.com/joyent/registrar).

This documentation provides descriptions of the semantics binder provides, and
notably the mechanisms by which systems participate in the ecosystem.

# Overview

Binder supports `A` records (and when SDC moves to IPv6, `AAAA`), in either
"traditional" or round-robin fashion, depending on the configuration of the
name.  What is returned is dependent on the type of the record stored in ZK
(all names and stored there).  Effectively, there are "service" records, which
means a highly-available load-balanced service, and everything else.  For the
former, binder is intelligent enough to walk the set of hosts registered in
ZK and return DNS RR records for the load-balancers, not the hosts (the results
are just shuffled).  For individual "host" records (including resolving an LB
itself), binder will return the address IFF the host is up (because the host
will have an ephemeral node).  That's a lot of information that assumes you
understand how ZK works, so let's walk through a practical example to clarify.

Suppose we have "service" foo.joyent.com.  `foo` is the unique name for a set of
load balancers that talk to something. We don't really care what.  When some box
tries to resolve foo.joyent.com (like dig @<binder ip> -t A foo.joyent.com),
binder will imemdiately talk to ZooKeeper (which in most cases will be running
on localhost), and go stat/read `/com/joyent/foo`.  Assuming that exists (this
record should be persistent, regardless of what hosts are present), it should
look like this:

    {
        type: 'service',
        service: {
            srvce: '_http',
            proto: '_tcp'
            port: 80
        },
        ttl: 60
    }

What binder does is treat the JSON payload like a C union. That is, the `type`
field is always expected to be present, and it behaves differently depending on
that.  So here, we see that we have a standard HTTP service, running on port 80
(the payload is set up to eventually support SRV records).  When binder sees
that, it knows it should then look for children in ZK for `/com/joyent/foo`.
Each of those children will have a similar payload, but will be of either type
`load_balancer` or `host`.  In this case, since we asked to resolve the
"service" name, binder looks at all the children, and looks for records like:

    {
        type: 'load_balancer',
        load_balancer: {
            address: 10.1.2.3
        },
        ttl: 30
    }

It takes all the addresses, shuffles them, and returns a set of A records to the
client.  For completeness sake, the host records look pretty much identical:

    {
        type: 'host',
        host: {
            address: 10.1.2.3
        },
        ttl: 20
    }

How load-balancing works is beyond the scope of this document (reference
`muppet`).  Usually all of the host and LB records are "ephemeral" in ZK,
which means when a host dies, it automatically falls out of DNS rotation, and
when it comes back or is replaced, it is phased back in automatically. That
said, binder does keeps a short-lived cache over ZK so it's not whacking
ZK for every resolve call.  To finish out the example, while we illustrated
how binder would resolve `foo.joyent.com` first, we can always ask for a host
we know is an individual member of that service. So, for example, asking to
resolve `087d866d-b88a-4a78-80ec-750c26f6eccf.foo.joyent.com`, binder would go
read `/com/joyent/foo/087d866d-b88a-4a78-80ec-750c26f6eccf` and (assuming it's
a host) see:

    {
        type: 'host',
        host: {
            address: 10.1.2.3
        }
    }

So it would do no further logic, and simply return an A record of with that IP.

Lastly, as part of the "binder ecosystem", you probably want to look at
`registrar`, which is a small agent expected to run on every host.  It starts up
and registers $self with ZooKeeper.  Other hosts should have their resolv.conf
set to point at the ring of ZooKeeper/binder IPs.

# Record Format Reference

## Host

Host records are the simplest, and are expected to exist anywhere from `/` in
ZK with the DNS record reversed.  As above `foo.joyent.com` should live at
`/com/joyent/foo`.  The JSON payload at that path is very simple:

    {
        type: 'host',
        host: {
            address: <your ip here>
        },
        ttl: 30
    }

While binder doesn't care, these nodes are expected to be ephemeral. These
nodes should be a leaf node.

## load_balancer

A `load_balancer` record is nearly identical to a `host` record, just the type
is set differently, such that when a load_balancer is a child of a service
entry, binder returns RR LB addresses:

    {
        type: 'load_balancer',
        load_balancer: {
            address: <your ip here>
        },
        ttl: 30
    }

While binder doesn't care, these nodes are expected to be ephemeral. These
nodes should be a leaf node.

## service

A `service` record is expected to be a non-leaf node, such that it has child
load_balancer and host records underneath it.  Service records are filled in to
look like an SRV record (although SRV records are not currently supported); all
that is honored right now for RR records is the TTL (i.e., weights are not
supported).

    {
        type: 'service',
        service: {
            srvce: '_http',
            proto: '_tcp',
            port: 80
        },
        ttl: 60
    }

These records should be persistent, and non-leaf nodes.

## database

A `database` record is intended to only be written by the postgres/manatee
system.  It strongly assumes the record maps to a PG shard record that manatee
uses, which means it _must_ be structured as an object with `primary`, `standby`
and `async`, as either `pg://` or `tcp://` url, and the hostname must be an
IP address (i.e., if you have `tcp://foo@db.example.com/pg`, bad things will
happen).

    {
        type: 'database',
        database: {
            primary: 'tcp://user@192.168.0.1/postgres',
            standby: 'tcp://user@192.168.0.2/postgres',
            async: 'tcp://user@192.168.0.3/postgres'
        },
        ttl: 10
    }

While binder doesn't care, these nodes are expected to be persistent. It also
doesn't matter if these are written as a leaf-node or not (i.e., binder doesn't
try to do anything more once it sees type=database).  In most cases, each DB
host will have an A record at hostA.db.example.com (e.g., as an immediate
child).

# Configuration

As binder is expected to run on the same host as a ZooKeeper server, it is
hard-coded to talk to `::1` to find ZK.  You can override this by setting the
environment variable `ZK_HOST` to some other IP address.  This is really only
for testing.  Also, it has a fixed in-memory cache of 1000 elements and 60s
expiration time over ZK.  These can be overriden with command line flags of
`-s` and `-a`, respectively.  Although, again, it's defaulted, and hard coded
in SMF that way.  There is no config file for binder.

# Troubleshooting

binder is pretty simple software, and if it's not returning what you expect, you
almost assuredly have ZK misconfigured, or updated one that binder isn't
attached to (like the one that ships with usb-headnode...).  That said, you can
hack the SMF manifest in /opt/smartdc/binder/smf/manifests/binder.xml and add
`-d 2` to spew all nature of logs via SMF.
