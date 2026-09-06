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

This document specifies ABBA, an extension to CUBIC congestion control for paths
whose available bandwidth changes rapidly during a connection, or that cause
non-congestive loss. While the round-trip time indicates a drained bottleneck
queue, ABBA raises the congestion window faster than CUBIC would, adding a
controlled amount of queueing. This acceleration is gated on an observation of
the extent of that queue, so it does not engage where a shared path is
congested. ABBA modifies window increase only: every loss and every ECN-CE mark
produces the reduction that CUBIC specifies.

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
between base stations or between satellites shifts it abruptly. Packets are
dropped in bursts rather than at random: at the moment of a handover, when the
link layer gives up retransmitting, and when the buffer can no longer hold the
queue. Only the last of these is congestion.

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


# Overview

An ABBA sender maintains the following state in addition to that of CUBIC:

full_rtt:
: The smoothed round-trip time observed when the bottleneck queue was last known
  to be full.

cur_period_min:
: The lowest round-trip time sample taken since the most recent congestion
  event. Unset until a sample has been taken.

past_periods_min:
: An estimator, holding a smoothed value and a variance, fed with cur_period_min
  at the end of each period between congestion events. The period in progress is
  not included.

last_high_queue_at:
: The time at which the bottleneck queue was most recently observed to be
  substantially occupied.

The sender also uses min_rtt, the lowest round-trip time observed over the
lifetime of the connection, and latest_rtt and smoothed_rtt as maintained by the
transport. W_max, cwnd_epoch, C, alpha_cubic and beta_cubic are as defined in
{{!CUBIC}}, and cwnd_cubic denotes the congestion window that CUBIC's congestion
avoidance sets on an acknowledgement, whichever of its regions applies
({{Section 4.3 of !CUBIC}} through {{Section 4.5 of !CUBIC}}).


# Accelerated Increase {#increase}

While the round-trip time indicates that the bottleneck queue has drained, the
sender increases its congestion window faster than CUBIC alone would, by an
amount derived from the queueing that increase will add. The accelerated increase
does not replace CUBIC's own: the greater of the two applies, so the sender is
back on CUBIC's curve as soon as the queue reforms.

In congestion avoidance, the sender sets its congestion window to
abba_cwnd(cwnd, cwnd_cubic) in place of cwnd_cubic, where cwnd is the window
before that update.

~~~
abba_cwnd(cwnd, cwnd_cubic):
  if congestion-window limited
      and is_set(past_periods_min)
      and full_rtt > min_rtt + 10ms
      and full_rtt > latest_rtt * 1.05
      and latest_rtt < drain_threshold():
    return max(cwnd_cubic, cwnd + segments_acked * ratio())
  return cwnd_cubic

drain_threshold():
  estimated_floor = past_periods_min.smoothed - past_periods_min.variance / 2
  return min(max(estimated_floor, min_rtt + 2ms), cur_period_min + 2ms)

ratio():
  return min(max(2ms / drain_threshold(), 1/40), (1 / beta_cubic - 1) / 2)
~~~

Requiring past_periods_min to hold a value keeps acceleration out of the initial
slow start. The two full_rtt conditions require the path to have shown a queue
deep enough for its ends to be told apart, and headroom to remain between the
current round-trip time and a full queue.

The estimated floor lets drain_threshold follow the path's round-trip time floor
when min_rtt has ceased to describe it. Bounding it at cur_period_min + 2ms keeps
acceleration within 2ms of the best round-trip time seen in the period in
progress, whatever the estimator holds.

Deriving the rate from a quantity of queueing per round-trip, rather than from a
fixed proportion of the window, keeps the queue that acceleration can build
independent of the window size and of the bandwidth of the path. Round-trip time
feedback arrives one round late, so on a path whose round-trip time is below
100ms the conditions cease to hold no later than when 4.5ms of queueing has been
built.

The upper bound holds one round-trip of acceleration to half of what would be
needed to reverse a reduction, so that a sender under sustained congestion
yields; see {{yield}}. A sender that reduces by a different factor when the
signal is an ECN-CE mark uses that factor after such a mark.


# Observing the Bottleneck Queue {#observe}

Accelerated increase requires the sender to know both ends of the range the
round-trip time takes on this path: the value when the bottleneck queue is empty,
and the value when it is full. min_rtt supplies the former. This section defines
how the latter is obtained, and how it is retaken when it ceases to describe the
path.

## The Full-Queue Round-Trip Time {#full-rtt}

Slow start overshoots the capacity of the path, so the queue is full when slow
start ends.

~~~
on recovery exit:
  if entered from slow start:
    full_rtt = smoothed_rtt
    last_high_queue_at = now
~~~

smoothed_rtt is read before the RTT estimator is updated by the acknowledgement
that ends the recovery: that acknowledgement covers a packet sent after the
congestion window was reduced, so its sample reflects a queue that has already
begun to drain. Waiting for the recovery period to end, rather than reading the
value at the congestion event itself, lets the samples arriving during recovery
contribute.

Conservative variants of slow start, such as HyStart++ {{?HYSTART=RFC9406}},
still build a queue of roughly one min_rtt, their reaction being delayed by a
round-trip over which the sender doubles its rate. full_rtt therefore comes out
at about twice min_rtt, which satisfies the condition in {{increase}} wherever
min_rtt exceeds 10ms. On shorter paths the queue built may fall short of that,
and acceleration does not engage.

## Per-Period Minima

min_rtt is the lowest round-trip time of the whole connection, and may no longer
describe a path whose floor has moved. The sender therefore also tracks the
lowest round-trip time of the period in progress, which bounds the drain
threshold of {{increase}}, and an estimate formed from the minima of completed
periods, which places it.

~~~
on an rtt sample:
  if taken outside a recovery period
      and (not is_set(cur_period_min) or sample < cur_period_min):
    cur_period_min = sample

on a congestion event:
  if is_set(cur_period_min):
    past_periods_min = update(past_periods_min, cur_period_min)
  unset cur_period_min

update(est, sample):
  if not is_set(est):
    est.smoothed = sample
    est.variance = 5ms
  else:
    est.variance = est.variance * 3/4 + abs(est.smoothed - sample) / 4
    est.smoothed = est.smoothed * 7/8 + sample / 8
~~~

Samples taken during recovery are excluded, the congestion window being under
reduction at the time.

Neither the weights nor the exclusion of the period in progress is incidental:
together they are what keeps the estimator independent of the sender's own
acceleration ({{fairness}}).


## Recalibration {#recalibrate}

full_rtt describes the path as it was when the observation was taken. If the
queue does not return for long enough, either nothing on the path, including the
sender itself, has been able to fill the available bandwidth, or the
characteristics of the path have changed. In both cases the observation is
retaken: the sender returns to slow start, with no slow start threshold and its
congestion window unchanged. W_max is cleared, to prevent the sender from
entering fast convergence once that slow start concludes
({{Section 4.7 of !CUBIC}}).

~~~
on an rtt sample:
  if in congestion avoidance and congestion-window limited:
    if smoothed_rtt >= (min_rtt + full_rtt) / 2:
      last_high_queue_at = now
    else if now - last_high_queue_at >= 2 * expected_high_queue_interval():
      recalibrate()

on an ECN-CE mark:               # including one received during recovery
  last_high_queue_at = now

recalibrate():
  W_max    = unset
  ssthresh = infinity            # cwnd is retained

expected_high_queue_interval():
  K    = cbrt((cwnd_epoch / beta_cubic - cwnd_epoch) / C)
  reno = C * K^3 / alpha_cubic * full_rtt
  return min(K, reno) * cbrt(full_rtt / min_rtt)
~~~

expected_high_queue_interval is the time a flow on the current
congestion-avoidance trajectory would take to refill the queue. Both a CUBIC and
a Reno-friendly flow are considered because a sender follows the greater of the
two curves, and the shorter of the two return times is used.

Only ECN-CE marks count as observations here, not losses. A mark is unambiguous
evidence that the queue was deep, whereas a loss may be non-congestive; were
losses to refresh the timestamp, frequent random loss on a drained path would
restart the interval indefinitely and recalibration could never arm on the paths
it exists for.

Twice the interval is used so that a competing flow refilling the queue on its
own trajectory refreshes the observation well within the window. Recalibration
therefore does not occur while any flow is making use of the bottleneck.

Retaining the congestion window leaves the sender probing from its current rate.
full_rtt is retaken when the recovery that ends that slow start exits, as
described in {{full-rtt}}.


# Properties

## What Acceleration Can Take {#fairness}

Whether acceleration engages at all turns on the bottom and the top of the
bottleneck queue being distinguishable.

Where they are not — full_rtt within 10ms of min_rtt — the conditions in
{{increase}} never hold and the sender behaves as CUBIC throughout. That is the
case in which the delay signal could not separate a drained queue from an
occupied one, and it is also the case, a shallow bottleneck buffer, in which
acting on a misreading would cost a competing flow most.

Where they are distinguishable, acceleration can engage even while a competing
flow occupies the bottleneck, since past_periods_min follows the minima that flow
produces and the estimated floor rises with them. Four things bound what this
permits:

* The estimated floor sits half a variance below the smoothed minima. The wider
  those minima are spread, which is what competition produces, the further below
  them the gate sits.

* drain_threshold is capped at cur_period_min + 2ms whatever past_periods_min
  holds, so acceleration never engages more than 2ms above the best round-trip
  time of the period in progress.

* The congestion window is the greater of the accelerated value and the one CUBIC
  sets, so acceleration never slows the sender, and control returns to CUBIC's
  curve as soon as the queue reforms.

* The rate adds about 2ms of queueing per round-trip, against a range the
  conditions require to exceed 10ms. With the round-trip of feedback delay, the
  queue built stops at about 4.5ms: less than half the range the path has
  demonstrated.

Nor can the sender raise the gate by accelerating. Where estimated_floor
determines drain_threshold, acceleration requires latest_rtt to be below it, and
cur_period_min is no greater than latest_rtt; the value fed into the estimator at
the end of such a period is therefore below the estimate itself. With a
smoothed-value weight of 1/8, a variance weight of 1/4, and an estimated floor of
the smoothed value less half the variance, feeding a sample below the current
estimate lowers it. The three are related: writing the weights as `a` and `b` and
the floor as `smoothed - k * variance`, the condition for a sample to lower the
floor is exactly that the sample is below the floor when `a = b * (1 - k)`. The
weights are therefore part of this property rather than free parameters.

Recalibration cannot arm while the bottleneck is in use at all. A competing flow
refilling the queue on its own congestion-avoidance trajectory produces a
high-queue observation well inside twice expected_high_queue_interval, and each
such observation restarts the interval.

## Yielding under Sustained Congestion {#yield}

ABBA modifies the increase of the congestion window only. Every lost packet and
every ECN-CE mark produces the reduction the underlying controller specifies, and
no congestion signal is suppressed, deferred, or scaled.

Because ratio is capped at half of what would reverse a reduction over one
round-trip, a sender cannot recover a reduction before a further congestion
signal can arrive. A sender whose round-trip time signal is misleading therefore
increases too quickly for a round-trip or two and then yields.


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
