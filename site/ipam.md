---
title: Automatic IP Address Management
layout: default
---

# Automatic IP Address Management

Weave can automatically assign containers an IP address that is unique
across the network.

 * [Usage](#usage)
 * [Initialisation](#initialisation)
 * [Choosing an allocation range](#range)
 * [Automatic allocation across multiple subnets](#subnets)
 * [Mixing automatic and manual allocation](#manual)
 * [Stopping and removing peers](#stop)
 * [Troubleshooting](#troubleshooting)

## <a name="usage"></a>Usage

Containers are automatically allocated an IP address when none is
specified when the container is started, e.g.

    host1# C=$(weave run -ti ubuntu)

You can see which address was allocated with
[`weave ps`](troubleshooting.html#list-attached-containers):

    host1# weave ps $C
    a7aff7249393 7a:51:d1:09:21:78 10.128.0.1/10

Weave detects when a container has exited and releases its
automatically allocated addresses so they can be re-used.

Automatic IP address assignment is available for the `run`, `start`,
`attach`, `detach`, `expose`, `hide` and `launch-dns` commands.

## <a name="initialisation"></a>Initialisation

Just once, when the first automatic IP address allocation is requested
in the whole network, weave needs a majority of peers to be present in
order to avoid formation of isolated groups, which could lead to
inconsistency, i.e. the same IP address being allocated to two
different containers. Therefore, you must either supply the list of
all peers in the network to `weave launch` or add the `-initpeercount`
flag to specify how many peers there will be.  It isn't a problem to
over-estimate by a bit, but if you supply a number that is too small
then multiple independent groups may form.

To illustrate, suppose you have three hosts, accessible to each other
as `$HOST1`, `$HOST2` and `$HOST3`. You can start weave on those three
hosts with these three commands:

    host1$ weave launch $HOST2 $HOST3

    host2$ weave launch $HOST1 $HOST3

    host3$ weave launch $HOST1 $HOST2

Or, if it is not convenient to name all the other hosts at launch
time, you can give the number of peers like this:

    host1$ weave launch -initpeercount 3

    host2$ weave launch -initpeercount 3 $HOST3

    host3$ weave launch -initpeercount 3 $HOST2

## <a name="range"></a>Choosing an allocation range

By default, weave will allocate IP addresses in the 10.128.0.0/10
range. This can be overridden with the `-iprange` option, e.g.

    host1# weave launch -iprange 10.2.0.0/16

and must be the same on every host.

The range parameter is written in
[CIDR notation](http://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) -
in this example "/16" means the first 16 bits of the address form the
network address and the allocator is to allocate container addresses
that all start 10.2. The ".0" and ".-1" addresses in a subnet are not
used, as required by
[RFC 1122](https://tools.ietf.org/html/rfc1122#page-29).

Weave shares the IP address range across all peers, dynamically
according to their needs.  If a group of peers becomes isolated from
the rest (a partition), they can continue to work with the address
ranges they had before isolation, and can subsequently be re-connected
to the rest of the network without any conflicts arising.

Once you have given a range of addresses to the IP allocator, you must
not use any addresses in that range for anything else.  If, in our
example, you subsequently executed `weave run 10.2.3.1/24 -ti ubuntu`,
you run the risk that the IP allocator will assign the same address to
another container, which will make network traffic delivery
intermittent or non-existent for the containers that share the same IP
address.

## <a name="subnets"></a>Automatic allocation across multiple subnets

When
[running containers on different subnets](features.html#application-isolation),
you may wish to request the allocation of an address from a particular
subnet. This is done by specifying the subnet with `net:<subnet>`, in
CIDR notation, e.g.

    host1# C=$(weave run net:10.2.7.0/24 -ti ubuntu)

You can ask for multiple addresses in different subnets and add in
manually-assigned addresses (outside the automatic allocation range),
for instance:

    host1# C=$(weave run net:10.2.7.0/24 net:10.2.8.0/24 ip:10.3.9.1/24 -ti ubuntu)

When working with multiple subnets in this way, it is usually
desirable to constrain the default subnet - i.e. the one chosen by the
allocator when no subnet is supplied - so that it does not overlap
with others. One can specify that with `-ipsubnet`:

    host1# weave launch -iprange 10.2.0.0/16 -ipsubnet 10.2.3.0/24

`-iprange` should cover the entire range that you will ever use for
allocation, and `-ipsubnet` is the subnet that will be used when you
don't explicitly specify one.

When specifying addresses, the default subnet can be denoted
symbolically with `net:default`.

## <a name="manual"></a>Mixing automatic and manual allocation

If you want to start containers with a mixture of
automatically-allocated addresses and manually-chosen addresses, *and
have the containers communicate with each other*, you can choose a
`-iprange` that is smaller than `-ipsubnet`, For example, if you
launch weave with:

    host1# weave launch -iprange 10.9.0.0/17 -ipsubnet 10.9.0.0/16

then you can run all containers in the 10.9.0.0/16 subnet, with
automatic allocation using the lower half, leaving the upper half free
for manual allocation.

## <a name="stop"></a>Stopping and removing peers

You may wish to `weave stop` and re-launch to change some config or to
upgrade to a new version; provided the underlying protocol hasn't
changed it will pick up where it left off and learn from peers in the
network which address ranges it was previously using. If, however, you
run `weave reset` this will remove the peer from the network so
if Weave is run again on that node it will start from scratch.

For failed peers, the `weave rmpeer` command can be used to
permanently remove the ranges allocated to said peer.  This will allow
other peers to allocate IPs in the ranges previously owner by the rm'd
peer, and as such should be used with extreme caution - if the rm'd
peer had transferred some range of IP addresses to another peer but
this is not known to the whole network, or if it later rejoins
the Weave network, the same IP address may be allocated twice.

Assuming we had started the three peers in the example earlier, and
host3 has caught fire, we can go to one of the other hosts and run:

    host1$ weave rmpeer host3

Weave will take all the IP address ranges owned by host3 and transfer
them to be owned by host1. The name "host3" is resolved via the
'nickname' feature of weave, which defaults to the local host
name. Alternatively, one can supply a peer name as shown in `weave
status`.

## <a name="troubleshooting"></a>Troubleshooting

The command

    weave status

reports on the current status of the weave router and IP allocator:

````
weave router git-8f675f15c0b5
...
Allocator universe 10.2.0.0/16
Owned Ranges:
  10.2.1.1 -> 96:e9:e2:2e:2d:bc (host1) (v3)
  10.2.1.128 -> ea:84:25:9b:31:2e (host2) (v3)
  10.2.1.192 -> ea:6c:21:09:cf:f0 (host3) (v9)
Allocator default subnet: 10.2.1.0/24
````

The first section covers the router; see the [troubleshooting
guide](troubleshooting.html#status-report) for full details.

The 'Allocator' section, which is only present if weave has been
started with the `-iprange` option, summarises the overall position and
lists which address ranges have been assigned to which peer. Each
range begins at the address shown and ends just before the next
address, or wraps around at the end of the subnet. The 'v' number
denotes how many times that entry has been updated.

The 'Free IPs' information may be out of date with respect to changes
happening elsewhere in the network.
