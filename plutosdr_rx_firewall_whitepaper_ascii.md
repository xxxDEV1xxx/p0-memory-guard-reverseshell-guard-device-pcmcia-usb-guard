# Kernel-Level Packet Enforcement on PlutoSDR

## A Deterministic Linux RX-Handler Firewall Proof of Concept

**Author:** Christopher T. Williams\
**Platform:** PlutoSDR, ARMv7 hard-float, Linux
5.15.0-20952-ge14e351533f9-dirty\
**Status:** Engineering whitepaper draft\
**Date:** October 2026

------------------------------------------------------------------------

## Abstract

This paper documents a working proof of concept for selective packet
enforcement on a PlutoSDR running embedded Linux. A loadable kernel
module registers a receive handler on the USB network interface `usb0`,
inspects the received `sk_buff`, and consumes packets matching a narrow
IPv4 ICMP predicate. Non-matching traffic is returned to the normal
receive path.

During controlled tests recorded during development, ping to the Pluto
management address failed while SSH remained available. The principal
result is the demonstrated use of the kernel RX-handler boundary to stop
selected packets from continuing through the ordinary receive path. This
is a proof of concept, not a production-hardened firewall.

## 1. Scope and Contributions

The module registers `p0_rx_handler` on `usb0` using
`netdev_rx_handler_register()`. Its current rule consumes IPv4 packets
whose IPv4 Protocol field identifies ICMP; all other packets are passed
onward.

This firewall is separate from the P0 scanner. The scanner is outside
the implementation scope of this paper.

The work documents: - Discovery of the actual packet layout on the
target receive path. - A deterministic IPv4 ICMP classifier. - Selective
packet consumption while preserving SSH in the recorded test. - Kernel
identity, configuration, source, and integrity metadata for recreation.

## 2. Platform and Constraints

The tested device is a PlutoSDR based on ARMv7 hard-float, running Linux
`5.15.0-20952-ge14e351533f9-dirty`, build #10. The running configuration
reports `CONFIG_MODULES=y` and `CONFIG_MODVERSIONS=y`.

The implementation uses the kernel RX-handler mechanism. It does not
depend on iptables, Netfilter, tc, XDP, or a userspace firewall daemon.

The recorded Linux source commit is
`e14e351533f934047ba0473e836e561682ec67fe`. The firmware source lineage
is plutosdr-fw v0.38.

The original build lacked the exact matching `Module.symvers`, so
modpost emitted symbol-version warnings. The module nevertheless loaded
on the tested kernel and enforcement was observed. This is a
reproducibility caveat, not a recommended release-build practice.

## 3. Architecture and Packet-Layout Discovery

The receive path on the tested `usb0` interface was observed to present
the IPv4 network header at `skb->data`, rather than an Ethernet header.
Diagnostic bytes began with values such as `45 10 ...`, consistent with
an IPv4 header.

An initial classifier assumed Ethernet-header offsets. That classifier
did not match as intended, and ping continued to succeed. The
implementation was corrected to inspect the IPv4 header at the observed
offset. This result emphasizes that packet offsets must be established
on the actual receive path.

### ASCII Architecture Diagram

``` text
             PlutoSDR Linux kernel
  +------------------------------------------------------+
  |                                                      |
  |  +-------------------+                               |
  |  | USB interface     |                               |
  |  | usb0              |                               |
  |  +---------+---------+                               |
  |            | incoming sk_buff                        |
  |            v                                         |
  |  +---------------------------+                       |
  |  | Registered RX handler     |                       |
  |  | p0_rxhandler              |                       |
  |  | netdev_rx_handler_register|                       |
  |  +-------------+-------------+                       |
  |                |                                     |
  |                v                                     |
  |  +---------------------------+                       |
  |  | Deterministic classifier  |                       |
  |  | skb protocol is IPv4?     |                       |
  |  | IPv4 version nibble = 4?  |                       |
  |  | skb->data[9] = ICMP?      |                       |
  |  +-------------+-------------+                       |
  |                |                                     |
  |          +-----+-----+                               |
  |          |           |                               |
  |        MATCH      NO MATCH                           |
  |          |           |                               |
  |          v           v                               |
  |  +---------------+  +-----------------------------+  |
  |  | CONSUMED      |  | PASS                        |  |
  |  | Drop from     |  | Continue normal Linux      |  |
  |  | normal RX path|  | receive processing         |  |
  |  +-------+-------+  +--------------+--------------+  |
  |          |                         |                 |
  +----------|-------------------------|-----------------+
             |                         |
             v                         v
       ICMP / ping blocked       TCP / SSH preserved
       (observed test)           (observed test)

  Separate component:
  +-------------------------------+
  | P0 scanner                    |
  | Independent module/component  |
  | Not part of firewall path     |
  +-------------------------------+
```

### Observed Classifier

``` c
if (skb->len >= 10 &&
    skb->protocol == htons(ETH_P_IP) &&
    (skb->data[0] & 0xf0) == 0x40 &&
    skb->data[9] == IPPROTO_ICMP) {
    return RX_HANDLER_CONSUMED;
}
return RX_HANDLER_PASS;
```

The classifier checks the skb protocol, IPv4 version nibble, and IPv4
Protocol field. The length check ensures at least 10 bytes are available
before reading byte 9.

This predicate matches IPv4 ICMP broadly; it does not distinguish ICMP
message type or code. The project label "P0 signature" does not imply
that this predicate uniquely identifies malicious traffic.

## 4. Enforcement Semantics

`RX_HANDLER_CONSUMED` indicates that the packet is consumed at the
RX-handler boundary and does not continue through the ordinary receive
path. `RX_HANDLER_PASS` allows processing to continue.

The handler registers on `usb0` under the RTNL lock. The module's exit
path unregisters the handler under the RTNL lock and releases the device
reference.

This is kernel RX-path enforcement, not merely observation through an
AF_PACKET socket. AF_PACKET was useful during discovery, but dropping an
AF_PACKET copy alone does not prevent the original packet from being
processed by the normal kernel stack.

## 5. Experimental Method and Results

The recorded tests showed:

-   Before enforcement, ping to the target succeeded.
-   With the narrow ICMP-consuming module loaded, a controlled ping test
    reported one transmitted packet and zero received packets (100%
    loss).
-   An SSH test to the same management address returned `SSH_PASS`.

An earlier rule that consumed all IPv4 packets interrupted SSH
management. After narrowing the rule to IPv4 ICMP, ping was blocked
while SSH remained available.

These are functional spot tests, not a comprehensive security
evaluation. No throughput, latency, CPU, memory, stress, or
adversarial-header benchmarks are claimed.

## 6. Build and Reproducibility

The recreation bundle contains: - `p0_rxhandler.ko` - `p0_rxhandler.c` -
`Makefile` - `kernel.config` - `uname.txt` - `proc-version.txt` -
`cmdline.txt` - `README.txt` - `SHA256SUMS`

Known-good module SHA-256:

``` text
9c85afaa9549ab03d7f3c49ddeae44fb00e23b47be964a47998fcdb792c033a6
```

The original build used the ARM hard-float cross-toolchain and the
matching Pluto Linux source tree:

``` sh
LD_LIBRARY_PATH=/data/armhf-toolchain/usr/lib/x86_64-linux-gnu \
make ARCH=arm \
  CROSS_COMPILE=/data/armhf-toolchain/usr/bin/arm-linux-gnueabihf- \
  EXTRA_CFLAGS="-march=armv7-a" \
  -C /data/pluto-linux \
  M=/data/pluto-linux/p0_rxhandler \
  modules
```

Because `CONFIG_MODVERSIONS=y`, a high-confidence recreation should use
the exact matching symbol-version data and verify module compatibility.
Do not treat the original missing-`Module.symvers` build as ideal
release practice.

Deployment checklist: 1. Confirm target kernel identity and build. 2.
Verify the module hash. 3. Load the module on a recoverable test device.
4. Inspect kernel logs for successful registration. 5. Test ICMP
blocking. 6. Test SSH availability. 7. Unload the module and confirm
removal.

The module is not configured for automatic loading.

## 7. Safety, Limitations, and Threat Model

This prototype is interface-specific and depends on the kernel version,
driver path, skb layout, and registration semantics. It is not a general
policy engine. It does not provide IPv6 filtering, connection tracking,
configurable rules, persistent policy, automatic rollback, or an
out-of-band recovery channel.

The classifier relies on the observed IPv4 header location and a minimal
set of checks. A more robust implementation should use appropriate
kernel header-access patterns and account for skb linearity and header
offsets when expanding to other paths.

The demonstrated behavior applies to the tested `usb0` receive path. It
should not be described as a universal filter for all interfaces,
locally generated traffic, alternate receive paths, or all ingress and
egress traffic.

**Safety warning:** Do not broaden the consume rule without a recovery
plan. Consuming all IPv4 traffic previously interrupted SSH management.

The compiled module declares `MODULE_LICENSE("GPL")`. The source and
documentation bundle also carries the custom permission and warranty
notice supplied by the author. These notices should be reviewed together
before redistribution.

## 8. Future Work

Priorities for engineering maturity: 1. Build against exact kernel
artifacts and symbol-version data. 2. Add an out-of-band recovery path
before testing broader rules. 3. Add structured counters and
configurable rules. 4. Test malformed and truncated packets, IPv4
options, fragments, non-linear sk_buffs, and alternate interfaces. 5.
Measure throughput, latency, CPU load, and packet loss. 6. Test
lifecycle, error paths, and repeated load/unload. 7. Publish a
reproducible automated test harness.

The P0 scanner should be evaluated as a separate component. It is not
integrated into the current firewall proof of concept.

## Appendix A. Key Implementation Details

-   Interface: `usb0`
-   Kernel primitive: `netdev_rx_handler_register()`
-   Handler result for matching ICMP: `RX_HANDLER_CONSUMED`
-   Handler result for other traffic: `RX_HANDLER_PASS`
-   Diagnostic logging: first 20 packets, including a 40-byte prefix
    when available
-   Removal: `rmmod p0_rxhandler`

## Appendix B. Author Notice

Copyright (c) 2026 Christopher T. Williams.

Permission is granted to use, copy, modify, and distribute this software
for research, testing, and security-analysis purposes, provided that the
copyright notice and this permission notice are retained.

THIS SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, OR NON-INFRINGEMENT.
THE AUTHOR SHALL NOT BE LIABLE FOR ANY CLAIM, DAMAGES, OR OTHER
LIABILITY ARISING FROM THE USE OF THIS SOFTWARE.

------------------------------------------------------------------------

Document status: Initial engineering whitepaper draft prepared from the
recorded implementation and tests. Update with measured performance data
and a reproducible test log before external publication.
