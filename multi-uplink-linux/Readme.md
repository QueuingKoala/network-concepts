Multi-uplink routing for Linux endpoints
========================================

This concept directory discusses how to properly configure a Linux-based system
to handle connections from multiple independent uplinks.

This topic is not related to link aggregation or dynamic routing; you will not
be able to "combine" the bandwidth from both uplinks for a single stream.
However, you will be able to accept connections on either link, and the reply
traffic will be correctly bound to the interface.

Overview
--------

With independent uplinks, you must keep the replies you send back going out the
same interface and with the same source IP as the original sender used to reach
your endpoint. ISPs drop traffic sourced from IP space that is not theirs for
end-customers.

Normally the kernel identifies all traffic that is not on your link (or defined
by another route) and sends it to your default gateway. This is a problem if the
default gateway is on link#1, but the traffic arrived from link#2, because the
reply will go out the wrong interface, using the wrong IP.

Proper Integration of these steps
------------------

First, a quick note about integrating the examples you see below.

While some of the examples here contain stand-alone commands, a proper
configuration should integrate these into the canonical utilities for loading
rules or routing policies. It is almost always the wrong approach to call
"scripts" to do this.

### Integrating Netfilter rules
  Most distros provide an init wrapper around the `iptables-restore` program,
  which loads Netfilter rules in an atomic and one-shot fashion. You should
  integrate the sample rules here into your *own* complete ruleset.

### How to read the Netfilter rule examples
  If you are not familiar with the `iptables-save(8)` format, you may find it
  fairly self-explanatory if you're already familiar with Netfilter. Check out
  my `netfilter-samples` project for further docs and a BNF description for more
  details.

  Since I discourage use of scripts to load Netfilter rules (kittens die every
  time this is done) my rule examples are in `iptables-save(8)` format.

  This means the following command:

    iptables -t filter -A INPUT -p tcp --dport 22 -j ACCEPT

  would be written like this:
  
    # table: filter
    -A INPUT -p tcp --dport 22 -j ACCEPT

  Note that in `iptables-save(8)` format, the table is defined before rules in
  it are, so my individual rule examples merely reference the table they must be
  included under. It is up to you to integrate these rules in the correct place
  in your complete Netfilter ruleset.

### Integrating routing tables and rules
  Likewise, distro networking components should have features allowing for
  routes and rules to be added as the system starts up; this facility is usually
  provided along with that to set your addressing, gateways, and DNS.

Procedure Overview
------------------

The procedure to do this can be broken down into a series of steps.

1. Mark sessions arriving on each uplink for later identification.
2. Copy the session-mark to the per-packet mark for routing integration.
3. Create secondary routing tables for egress to each uplink.
4. Create routing rules to send marked traffic to the right routing table.
5. Set reverse-path filtering to 'loose' or 'disabled' mode.

We'll cover each of these steps a little further below.

### Some notes on the method used here
  The flow of a session contains inbound data arriving from a source, bound for
  the receiving system, which is presumably your Linux host.

  In order to correlate that inbound traffic with the associated reply traffic,
  the Netfilter connection tracking system (or *conntrack* ) is used. This also
  has the useful effect of keeping *related* traffic, such as ICMP errors or
  protocol-specific secondary sockets (like FTP) flowing properly. Conntrack is
  powerful, and is used as the core backend component for this setup.

  The remainder of the procedure deals with translating the higher-level session
  info conntrack has to the lower-level routing decision. Each packet is routed
  *without* knowledge to fact that it is part of a larger stream of data.

### Terminology list
  A quick list of terms is expressed here:

  * `ctmark` (or `connmark`) is a mark that is applied to an entire session and
    managed by the conntrack subsystem of Netfilter.

  * `fwmark` (or `nfmark`) is the Netfilter mark that applies per-packet only.

### Assumptions in this example
  1. It is assumed you have the required tools available. This includes the
     `iproute2` package, kernel support for Netfilter's mark and connmark
     features, and a userland that supports manipulation over your routing
     tables and Netfilter state.
  
  2. Addressing and interfaces in these examples:
    
    * eth1 is defined as uplink #1, with an IP of 203.0.113.8/24
    * eth1 has a gateway of 203.0.113.1
    * eth2 is defined as uplink #2, with an IP of 192.0.2.31/24
    * eth2 has a gateway of 192.0.2.1

Detailed Procedure
------------------

### Mark inbound sessions
  In order to leverage conntrack, a unique ctmark is applied to all traffic from
  the uplink interfaces. While it is possible to do this only on the interface
  that does not contain the default route, this example does it for both
  interfaces for clarity.

  Some performance tweaks are possible to avoid re-applying the ctmark for
  already-marked sessions, but there are potential pitfalls. A later section may
  cover this, but here we simply (re)-mark all traffic.

  Let's give traffic from eth1 a ctmark of 1, and eth2 a ctmark of 2. Any other
  value is possible as well, including use of mark masks; consult the Netfilter
  docs for details. We'll stick to standard marks though.

  The following rules on the mangle table will do this:

    # table: mangle
    -A PREROUTING -i eth1 -j CONNMARK --set-mark 0x1
    -A PREROUTING -i eth2 -j CONNMARK --set-mark 0x2

### Restore ctmark to fwmark
  In order to use the ctmark in routing rules, all outbound traffic must be
  marked with the packet-specific fwmark. Routing rules cannot see the ctmark,
  only the fwmark.

  The following rules on the mangle table handle the mark restoring:

    # table: mangle
    -A OUTPUT -m connmark --mark 0x1 -j MARK --restore-mark
    -A OUTPUT -m connmark --mark 0x2 -j MARK --restore-mark

### Secondary routing tables
  To match return traffic to the same interface and IP it arrived on, separate
  routing tables must be created. Here we create both the default route and a
  link-local route for each table.

  As with the marks earlier, the table numbers are completely arbitrary, and
  could even use named tables instead as defined by your site needs at
  `/etc/iproute2/rt_tables`.

  The following commands create these tables (though remember to integrate these
  properly as noted earlier.)

    ip route add default via 203.0.113.1 dev eth1 table 101
    ip route add 203.0.113.0/24 dev eth1 scope link table 101
    ip route add default via 192.0.2.1 dev eth2 table 102
    ip route add 192.0.2.0/24 dev eth2 scope link table 102

### Routing rules
  In order to send marked packets to the right routing table for lookup, rules
  must be created that direct packets to one of the new tables. This is called
  Policy Routing since we are routing on a 'policy' that we define: namely the
  packet marks that in turn reference the receiving interface.

  The routing rules are set up as follows (again: please integrate these
  properly.)

    ip rule fwmark 0x1 table 101
    ip rule fwmark 0x2 table 102

### Dealing with reverse-path filtering
  A feature of the Linux kernel is known as reverse-path filtering, or the
  sysctl node `rp_filter`. This option is incompatible with certain complex
  routing setups, including this policy routing setup.

  Full documentation on rp filtering can be found in the linux kernel source at
  `~linux/Documentation/networking/ip-sysctl.txt` (search for `rp_filter`.)

  Note that the `rp_filter` sysctl node has 3 types of places it can be set at:

  * `net.ipv4.conf.all.rp_filter`
  * `net.ipv4.conf.IFACE.rp_filter` (where IFACE is the egress interface name)
  * `net.ipv4.conf.default.rp_filter` (which gets applied to all new interfaces)

  The *highest* value between the `all` and `IFACE` values are used. It is
  **required** for this policy routing setup that `loose` (value 2) or
  `disabled` (value 0) is set.

  While 0 is the kernel default, many distros today set `rp_filter` to 1. This
  means that setting loose mode with a value of 2 to in the `all` node parent is
  likely easier than setting all the values to 0 such that no `1` value could
  mess things up.

  Vary the approach depending on your local needs.
