---
title: "Walk through: OVN Chassis Binding Process"
date: 2023-09-19 15:00:00 +0700
categories: [OVN]
tags: [ovn, live-migration]
---

In this post, I will walk you through the process that OVN follows to determine
chassis to bind a particular port at.

## Introduction

In virtual environments, it's important to be able to control location of a
port. Your compute workload (a virtual machine or a container) is running on a
particular node (chassis), and it's obvious that you want its ports to be
implemented on the same node. (Otherwise it won't have any network
connectivity.)

For this reason, OVN provides API to enforce a particular locality for a port
binding. This can be achieved by setting a particular option for a
`Logical_Switch_Port`, namely, `options:requested-chassis`.

## Implicit binding locality

But is using `options:requested-chassis` required? Not really. Let me
elaborate.

The way `ovn-controller` works is, it monitors all OVS interfaces that are
attached to its integration bridge (usually, `br-int`). Whenever a new
interface pops up on the bridge, it checks its `external_ids:iface-id` key and,
if a matching `Port_Binding` with the same name exists in the OVN Southbound
database, then `ovn-controller` "claims" the port for itself.

The OVS interface that carries `external_ids:iface-id` is created by CMS (e.g.
in OpenStack, it's created by Nova Compute,
[using os-vif library](https://github.com/openstack/os-vif/blob/0a0dec37e42e220ffa750650027714bd7289faa1/vif_plug_ovs/ovsdb/ovsdb_lib.py#L172).)
The external entity that creates the OVS interface may make sure that only one
interface with a particular `external_ids:iface-id` exists in the cluster at
any particular moment, in which case all `ovn-controller` instances, except
one, will ignore the `Port_Binding`.

And yet, there are situations when it may be impossible, or hard, to guarantee
this uniqueness in the cluster. In this case, it may be wise to explicitly
limit the locality of a logical port to a chassis. And that's where
`options:requested-chassis` comes handy.

## CMS to OVN Chassis, and back again

Explicit request of a particular chassis starts when the CMS creates a
`Logical_Switch_Port` in the OVN Northbound database. The CMS creates a logical
port and sets this option (as a string) in the database. For example:

```shell
$ ovn-nbctl list Logical_Switch_Port
...
_uuid               : 61acac6b-2410-465d-9db5-197e18f11259
addresses           : ["fa:16:3e:10:e6:74 10.0.0.44 fd23:ded7:305c:0:f816:3eff:fe10:e674"]
name                : "6d3d9b33-f468-482f-8b0a-6ffd41f71d40"
options             : {mcast_flood_reports="true", requested-chassis=compute1}
type                : ""
up                  : false
...
```

Once `ovn-northd` sees the new `Logical_Switch_Port` in the Northbound
database, it translates it into a corresponding `Port_Binding` record in the
Southbound database. Among other things, it attempts to locate a `Chassis`
record with the same `name` field as the name listed in
`options:requested-chassis` and, assuming it's found[^0], fills in the
corresponding `requested_chassis` field of the `Port_Binding` record[^1].

```shell
$ ovn-sbctl list Port_Binding
...
_uuid               : 7c195cdf-3fb6-4b33-9926-72467187f1c3
chassis             : []
logical_port        : "6d3d9b33-f468-482f-8b0a-6ffd41f71d40"
mac                 : ["fa:16:3e:10:e6:74 10.0.0.44 fd23:ded7:305c:0:f816:3eff:fe10:e674"]
options             : {mcast_flood_reports="true", requested-chassis=compute1}
requested_chassis   : dea9e5b0-95ad-457c-814a-f1dc0527c5ba
up                  : false
...
```

In the meantime, the CMS may also create an OVS interface on the chassis of
choice. The `ovn-controller` instance running on the chassis will detect this
event, pull the `Port_Binding` record with the corresponding name and, if
`requested_chassis` field is set, will compare its own `Chassis` to the
requested one. If they don't match, it will ignore the interface, meaning that
the interface OpenFlow flows won't be configured.

But if the `Chassis` matches, then the `ovn-controller` instance will "claim"
this `Port_Binding`.

## Port claims

Whether the `Port_Binding` was tagged with `requested_chassis` or not, if the
`ovn-controller` decides to "claim" the port, the following two important
actions are taken.

1. **OpenFlow tables are updated to implement the port on the chassis.** This
   will allow the workload that uses the port to connect to the network.
2. **Southbound database is updated to confirm that the port is configured.**
   This feedback can then be used by the CMS to indicate to the port owner that
   the port is ready for use.

There are several `Port_Binding` fields that are updated as part of the
process. Among them are:

1. **`chassis`**: It is updated to refer to the `Chassis` record that
   corresponds to the current chassis. In case when `options:requested-chassis`
   is used, both `requested_chassis` and `chassis` fields of the binding will have
   identical values.
2. **`up`**: It is set to `true`. This information is then proxied back to the
   Northbound database that is visible to CMS (and - through the integration
   driver - to the CMS end user).

```shell
$ ovn-sbctl list Port_Binding
...
_uuid               : 7c195cdf-3fb6-4b33-9926-72467187f1c3
chassis             : dea9e5b0-95ad-457c-814a-f1dc0527c5ba
logical_port        : "6d3d9b33-f468-482f-8b0a-6ffd41f71d40"
mac                 : ["fa:16:3e:10:e6:74 10.0.0.44 fd23:ded7:305c:0:f816:3eff:fe10:e674"]
options             : {mcast_flood_reports="true", requested-chassis=compute1}
requested_chassis   : dea9e5b0-95ad-457c-814a-f1dc0527c5ba
up                  : true
...
```

## Claim storms

There may be situations when, due to a mistake or asynchronous nature of cloud
environments, multiple chassis end up having an OVS interface that lists the
same `external_ids:iface-id`. Such situation may be problematic, because each
of the `ovn-controller` instances on different chassis may try to claim the
same port binding for itself.

Moreover, since they always try to converge to the state where they are the
owners of the binding, each of them will try to update `chassis` field of the
binding over and over, effectively producing a transaction storm for the
Southbound database. This may not only affect the dataplane connectivity for
the port in question, but also generate a significant load on the database, as
well as any of its consumers that may e.g.  monitor OVSDB events and will have
to process the storm of update events.

When this happens, you may observe `ovn-controller` log files filled in with an
enormous number of repeating messages, as in:

```
2021-06-21T21:28:45.825Z|18183|binding|INFO|Changing chassis for lport 6d3d9b33-f468-482f-8b0a-6ffd41f71d40 from 0fd5b282-a152-47e6-84a9-c3d5645ffe86 to a16c2360-e86e-45ec-9223-103e9fe813c7.
2021-06-21T21:28:45.825Z|18184|binding|INFO|6d3d9b33-f468-482f-8b0a-6ffd41f71d40: Claiming fa:16:3e:10:e6:74 10.0.0.44/24
2021-06-21T21:28:45.831Z|18185|binding|INFO|Changing chassis for lport 6d3d9b33-f468-482f-8b0a-6ffd41f71d40 from 0fd5b282-a152-47e6-84a9-c3d5645ffe86 to a16c2360-e86e-45ec-9223-103e9fe813c7.
2021-06-21T21:28:45.831Z|18186|binding|INFO|6d3d9b33-f468-482f-8b0a-6ffd41f71d40: Claiming fa:16:3e:10:e6:74 10.0.0.44/24
2021-06-21T21:28:45.836Z|18187|binding|INFO|Changing chassis for lport 6d3d9b33-f468-482f-8b0a-6ffd41f71d40 from 0fd5b282-a152-47e6-84a9-c3d5645ffe86 to a16c2360-e86e-45ec-9223-103e9fe813c7.
```

This situation is obviously undesirable, that's why we
[now](https://github.com/ovn-org/ovn/commit/4dc4bc7fdb848bcc626becbd2c80ffef8a39ff9a)
make sure that `ovn-controller` does not attempt to claim the same port binding
more often than every `0.2` seconds.

But is it always problematic when multiple chassis carry an OVS interface for
the same `Port_Binding`? No, sometimes it's actually helpful.

## Live migration

One scenario that benefits from being able to bind the same binding to multiple
chassis is Live Migration. In brief, Live Migration allows to migrate a running
compute workload (a virtual machine) from one hypervisor to another, without
visibly interrupting the execution of the workload. The actual virtual machine
migration is handled by the underlying hypervisor techonology (e.g.  libvirt)
and is beyond the scope of the blog post.

In Live Migration scenario, it's very important that not only the compute
service (as seen by the virtual machine owner) is not interrupted, but that the
corresponding network service is not interrupted either. (What good makes a
virtual machine that can crunch numbers but that cannot communicate with the
outside world?)

In OpenStack, Nova Compute first migrates the running virtual machine to
another hypervisor; then, once the completion of the process is detected, it
informs the other OpenStack components, specifically, Storage (Cinder) and
Network (Neutron), about the change in the port binding location. At this
point, each service propagates the change through its databases (Neutron -> OVN
NB -> OVN SB) and, eventually, configure the port in the new chassis to provide
connectivity for the virtual machine running in the destination host.

The problem with this approach is that, until

- Nova informs Neutron, and
- Neutron updates OVN Northbound service, and
- OVN updates the new hypervisor `ovn-controller` instance, and
- it has a chance to configure the new OpenFlow tables for the port

...the virtual machine is disconnected from the network. This process is,
first, not immediate, and, second - and probably more important! - depends on
all controller services in the chain to be available and instant.  If anything
goes wrong, the connectivity for the virtual machine may not restore for a
significant time.  Not exactly the experience we would like our users to have.

## Multichassis port bindings

The core of the problem in the scenario above is that OVN will not configure
the port on the destination hypervisor long after the virtual machine is
already running there. There is no hard reason for this to happen though.

This is why we introduced a new feature in OVN called Multichassis port
bindings. The basic idea behind it is that a particular port binding can now be
bound to more than one chassis.

In Live Migration scenario, this allows the `ovn-controller` instance that is
running on the destination hypervisor to start configuring the port long before
the virtual machine actually starts running there. Once it does, the port and
its OpenFlow flows are ready to serve ingress and egress traffic for the port.

From the CMS perspective, the only difference in the interface with OVN is that
instead of a single value set for `options:requested-chassis`, for the duration
of Live Migration, it
[sets](https://github.com/openstack/neutron/blob/dbe4ba910b3236ff3ac42e33dcb4cc067b1f9177/neutron/plugins/ml2/drivers/ovn/mech_driver/ovsdb/ovn_client.py#L445)
the option to a comma separated list of chassis names.

What happens then is that `ovn-northd` updates the fields in the corresponding
`Port_Binding` record as follows:

1. The `requested_chassis` field is set to refer to the `Chassis` that
   corresponds to the first element of the list.
2. The new[^2] `additional_requested_chassis`[^3] field is set to refer to a
   list of the `Chassis` records that correspond to the rest of chassis names
   from the `options`.[^4]

Once `Port_Binding` is updated, each `ovn-controller` that has an OVS interface
with the corresponding `exernal_ids:iface-id` name will claim the port as
follows.

1. If the current chassis is not listed in either `requested_chassis` or
   `additional_requested_chassis` fields, then the port is not claimed for the
   chassis.
2. If the current chassis is listed in `requested_chassis`, then the port is
   claimed by the chassis, confirming this chassis as "main". Among other
   things, the `Port_Binding.chassis` field is updated to refer to the current
   chassis. The main chassis will also update the `Port_Binding.up` field.
3. If the current chassis is listed in `additional_requested_chassis`, then the
   port is claimed by the chassis, confirming this chassis as "additional".
   Among other things, the `Port_Binding.additional_chassis` field is updated to
   include the current chassis. The additional hypervisor will *not* touch
   `Port_Binding.up` field.

The final state of the `Port_Binding` record may now look as follows.

```shell
$ ovn-sbctl list Port_Binding
...
_uuid               : 7c195cdf-3fb6-4b33-9926-72467187f1c3
additional_chassis  : [0fd5b282-a152-47e6-84a9-c3d5645ffe86]
chassis             : dea9e5b0-95ad-457c-814a-f1dc0527c5ba
logical_port        : "6d3d9b33-f468-482f-8b0a-6ffd41f71d40"
mac                 : ["fa:16:3e:10:e6:74 10.0.0.44 fd23:ded7:305c:0:f816:3eff:fe10:e674"]
options             : {mcast_flood_reports="true", requested-chassis=compute1,compute2}
requested_additional_chassis   : [0fd5b282-a152-47e6-84a9-c3d5645ffe86]
requested_chassis   : dea9e5b0-95ad-457c-814a-f1dc0527c5ba
up                  : true
...
```

At this point, both chassis have the port wired and ready to serve traffic.
Once the hypervisor completes Live Migration to the new chassis, OVN will -
hopefully - be ready to immediately serve it there.

## Finalizing migration

At this point, the virtual machine is successfully running on the new chassis,
and its network connectivity is working. But the CMS still has the old port
binding, no longer used, dangling on the original chassis. It's wise to garbage
collect it. To achieve this, the CMS updates the
`Logical_Switch_Port.options:requested-chassis` to list the only active
chassis and allows the OVN machinary to update Southbound database and OpenFlow
tables.

---

[^0]: Each `ovn-controller` is supposed to create and maintain its own
      `Chassis` record in the Southbound database.

[^1]: The `requested_chassis` field in the Southbound database is a reference
      to the corresponding `Chassis` record, and not a string.

[^2]: In an ideal world, we would not introduce a separate
      `additional_requested_chassis` and `additional_chassis` fields. Instead,
      we would transform the existing `requested_chassis` and `chassis` fields
      into lists. Sadly, we had to consider backwards compatibility for the
      database schema transformations.

[^3]: _chassis_ here is plural.

[^4]: In case of Live Migration, the list will never contain more than two
      chassis names. Regardless, the feature was implemented in such a way
      that allows to define more than two chassis in the list. Such use is
      beyond the scope for this blog post.

