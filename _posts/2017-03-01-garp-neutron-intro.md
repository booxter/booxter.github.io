---
title: "Gratuitous ARP in OpenStack Neutron with ML2/OVS"
description: "
  This post is an introduction to gratuitous ARP in OpenStack Neutron.
  It describes the protocol and its usage in OpenStack Neutron.
  The next post will discuss some of the issues we had with it.
  "
date: 2017-03-01 15:00:00 +0700
tags: [openstack, ovs, neutron, networking, kernel]
---

This and the [next](../garp-neutron-war-story) posts are an adoption of my old
__2017__ series from another blog. The factual material may be obsolete. I'm
keeping the posts here for historical reasons, and also because I find the
narrative of the story somewhat curious. Some minor stylistic changes were
applied. Obsolete links updated as needed.

---

## ARP (Address Resolution Protocol)

### Motivation

ARP is one of the most widely used protocols in modern networks. Its history
goes back to early 80s into the times of DARPA backed internetworking
experiments. The very first
[RFC 826](https://datatracker.ietf.org/doc/html/rfc826) that defined the
protocol is dated November 1982 (it's 35 years old at the time of writing).
Despite the age, it's still a backbone of local IPv4 network connectivity. Even
in 2017 (the year I draft this post), it's still very hard to find a IPv6-only
network node, especially outside cloud environments. But that's IPv4, so how
does ARP fit into the picture?

To understand the goal of ARP, let's first look at how network nodes are
connected. The general model can be described as a set of hosts, each having
one or more Network Interface Controller (NIC) cards connected to a common data
link fabric. This fabric comes in different flavors (Ethernet, IEEE 802.11 aka
WiFi, or FireWire). Irrespective of particular fabric flavor, all of them
provide similar capabilities. One of the features that are expected from all of
them is some form of endpoint addressing, ideally globally unique, so that
network hosts connected to a shared medium can distinguish each other and
address transferred data to specific peers. Ethernet and IEEE 802.11 are
probably the most popular data link layers in the world, and since they are
largely identical in terms of NIC addressing, in next discussions we will
assume Ethernet fabric unless explicitly said otherwise.

For Ethernet, each NIC card produced in the world gets a unique 48-bit long
hardware address allocated by a vendor under IEEE supervision that guarantees
that no hardware address is allocated to two NIC cards. Uniqueness is to ensure
that whichever hardware you plug into your network, it will never clash in
address space with any other card also attached to the network. An example of a
EUI-48 address would be, in commonly used notation, f4:5c:89:89:cd:54. These
addresses are widely known as MAC addresses, and so I will also use this term
moving forward.

It all means that your NIC already has a unique address, so why do you even
need IP addresses? Sadly, people are bad at memorizing 48 randomized bits, so
an easier scheme would be handy. Another problem is whenever your NIC dies and
you replace it with a new one, the new card will have another unique address,
and so you would need to advertise the new MAC address to all your network
peers that may need to access your host.

And so engineers were looking for a better scheme to address network hosts. One
of those successful alternative addressing proposals was IPv4. In this scheme,
IPv4 addresses are defined 32-bit long. Still a lot, but the crucial point is
that now you could pick addresses for your NIC cards. With that freedom, you
could pick the same bit prefix for all your hosts, distinguishing them by a
shorter number of trailing bits, and memorize just those unique bits, and
configure your networking software to use the same prefix for network
communication with other hosts. Then whenever you want to address a host, you
pass unique trailing bits assigned to the host into your networking stack and
allow it to produce the resulting address by prepending the common prefix.

The only problem with this approach is that now you have two address schemes:
MAC addresses and IP addresses, with no established mapping between them. Of
course, in small networks, you could maintain static IP-to-MAC mappings in sync
on every host, but that is error prone and doesn't scale well.

And that's where ARP comes in to the stage. Instead of maintaining static
mappings across hosts, the protocol allows to dynamically disseminate the
information on the wire.

Quoting the abstract of RFC 826:

> Presented here is a protocol that allows dynamic distribution of the
> information needed to build tables to translate an address A in protocol P's
> address space into a 48.bit Ethernet address.

And that's exactly what we need.

While the abstract and even the RFC title talk about Ethernet, the mechanism
rendered so successful that it was later expanded to other data links,
including e.g. [FireWire](https://datatracker.ietf.org/doc/rfc2734/).

### Basics

The protocol introduces both ARP packet format as well as its state
machine. Sadly, the RFC doesn't contain a visual scheme for ARP packets, but
we can consult the protocol Wikipedia
[page](https://en.wikipedia.org/wiki/Address_Resolution_Protocol#Packet_structure).

The RFC describes an address translation (ARP) table for each host storing
IP-to-MAC mappings. It also defines two operations: a REQUEST and a REPLY.
Whenever a host wants to contact an IP address for which there is no mapping in
the local ARP table, the host sends a REQUEST ARP packet to broadcast
destination MAC address asking the question "Who has the IP address?" Then it's
expected that the host carrying the IP address will send a REPLY ARP packet
back with its own MAC address set in "Sender hardware address" field. The
original host will then update its ARP table with a new IP-to-MAC mapping and
will use the newly learned value as a destination MAC address for all
communication with the IP address.

One thing to clarify before we move forward: this is all true assuming both
interacting hosts are on the same layer-2 network segment, without an IP
gateway (router) in between. If hosts are located in different segments, then
connection between them is established through a router. In this case, a host
willing to communicate with a host in another segment will determine that fact
by inspecting its IP routing table. Since the destination IP address then
would not belong to the local network IP prefix, the host will instead send the
data to the default router IP address. (Of course, at this point the host may
also determine that its ARP table doesn't contain an entry for the gateway IP
address yet, in which case it will use ARP to learn about the router MAC
address.)

### ARP table invalidation

One interesting aspect of the original RFC is that it doesn't define a
mechanism to update existing ARP table entries with new MAC addresses. Back in
1982, it was probably widely assumed that mobile IP stations roaming across
network segments changing devices used to connect to outside world on the fly
(think about how your smartphone seamlessly switches from WiFi to
LTE) were not a too realistic use case. But even then, in "Related issue"
section of the document, some ideas on how it could be implemented if needed
were captured.

One suggestion was for every host to define "aging time" for its ARP entries.
If a peer host is detected as unreachable (probably because there was no
incoming traffic using both the MAC and IP addresses stored in ARP table), the
originating host could remove the corresponding ARP entry from its table after
it's "aged". This mechanism is indeed used in most modern ARP implementations,
with 60 seconds being the common default for Linux systems (can be overridden
using
[gc_stale_time](https://git.kernel.org/pub/scm/docs/man-pages/man-pages.git/tree/man7/arp.7?id=e043552f408112c3b1843a7e0b3b2fddb6ab94d1#n195)
sysctl setting).

It means that your connectivity to a roaming IP host will heal itself after a
minute of temporary down time. While that's great, some use cases would benefit
from a more rapid reaction of hosts to network changes.

And that's where gratuitous ARP comes into play.

### Gratuitous ARP

Gratuitous ARP is an ARP packet that was never asked for (hence its alternative
name - unsolicited ARP). RFC 826, "Related issue" section, mentions an
algorithm to update existing ARP table entries in the network based on
unsolicited ARP packets. But it's only
[RFC 2002, "IP Mobility support"](https://datatracker.ietf.org/doc/html/rfc2002)
from year 1996 that made it part of a standard and introduced the very term
"gratuitous ARP".

RFC 2002 discusses protocol enhancements for IP networks to allow for IP
devices roaming across networks without introducing significant connectivity
delays or disruptions. Among other things, it defines the algorithm to be used
to update existing ARP table entries with new MAC addresses. For this matter,
it adopts the proposal from RFC 826, where a host can broadcast a gratuitous
ARP packet into a network, and its peers then update their tables with the new
MAC address sent, restoring connectivity even before old ARP entries expire.

There are two main use cases for gratuitous ARP. One is to quickly switch
between multiple devices on the same host. Another is to move services exposed
through an IP address from one host to another transparently to network peers.

This last scenario may happen either as part of a planned action on behalf of
an Ops team managing a service, or triggered by a self-healing mechanism used
in networks to guarantee availability of services in case of software or
network failures. One piece of popular software that allows to fail over IP
addresses from one host to another is [keepalived](https://www.keepalived.org/)
which uses the VRRP protocol to negotiate between hosts which node should carry
IP addresses managed by the software.

In OpenStack Neutron, gratuitous ARP is how floating IP addresses roam between
ports; they also help with failing over IP addresses between HA router
instances.

## Gratuitous ARP for OpenStack Neutron

### Usage

To recollect, the primary use for gratuitous ARP in OpenStack Neutron L3 agent
is to update network peers about the new location of a "floating" IP address
("elastic" in AWS-speak) when it's disassociated from one port and then
associated to another port with a different MAC address. Without issuing a
gratuitous ARP on new association, it may take significant time before a reused
floating IP address mapping is updated as a result of the "aging" process.

Gratuitous ARP is also used by the L3 agent to implement HA for Neutron
routers. Whenever a new HA router instance becomes "master", it adds IP
addresses managed by Neutron to its interfaces and issues a set of gratuitous
ARP packets into attached networks to advertise the new location. Network peers
then update their ARP tables with new MAC addresses from those packets and in
this way don't need to wait for old entries to expire before connectivity would
be restored. The switch to the new router instance is then a lot smoother.

### Implementation

There are two distinct implementations for gratuitous ARP in OpenStack Neutron,
one for each distinct router deployment mode: legacy and HA. The difference
comes primarily from the fact that legacy router data plane is fully realized
by the L3 agent, while HA routers "outsource" IP address management to
`keepalived` daemon spawned by the agent. (The third deployment mode - DVR - is
largely covered by those two, where specific implementation depends on whether
DVR routers are also HA or not; for this reason I won't mention DVR going
forward).

Let's consider each distinct deployment mode separately, starting with legacy.

#### Legacy routers

Legacy mode is what once was the only mode supported by OpenStack Neutron. In
this mode, the L3 agent itself implements the whole data plane, creating
[network namespaces](https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)
for routers, creating ports, plugging them into the external `br-ex` bridge,
and adding fixed and floating IP addresses to router ports. Besides that, the
agent also issues gratuitous ARP packets into attached networks when a new IP
address is added to one of its ports. This is to update network peers about the
new mappings. Peers may use those unsolicited updates either to update any
existing ARP entries with a new MAC address, or to "warm up" their tables with
IP-to-MAC mappings even before the very first IP datagram is issued to the
router IP address (this is something that Linux kernel does when
[arp_accept](https://github.com/torvalds/linux/blob/3ef2bc099d1cce09e2844467e2ced98e1a44609d/Documentation/networking/ip-sysctl.txt#L1221-L1232)
sysctl setting is enabled for the receiving interface).

When the L3 agent sends gratuitous ARP packets for an IP address, this is what
you can find in the agent log file:

```
2017-04-28 20:53:11.264 14176 DEBUG neutron.agent.linux.utils [-] Running command (rootwrap daemon): ['ip', 'netns', 'exec', 'qrouter-726095be-5916-489b-be05-860e2f19d556', 'ip', '-4', 'addr', 'add', '10.1.0.1/26', 'scope', 'global', 'dev', 'qr-864545b9-5f', 'brd', '10.1.0.63'] execute_rootwrap_daemon /opt/stack/new/neutron/neutron/agent/linux/utils.py:108
```

And then later:

```
2017-04-28 20:53:11.425 14176 DEBUG neutron.agent.linux.utils [-] Running command (rootwrap daemon): ['ip', 'netns', 'exec', 'qrouter-726095be-5916-489b-be05-860e2f19d556', 'arping', '-A', '-I', 'qr-864545b9-5f', '-c', '3', '-w', '4.5', '10.1.0.1'] execute_rootwrap_daemon /opt/stack/new/neutron/neutron/agent/linux/utils.py:108
```

As you have probably figured out, the first snippet shows the agent adding a
new IPv4 address `10.1.0.1` to an internal router `qr-864545b9-5f` port, and
the second snippet is where the agent sends gratuitous ARP packets advertising
the new IP address into the network to which the `qr-864545b9-5f` port is
attached to (this is achieved by calling the `arping` tool from iputils package
with the right arguments).

Let's have a look at each of the arguments passed into `arping` tool.

- The very first option is -A, and it's used to issue gratuitous (broadcast)
  ARP packets. Without the option, the tool would send unicast ARP REQUEST
  packets for the IP address, and would wait for a REPLY (the unicast mode may be
  useful when you need to check if there are any other hosts in the network
  carrying the same IP address, or to sanity check an existing IP-to-MAC
  mapping). The packets sent are of REPLY type. (If we would use -U instead, it
  would send REQUEST packets.)

- The next option is -I, and it specifies the interface to issue the packets
  on.

- The -c option defines the number of ARP packets to issue into the network.
  There is
  [always](https://github.com/iputils/iputils/blob/1ef0e4c86d358c8e217b90686394196412e184d2/arping.c#L382)
  a 1 second interval between the packets. Since we call it with -c 3, it
  issues three packets in two second time span.

- The next option is -w 4.5 and it means that we will wait for 4.5 seconds (or
  better, 4 seconds because the tool
  [doesn't recognize](https://github.com/iputils/iputils/blob/1ef0e4c86d358c8e217b90686394196412e184d2/arping.c#L1044)
  floating part of the argument) before exiting it. In general, the tool will
  exit after two seconds, but when the interface used to send packets is gone
  while the tool is running, it may block its execution since it will never be
  able to successfully send all three packets. The option guarantees that the
  thread running the tool will eventually make progress.

- The very last argument is the IP address to advertise. A single port may
  carry multiple IPv4 addresses, so it's crucial to define which of those
  addresses should be advertised.

#### HA routers

[HA support](https://docs.openstack.org/ocata/networking-guide/deploy-ovs-ha-vrrp.html)
is a relatively new addition to OpenStack Neutron routers. To use HA for
Neutron routers, one should configure Neutron API controller to expose
l3-ha API extension, at which point users are able to create highly available
routers.

For those routers, data plane is managed both by the L3 agent as well as the
`keepalived` daemon that the agent spawns for every HA router it manages. The
agent first prepares the router namespace, its ports, rules for NAT
translation; but then it falls back to the `keepalived` daemon which manages IP
addresses on ports. For this matter, the agent
[generates a configuration file](https://github.com/openstack/neutron/blob/dbc91498cc7d34332aabe0cdbf6b156ab167684c/neutron/agent/linux/keepalived.py#L279-L324)
listing all managed IP addresses and
[passes it](https://github.com/openstack/neutron/blob/dbc91498cc7d34332aabe0cdbf6b156ab167684c/neutron/agent/linux/keepalived.py#L432-L433)
into `keepalived`. The daemon then starts, negotiates with other
`keepalived` processes implementing the HA router who is going to be its
"master" (for this matter, VRRP is used), and if it's indeed "master", it
triggers state transition machinery, which, among other things, will add
managed IP addresses specified in the configuration file to appropriate router
ports. It will also send gratuitous ARP packets into the network to update
peers about the location of those IP addresses. If you then inspect your system
log, you may find the following messages there:

```
May  2 13:19:47 host-192-168-24-12 Keepalived[307081]: Starting Keepalived v1.2.13 (07/01,2016)
May  2 13:19:47 host-192-168-24-12 Keepalived[307082]: Starting VRRP child process, pid=307083
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: Netlink reflector reports IP 169.254.192.6 added
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: Netlink reflector reports IP fe80::f816:3eff:fe5f:d44b added
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: Registering Kernel netlink reflector
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: Registering Kernel netlink command channel
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: Registering gratuitous ARP shared channel
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: Opening file '/var/lib/neutron/ha_confs/b7fece4b-ea95-4eb6-b7b8-dc060325d1bc/keepalived.conf'.
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: Configuration is using : 64829 Bytes
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: Using LinkWatch kernel netlink reflector...
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) Entering BACKUP STATE
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) removing protocol Virtual Routes
May  2 13:19:47 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP sockpool: [ifindex(16), proto(112), unicast(0), fd(10,11)]
May  2 13:19:54 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) Transition to MASTER STATE
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) Entering MASTER STATE
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) setting protocol VIPs.
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) setting protocol E-VIPs.
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) setting protocol Virtual Routes
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) Sending gratuitous ARPs on ha-e09aa535-6f for 169.254.0.1
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) Sending gratuitous ARPs on qg-6cf347df-28 for 10.0.0.219
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) Sending gratuitous ARPs on qr-3ee577eb-4f for 10.100.0.1
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) Sending Unsolicited Neighbour Adverts on qr-3ee577eb-4f for fe80::f816:3eff:fe9a:c17
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: VRRP_Instance(VR_1) Sending Unsolicited Neighbour Adverts on qg-6cf347df-28 for fe80::f816:3eff:fec7:861a
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: Netlink reflector reports IP fe80::f816:3eff:fe9a:c17 added
May  2 13:19:56 host-192-168-24-12 Keepalived_vrrp[307083]: Netlink reflector reports IP fe80::f816:3eff:fec7:861a added
```

Here we can see `keepalived` transitioning to master state and immediately
issuing gratuitous updates after VIP addresses are set for managed interfaces.
(A careful reader will also notice that it also issues something called
`Unsolicited Neighbour Adverts` which is a similar mechanism for IPv6
addresses, but I won't go there.)

It would seem like it's good for the job. Sadly, the reality is uglier than one
could hope.

#### WTF#1: HA router reload doesn't issue gratuitous ARP packets

As we've learned during our testing of the HA feature, sometimes `keepalived`
forgot to send gratuitous ARP packets. It always happened when an existing
`keepalived` instance was asked to reload its configuration file because some
Neutron API operations triggered router updates that affected the file
contents. An example of an update could be e.g. adding a new floating IP
address to a port, or disassociating one. In this case, Neutron L3 agent would
generate a new configuration file and then
[send SIGHUP signal](https://github.com/openstack/neutron/blob/dbc91498cc7d34332aabe0cdbf6b156ab167684c/neutron/agent/linux/keepalived.py#L435)
to the running `keepalived` instance, hoping that it will catch the changes,
converge the data plane to latest configuration, and finally issue gratuitous
ARP updates. [It did not](https://bugzilla.redhat.com/show_bug.cgi?id=1391553).

Investigation, largely carried by John Schwarz, uncovered it was not an issue
with latest `keepalived` releases, but the one from RHEL7 repositories.
Bisecting releases, we've found out that the very first `keepalived` release
that was not exposing the buggy behavior was 1.3.20. Popular distributions
(RHEL7, Ubuntu Xenial) were still shipping older versions of the daemon (1.2.13
for RHEL7 and 1.2.19 for Xenial).

Though the issue was technically in `keepalived`, we needed to adopt OpenStack
to the buggy releases shipped with platforms we support. First considered
option was just fully restarting `keepalived`, which would correctly trigger
the gratuitous ARP machinery. The problem with this approach was that full
restart temporarily stops the VRRP thread that sends master health checks, and
with unfortunate timing, it sometimes results in an unnecessary "master" to
"backup" flip, operation that is both computationally costly as well as data
plane disruptive.

Since we couldn't just upgrade `keepalived`, it meant that Neutron L3 agent
would need to play some role in issuing gratuitous ARP packets, not relying on
the daemon to do the right job. For this matter,
[Neutron patch](https://review.opendev.org/c/400348/) was introduced. What the
patch does is it calls to `arping` tool whenever a new IPv4 address is added to
an interface managed by `keepalived`. A new address added indicates that VRRP
negotiation resulted in the locally running `keepalived` instance transitioning
to "master"; or it means a new floating IP address was added in the
configuration file just reloaded by the daemon. At this point it makes sense to
advertise the newly added addresses on the wire using gratuitous ARP, something
that in an ideal world `keepalived` would do for us.

We already had the
[neutron-keepalived-state-change](https://github.com/openstack/neutron/blob/dbc91498cc7d34332aabe0cdbf6b156ab167684c/neutron/agent/l3/keepalived_state_change.py)
helper daemon running inside HA router network namespaces
that
[monitors](https://github.com/openstack/neutron/blob/dbc91498cc7d34332aabe0cdbf6b156ab167684c/neutron/agent/l3/keepalived_state_change.py#L75)
router interfaces for new IP addresses
[to detect transitions](https://github.com/openstack/neutron/blob/dbc91498cc7d34332aabe0cdbf6b156ab167684c/neutron/agent/l3/keepalived_state_change.py#L77)
between `keepalived` states and then
[sends the information back](https://github.com/openstack/neutron/blob/dbc91498cc7d34332aabe0cdbf6b156ab167684c/neutron/agent/l3/keepalived_state_change.py#L79)
to neutron-server. To avoid introducing a new daemon just to issue gratuitous
ARP packets, we figured it's easier to
[reuse the existing one](https://github.com/openstack/neutron/blob/db4ea430df9ba1043cf15758922dd6612a5a1b2a/neutron/agent/l3/keepalived_state_change.py#L80-L87).

Of course, issuing gratuitous ARP packets from outside of
`keepalived` introduced some complications.

For one, the whole setup became slightly racy. For example, what happens when
`keepalived` decides to forfeit its mastership in the middle of
`neutron-keepalived-state-change` sending gratuitous ARP packets? Will we
continue sending those packets into the network even after `keepalived` removed
VIP addresses from its interfaces? Thanks to
[net.ipv4.ip_nonlocal_bind](https://github.com/torvalds/linux/blob/3ef2bc099d1cce09e2844467e2ced98e1a44609d/Documentation/networking/ip-sysctl.txt#L852-L855)
sysctl knob, it shouldn't be a concern. Its default value (0) means that
userspace tools (including `arping`) won't be able to send an ARP packet for an
IPv4 address that is not on the interface. If we hit the race, the worst that
could happen is that `arping` would hang, failing to send more gratuitous ARP
packets into the network, logging the "bind: Cannot assign requested address"
error message on its stderr. Since we set a hard time limit for the tool
execution (remember the -w 4.5 CLI arguments discussed above), it should be
fine. To stay on safe side, we would just
[set](https://github.com/openstack/neutron/blob/dbc91498cc7d34332aabe0cdbf6b156ab167684c/neutron/agent/l3/ha_router.py#L52-L53)
the sysctl knob inside each new router namespace to 0 to override whatever
custom value the platform may have for the setting.

There are still two complications with that though.

First, as it turned out, the `ip_nonlocal_bind` knob was set to 1 for DVR `fip`
namespaces,
[and for a reason](https://review.opendev.org/c/openstack/neutron/+/174129).
So we needed to make sure that it's set to `0` in all router namespaces except
`fip`.  Another issue that we surfaced was specific to RHEL7 kernel where the
`ip_nonlocal_bind` knob was not network namespace aware, so changing it in one
of namespaces affected all other routers. It was
[fixed](https://bugzilla.redhat.com/show_bug.cgi?id=1363661) in later RHEL7
kernels, and in the meantime, we could only hope that no one ever hosts both
DVR `fip` and HA `qrouter` namespaces on the same node, for they would clash.

#### WTF#2: `keepalived` forfeits mastership on multiple `SIGHUP`s sent in quick succession

Not completely related to gratuitous ARP, but since it's also about
`SIGHUP` handler, I figured I will mention this issue here too.

Some testing revealed that when multiple HA router updates arrived to Neutron
L3 agent in quick succession, `keepalived` sometimes forfeits its mastership,
flipping to "backup" with no apparent reason. Consequent network disruption
until a new `keepalived` "master" instance is elected included.

Further investigation, also led by John Schwarz, revealed that it always
happens when you would send multiple `SIGHUP` signals to `keepalived`,
irrespective to whether there were any changes to its configuration files.

It was clearly a [bug](https://bugs.launchpad.net/neutron/+bug/1647432) in the
daemon, but at this point we were used to work around its quirks, so it hasn't
taken a lot of time to come up with a special
[signal throttler](https://review.opendev.org/c/openstack/neutron/+/407099) for
`keepalived`. What it does is it introduces 3 second delays between consequent
SIGHUP signals sent to `keepalived` instances. Why 3 seconds? No particular
reason, except that it worked (anything below 2 seconds didn't), and it seemed
like a good idea to give `keepalived` a chance to send at least a single health
check VRRP message between reload requests, so we made it slightly longer than
the default health check interval
[which is 2 seconds](https://github.com/openstack/neutron/blob/dbc91498cc7d34332aabe0cdbf6b156ab167684c/neutron/conf/agent/l3/ha.py#L36)
for Neutron.

#### Reading logs

So how do I know that an HA router actually sent gratuitous ARP packets without
having access to a live machine? Let's say all I have is log files for Neutron
services.

For those packets that are sent by `keepalived` itself, it logs a message per
advertised IP address in syslog, as seen in a snippet provided earlier.

As for packets issued by `neutron-keepalived-state-change` daemon,
corresponding messages were originally logged in a file that was located in a
[directory](https://github.com/openstack/neutron/blob/dbc91498cc7d34332aabe0cdbf6b156ab167684c/neutron/agent/l3/ha_router.py#L123)
that also contained other files needed for the router, including
`keepalived` configuration and state files. The problem here is that once a
HA router is unscheduled from an L3 agent, it stops `keepalived` and cleans up
both the router namespace as well as all files used by the router, including
log files for `neutron-keepalived-state-change`. It means that after the router
is gone, you can't get your hands on the daemon log file. You are left in
darkness as to whether it even called to `arping`.

To facilitate post-cleanup debugging, in Pike release cycle we've made the
daemon
[to log to system log](https://review.opendev.org/c/openstack/neutron/+/453805/2/neutron/agent/l3/keepalived_state_change.py)
in addition to its own log file. With the patch, we can now see the
daemon messages in system journal, including those corresponding to `arping`
execution.

```
Apr 28 20:56:00 ubuntu-xenial-rax-ord-8650506 neutron-keepalived-state-change[20945]: 2017-04-28 20:56:00.338 20945 DEBUG neutron.agent.linux.utils [-] Running command: ['sudo', 'ip', 'netns', 'exec', 'qrouter-433765a8-f084-4fbd-9aea-447835c32b09@testceeee6ac', 'arping', '-A', '-I', 'qg-c317683_6ac', '-c', '3', '-w', '4.5', '10.0.0.215'] create_process /opt/stack/new/neutron/neutron/agent/linux/utils.py:92
Apr 28 20:56:00 ubuntu-xenial-rax-ord-8650506 sudo[24549]:    stack : TTY=unknown ; PWD=/ ; USER=root ; COMMAND=/sbin/ip netns exec qrouter-433765a8-f084-4fbd-9aea-447835c32b09@testceeee6ac arping -A -I qg-c317683_6ac -c 3 -w 4.5 10.0.0.215
Apr 28 20:56:02 ubuntu-xenial-rax-ord-8650506 neutron-keepalived-state-change[20945]: 2017-04-28 20:56:02.430 20945 DEBUG neutron.agent.linux.utils [-] Exit code: 0 execute /opt/stack/new/neutron/neutron/agent/linux/utils.py:153
```

Now whenever you have a doubt whether gratuitous ARP packets were sent by a
Neutron HA router, just inspect syslog. You should hopefully find there
relevant messages, either from `keepalived` itself or
from `neutron-keepalived-state-change` calling to `arping`.

---

You can find a continuation of this post, of sort,
[here](../garp-neutron-war-story).
