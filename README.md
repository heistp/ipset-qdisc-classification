# cake-custom-isolation

Using ipsets, tc-flow and eBPF to do scalable, custom host and flow isolation
with the Cake qdisc

## Introduction

Here we show a way to customize
[Cake](https://www.bufferbloat.net/projects/codel/wiki/Cake/)'s
host and flow isolation in a scalable way. Cake already has the ability to
override its flow classification using tc filters (see the
[cake man page](https://man7.org/linux/man-pages/man8/tc-cake.8.html#OVERRIDING_CLASSIFICATION_WITH_TC_FILTERS)),
but they are searched linearly. ISPs may have many subscribers, each with one or
more IP/MAC addresses. We want fairness at two levels, between the ISP's
subscribers, and the flows for each subscriber, without having to do a linear
search through all subscribers.

[IP sets](https://ipset.netfilter.org/) provide highly scalable sets of various
types, like IP addresses, MAC addresses, subnets, etc. Each of these set types
supports the skbinfo extension, which allows setting the firewall mark or skb
priority individually for each ipset entry. However, Cake expects to have tc
filters as children of the Cake qdisc that set the major and minor class ID to
override the host and flow hash. We can't do this with ipsets, at least
directly.

Instead, we use use ipsets to set either the priority field or a firewall mark,
then use either [tc-flow](https://man7.org/linux/man-pages/man8/tc-flow.8.html)
or [eBPF](https://man7.org/linux/man-pages/man8/tc-bpf.8.html) in a child filter
of the Cake qdisc to map one of these to the classid. Because tc-flow can only
map to the minor classid, it can only override the flow classification,
resulting in one queue per subscriber. To override the host and/or flow
classification, we use a simple eBPF classifier.

## Installation

Prerequisites: ipset, iperf3 (to run `cakeiso.sh` test script), clang and
libbpf (for eBPF compiliation)

It's usually not necessary to run `make` to rebuild the eBPF classifiers,
because the executable format is portable. The pre-built objects in this repo
have been tested on amd64 with kernel 5.10, and arm64 with kernel 5.4. If the
eBPF examples don't work, try recompiling with `make`.

## Running cakeiso.sh

`./cakeiso.sh` (must be run as root, see [cakeiso.md](cakeiso.md) for sample output)

This is a standalone test script that sets up a netns environment with three
namespaces, a client, a middlebox and a server. Three iperf3 servers are started
on the server. We test three flows from two different "subscribers", two TCP
flows and one unresponsive UDP flow, to show different host and flow isolation
scenarios.

Seven different scenarios show the key concepts with tc-flow and eBPF. Three
scenarios show how tc-flow can be used to override the flow classification, but
cannot be used to override the host classification. Four scenarios use the eBPF
classifiers `mark_to_classid.o` and `priority_to_classid.o` to override the host
and/or flow classification with firewall marks or the skb priority,
respectively.

**Note:** Ignore any warnings like the below, which happen when running eBPF
in network namespaces, and do not seem to affect the results:

```
Continuing without mounted eBPF fs. Too old kernel?
```

## More Info

### Beyond IP Addresses

IP sets can also match by MAC address, IP/port, subnet, etc., supporting many
thousands of elements with little runtime overhead.

### CAKE_QUEUES

Cake is only compiled with **1024 queues** by default (CAKE_QUEUES define).
Thus, the major and minor classid from the filter must be <= 1024, or else the
classification will not be overridden. CAKE_QUEUES may need to be increased,
depending on the requirements. Without changing it, the approach laid out here
is suitable for up to 1024 users, and practically, far fewer total concurrent
flows.

### Overriding the Tin

**Heads Up:** Cake also supports overriding the tin (priority level) using the
skb priority field, which may conflict with our method of putting the classid
into priority. If the major number of the priority field matches the qdisc
handle of the Cake instance, the minor number of the priority field will be used
to override the Cake tin, which may not be the intent. Either avoid conflicts
with the major number (Cake's handle can always be changed), or use marks
instead. The priority field to classid mapping is provided in case the mark
field is already in use for other purposes.

### Overriding both Isolation and Tin

It's possible for an ipset entry to override *both* the priority field and the
firewall mark. Thus, it's possible to customize both the isolation and the tin
at the same time, e.g. if Cake's qdisc handle is 1, we're using `diffserv4` and
we want to isolate an IP's traffic as host 1 and put its traffic into the video
tin, something like this should work for the ipset entry, in combination with
the `mark_to_classid.o` eBPF filter:

```
ipset add subscribers 10.7.1.2 skbmark 0x10000 skbprio 1:3
```
