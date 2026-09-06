---
title: "Accelerated Bottleneck Bandwidth Adaptation (ABBA)"
abbrev: "abba"
category: std
docname: draft-kazuho-ccwg-abba-latest
workgroup: "Congestion Control Working Group"
ipr: trust200902
keyword: internet-draft
pi: [toc, sortrefs, symrefs]
stand_alone: yes
author:
 -
    fullname:
      :: 奥 一穂
      ascii: Kazuho Oku
    org: Fastly
    email: kazuhooku@gmail.com

normative:

informative:

...

--- abstract

TBD.


--- middle

# Introduction {#intro}

CUBIC {{!CUBIC=RFC9438}} attains the bottleneck bandwidth by way of a standing
queue. Slow start overshoots the capacity of the path and builds that queue;
thereafter the ACK clock paces the sender at the rate the bottleneck delivers,
and the queue keeps the bottleneck busy across the round-trip so that the link
does not idle. Congestion avoidance then raises the congestion window gradually,
the more gradually the larger the window, which is what lets competing flows
converge on their shares. It lowers the window when congestion is signalled. The
design rests on two premises: that loss indicates congestion, and that the
capacity of the path is stable.

Radio paths satisfy neither premise. Available bandwidth varies continually as
radio quality changes with distance, obstruction, and interference. Handover
between base stations or between satellites shifts it abruptly, and can drop the
packets in flight at the moment of the switch. Depending on the characteristics
of the link, packets could also be lost for reasons unrelated to queue
occupancy.

CUBIC reacts to this in three ways:

* It reduces the congestion window when loss is observed. Doing so is safe,
  because the loss could be due to congestion.

* It sends a packet when an acknowledgement is received. A slowdown of the path
  therefore slows the sender of its own accord, and a speedup raises it in the
  same way, for as long as the sender keeps the bottleneck queue from emptying.

* It increases the congestion window only slowly. Once the queue has emptied,
  only an increase of the window can raise the sending rate, and the larger the
  bandwidth-delay product, the longer that takes. Where the available bandwidth
  increases faster than that, the sender does not fully utilize it.

This document specifies ABBA, which addresses the last of the three: the rate at
which the congestion window increases. It combines two elements: an accelerated
increase of the congestion window, and an observation of the extent of the
bottleneck queue.

The accelerated increase engages while the round-trip time indicates a drained
bottleneck queue. It adds a controlled amount of queueing per round-trip, so
that only a very shallow queue is built. Once that queue has formed, the
round-trip time is no longer at its floor and CUBIC's increase resumes.

To remain fair on congested paths that provide no isolation, the bottom and the
top of the bottleneck queue are observed. The bottom is the minimum round-trip
time on the path; the top is the round-trip time when the queue is full. Between
them, the latest round-trip time says how much of the queue is occupied.
Acceleration is permitted only where the bottom and the top are distinguishable.

The top of the queue is observed during slow start, which overshoots the
capacity of the path and fills the bottleneck. If the queue does not return for
long enough, then either nobody on the path, including the sender itself, has
been able to fill the available bandwidth, or the characteristics of the path
have changed. In either case the recorded value no longer describes the
bottleneck, and the sender returns to slow start to observe it again. Doing so
also recovers bandwidth that the accelerated increase alone would not, as on
paths where loss is frequent.

ABBA modifies window increase only. It does not suppress, defer, or scale any
congestion response: every lost packet and every ECN-CE mark {{?ECN=RFC3168}}
produces the reduction that CUBIC specifies. The accelerated increase is in turn
bounded below that reduction, so under sustained congestion the sender always
yields, however the round-trip time signal may be misread.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
