---
layout: page
title: About
permalink: /about/
---

I work on mobile core networks: 4G and 5G, roaming, and the signalling between them
(Diameter, GTP, SS7/MAP, NGAP). Most of my debugging starts from a packet capture.

Here I write up problems the way they arrive in practice: a ticket, the capture, what it
shows, the root cause, and the reply I would send.

Side projects:

- [capsight](https://github.com/Kx96xx/capsight): drop in a 4G/5G/SS7 capture, get a
  plain-English diagnosis of why a device couldn't attach, register or get data.
  `pip install capsight`
- [tshark-mcp](https://github.com/Kx96xx/tshark-mcp): Wireshark for AI assistants, a
  read-only MCP server that answers what went wrong in a pcap, with frame numbers.

Both are built with Claude Code; I design the protocol logic and rules and validate them
on real captures.
