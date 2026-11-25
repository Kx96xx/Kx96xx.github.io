---
layout: post
title: "5G connected, but no data: reading the capture"
description: "A device has a 5G data session but nothing loads. Fifteen packets show why, and the fix is one firewall rule."
tags: [5g, gtp, wireshark, troubleshooting]
---

> **The ticket:** "Our test device shows 5G with full bars and a data session, but every
> app times out. Ping to 8.8.8.8 fails. We already restarted the phone."

This is one of the most common tickets in mobile networks, and "restart the phone" never
fixes this version of it. The good news: a short capture on the core side answers it in
minutes. Here is how I read one.

The capture is public, from Abdelrahman Elsayed's
[5G SA core protocol lab](https://github.com/abdelrahman-fawaz18/5g-sa-core-protocol-lab)
(Open5GS + UERANSIM, MIT licence), so you can follow along:
`captures/fault-injection/missing-n6-nat/fault.pcap`.

## First question: is the problem before or after the session?

"No data" has two very different families of causes:

1. **The device never got a working session.** Registration or PDU session setup failed,
   so there is no tunnel. You would look at NGAP and NAS (registration, PDU session
   establishment rejects).
2. **The session exists, but packets get lost on the way.** The control plane is fine;
   the user plane is broken.

This capture contains only user-plane traffic, and that already answers the first
question: the device's packets arrive inside a GTP-U tunnel (TEID `0xa002`) and the UPF
unwraps them. A tunnel only exists once the session was set up. (The lab's own log
confirms registration and PDU session setup succeeded.) So we are in family 2.

## Follow one ping through the core

```
$ tshark -r fault.pcap
 1  0.000000  10.45.0.5 → 8.8.8.8  GTP/ICMP  Echo (ping) request  seq=1  ttl=64
 2  0.000129  10.45.0.5 → 8.8.8.8  ICMP      Echo (ping) request  seq=1  ttl=64
 3  0.000169  10.45.0.5 → 8.8.8.8  ICMP      Echo (ping) request  seq=1  ttl=63
 4  1.013913  10.45.0.5 → 8.8.8.8  GTP/ICMP  Echo (ping) request  seq=2  ttl=64
 ...
15  4.086081  10.45.0.5 → 8.8.8.8  ICMP      Echo (ping) request  seq=5  ttl=63
```

Each ping shows up three times, because the capture was taken on all interfaces of the
core host:

| Frame | Where | What it tells us |
|---|---|---|
| 1 | Inside the GTP-U tunnel, from the radio side (N3) | The device sent it; the radio delivered it to the UPF |
| 2 | After the UPF removed the tunnel header | The UPF accepted it for this session |
| 3 | Leaving towards the internet (N6), TTL 64 → 63 | The host routed it out |

So far everything works: all five pings take all three hops. Then there is nothing:
**no reply anywhere in the capture**, not outside the tunnel and not inside it.

## The clue is in frame 3

Look at the source address of the packet leaving for the internet: `10.45.0.5`. That is
the device's own address from the UE pool, a private (RFC 1918) address.

A packet with a private source can reach 8.8.8.8, but the reply is addressed to
`10.45.0.5`, and nobody on the internet has a route to that. The reply is dropped
somewhere upstream. From the device's side, it looks exactly like "no data".

A working network rewrites that source on the way out (source NAT, "masquerade") to an
address the internet can route back to, then reverses the translation for the reply.

The lab also published a capture after the fix, and the difference is one field:

```
$ tshark -r recovery.pcap
 1  10.45.0.5     → 8.8.8.8       GTP/ICMP  Echo (ping) request         (tunnel)
 2  10.45.0.5     → 8.8.8.8       ICMP      Echo (ping) request         (after UPF)
 3  192.168.0.32  → 8.8.8.8       ICMP      Echo (ping) request         (NAT applied)
 4  8.8.8.8       → 192.168.0.32  ICMP      Echo (ping) reply
 5  8.8.8.8       → 10.45.0.5     ICMP      Echo (ping) reply           (translated back)
 6  8.8.8.8       → 10.45.0.5     GTP/ICMP  Echo (ping) reply           (into the tunnel)
```

Frame 3 now leaves as `192.168.0.32`, the reply comes back to it, gets translated back to
`10.45.0.5` and goes down the tunnel to the device. Five out of five.

## Root cause

The source NAT rule for the UE pool on the UPF host was missing:

```
-A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
```

Forwarding was on, the forwarding rules were there, the session was up. Only the
translation was missing, which is why everything *looked* healthy from the control plane.
(In a production core the equivalent is the NAT or routing on the SGi/N6 side: a
carrier-grade NAT, or a return route for the UE pool if the addresses are meant to be
routable.)

## The reply I would send

> Thanks for the capture. Your device's data session is up and its traffic reaches our
> core: we can see each ping arrive through the tunnel and leave towards the internet.
> The replies never come back because the packets leave with the device's private address
> (10.45.0.5), which the internet cannot route to. The address translation for the device
> pool is missing on the gateway's internet-facing side.
>
> Nothing is wrong with the device or the SIM, so there is no need to restart or
> re-provision anything. Once the translation rule is restored, data works on the
> existing session; in the lab capture it recovered without even reconnecting.

## Doing this in one command

I built [capsight](https://github.com/Kx96xx/capsight) to automate exactly this kind of
reading. On the same file:

```
$ pip install capsight
$ capsight analyze fault.pcap

FAILURE   User plane one-way: traffic from 10.45.0.5 to 8.8.8.8 gets no reply.
WHY       5 GTP-U packets from 10.45.0.5 to 8.8.8.8 went into the tunnel and none came
          back (no reply inside or outside the tunnel). The same packets were seen leaving
          towards the data network (10x) still using the device's address 10.45.0.5 as
          source.
FIX       10.45.0.5 is a private address, so replies from 8.8.8.8 cannot route back. Add
          NAT/masquerade for the UE address pool on the UPF's N6/SGi side, or a return
          route for the pool.
```

And on `recovery.pcap`: `No known failure patterns found.`

## Takeaways

- Split "no data" first: **no session** (control plane) or **session but packets lost**
  (user plane). A GTP-U tunnel carrying the device's packets means you are in the second
  case.
- Follow one packet across every hop and find the last place it was seen. Here it was seen
  leaving for the internet, so the core did its job, up to one detail.
- Read the source address on the internet side. A private source leaving for the internet
  means missing NAT, or a missing return route.
- A capture of only one direction cannot prove a reply never came. Here the capture covers
  all interfaces, and the recovery capture shows replies are captured when they exist.
