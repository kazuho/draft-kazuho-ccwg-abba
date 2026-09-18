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
  CUBACK:
    title: "CUBACK: CUBIC Driven by the ACK Clock"
    target: https://github.com/kazuho/draft-kazuho-ccwg-cuback
    author:
      -
        ins: K. Oku
        name: Kazuho Oku

...

--- abstract

This document specifies ABBA, an extension to CUBIC congestion control for paths
whose available bandwidth changes rapidly during a connection. ABBA tracks the
relationship between the congestion window and the round-trip time of the path.
A round-trip time below what that relationship implies indicates an increase in
the bandwidth available at the bottleneck, and ABBA then permits window growth
faster than CUBIC would, in proportion to the discrepancy. No congestion signal
is suppressed: every loss and every ECN-CE mark produces the reduction CUBIC
specifies.

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

This document specifies ABBA, which addresses the last of the three. While the
bottleneck queue is occupied and the sender is limited by its congestion window,
round-trip time rises linearly with the congestion window: the slope is the
reciprocal of the bottleneck bandwidth, and the intercept is the queue held by
competing flows that take their losses asynchronously.

ABBA approximates that relationship with a line through two points, both
observed within one period between congestion events: the window and round-trip
time on entering recovery, where the queue is at its fullest, and those at the
lowest round-trip time observed afterwards, where it is at its emptiest. Where a
later observation falls below that line, the window is no longer producing the
queueing it did, which is to say the bandwidth of the bottleneck has increased.
ABBA then accelerates the increase of the window in proportion to that
discrepancy.

ABBA does not suppress or defer any congestion response: every lost packet and
every ECN-CE mark {{?ECN=RFC3168}} produces the reduction CUBIC specifies. The
model is moreover discarded at every congestion event, and the increase taken in
the round trip that follows a reduction never surpasses that reduction. A sender
under sustained congestion, where a signal arrives every round trip, therefore
yields.

The following subsections set out the objective ABBA is aimed at — the delivery
time of an object — and how it relates to mechanisms that pursue low queueing
delay.

## Optimizing for Object Delivery {#object-delivery}

A finite transfer benefits from available bandwidth only while it has data left
to send. Time spent acquiring that bandwidth can therefore account for a
substantial part of its completion time, even if the congestion controller would
eventually reach the same steady-state rate.

Together with Rapid Start {{?I-D.kazuho-ccwg-rapid-start}} and Cuback
{{CUBACK}}, ABBA addresses different parts of this acquisition. Rapid Start
accelerates the initial expansion of the congestion window and controls the
transition into congestion avoidance. Cuback governs subsequent growth,
including convergence when a flow competes with traffic already occupying the
bottleneck. Because its clock is derived from acknowledgements, a flow that is
gaining share traverses CUBIC's curve faster than elapsed time would carry it,
so a newcomer reaches its share sooner than under CUBIC, without the bottleneck
queue having to drain.

ABBA complements these mechanisms when the congestion window becomes
insufficient to use the available bandwidth. This can occur after an increase in
capacity, the departure of competing traffic, or a window reduction in response
to non-congestive loss. Where the round-trip time observations permit it, ABBA
accelerates growth beyond that of the underlying congestion-avoidance algorithm.

Drops on such paths are commonly reported as a packet loss ratio, but they do
not occur at random: they arrive in the bursts described above, and a burst is
more or less a single congestion event rather than a series of them. The
congestion events they trigger are therefore far rarer than the ratio suggests
under a random loss model. What matters more to delivery time than recovering
from those events is adapting to the bandwidth that becomes available between
them. When a succession of recoveries hides the relationship between round-trip
time and congestion window, ABBA still accelerates the growth of the window once
it has climbed well past the one the last event interrupted ({{extrapolate}}).

Low queueing delay is not an objective of ABBA, because when the data of an
object is already available at the sender, withholding it to keep the bottleneck
queue short does not remove the wait; it changes where the data waits, and
increases the chance of underutilizing the bottleneck. What withholding does
reduce is the pressure the sender places on competing flows to yield, which
would prolong bandwidth acquisition and consequently object delivery.


## Relationship to Active Queue Management {#low-latency}

Even though low queueing delay is not an objective, it is not precluded either.
Active queue management is used to enforce short queueing delays
{{?AQM=RFC7567}}, but that leads to occasional starvation of the bottleneck
queue, and hence to underutilization of the link, especially on paths where the
available bandwidth changes rapidly. ABBA provides the needed mitigation.

Being clocked by acknowledgements, an ABBA sender follows the rate the
bottleneck is delivering. Where the bandwidth falls, the delay the queue imposes
rises, and the bottleneck signals congestion. In response, the sender yields
immediately. Where the round-trip time stays flat or falls, the sender raises
its window quickly and takes up whatever is there. Where the available bandwidth
varies continually, as on a radio path, the sender's rate therefore follows the
fluctuation, while the queueing delay stays near the target the bottleneck
signals at.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the notation of {{!CUBIC}}, in which window sizes are
expressed in segments of the SMSS.

It also uses latest_rtt as maintained by the transport. beta_cubic is as defined
in {{!CUBIC}}, beta is beta_cubic or, where the congestion event was signalled
by an ECN-CE mark and the sender reduces by a different factor after such a mark
({{?ABE=RFC8511}}), that factor; and cwnd_cubic denotes the congestion window
that CUBIC's congestion avoidance sets on an acknowledgement, whichever of its
regions applies ({{Section 4.3 of !CUBIC}} through {{Section 4.5 of !CUBIC}}).


# Overview {#overview}

ABBA tracks the relationship between the congestion window and the round-trip
time of the path, and accelerates the increase of that window while the path
delivers better than the relationship implies.

A round-trip time below what that relationship implies means the path is no
longer the one it was measured on. Bandwidth may have been added, competing
traffic may have departed, or the window may have been reduced by a loss that
was not congestion. In each case the sender holds a window the path now carries
with less queueing than the relationship allows for, and ABBA increases the
window faster than CUBIC alone would, by an amount taken from that discrepancy.

Two points too close in round-trip time do not define a slope, and the model is
left flat until the path has shown more variation than that. Once the window has
grown well past the one that signalled congestion, the fitted line is left
behind for a simpler assumption: that the queue is occupied now, so round-trip
time should rise with the window from here on, faster still where competing
traffic is holding part of the queue. Where it does not rise, the path is
underutilized, and the sender accelerates until the round-trip time does.

Acceleration does not replace CUBIC's own increase. The greater of the
accelerated window and the window CUBIC sets applies, so acceleration can only
raise the window: once the round-trip time rises to what the model predicts, the
window holds until CUBIC's curve catches up and CUBIC resumes control of the
increase. Every congestion signal produces the reduction CUBIC specifies.

The sections below use two constants:

* RTT_SPAN_THRESH, 5ms, the range of round-trip time the path must have
  demonstrated before acceleration becomes available;
* MAX_SLOPE_RATIO, 2/3, the share of the fitted slope the model keeps.

They also use rtt_floor(), a function that returns the lowest round-trip time
observed over the last round trip.


# The Round-Trip Time Model {#model}

The model is a line through two points, both observed within one period between
congestion events: the high point, the window and round-trip time at the
congestion event that began the period, where the queue was built up; and the
low point, the lowest round-trip time observed since, where it was at its
emptiest. The line between them says what round-trip time a window in that range
should produce.

An ABBA sender therefore holds the following in addition to the state of CUBIC:

* high_cwnd and high_rtt, the high point ({{high}});
* low_cwnd and low_rtt, the low point ({{low}});
* a and b, the coefficients of the line, unset until a fit succeeds ({{fit}}).

All are established anew at each congestion event, and where such an event is
later determined to have been spurious ({{Section 4.9 of !CUBIC}}), they are
restored to the values they held before.

## The High Point {#high}

At the congestion event that begins a period, the sender records the congestion
window before the reduction as high_cwnd. The round-trip time recorded with it
depends on the signal: where the event was signalled by a packet loss, high_rtt
is the lowest round-trip time observed from the event through the end of the
recovery period; where it was signalled by an ECN-CE mark, high_rtt is
rtt_floor(), the lowest observed over the round trip preceding the mark.

The two signals place the observation differently. A loss is detected a round
trip after the packet was sent; senders go on transmitting until then, so the
queue remains full as their recovery periods open. Samples taken during
recovery therefore still describe a full queue, and the lowest of them is the
conservative estimate of what a full queue costs. An ECN-CE mark instead reports
a queue that was already persistently occupied ({{Section 5.1 of ECN}}), so the
observation to keep is the one from before the mark, and samples taken after it
are not admitted.

## The Low Point {#low}

The low point is initialized when the recovery period ends, at the congestion
window then in hand and at high_rtt. Thereafter, whenever an acknowledgement in
congestion avoidance carries a round-trip time below low_rtt, the point moves to
that sample and to the window that accompanied it. The model is fitted afresh on
initialization and on every such move.

By taking high_rtt initially as the low point's round-trip time, the model
starts flat and remains so until the path shows a round-trip time
RTT_SPAN_THRESH below it.

## Fitting the Line {#fit}

Whenever the low point is set or moves, the line is reevaluated from the two
points then in hand. Where those points do not support a line, either because
the low point has reached or passed the high one or because their round-trip
times lie within RTT_SPAN_THRESH of each other, the previous model is kept or a
flat one is left in its place.

~~~
fit():
  if high_cwnd <= low_cwnd:
    return                          # keep the previous model
  if high_rtt - low_rtt < RTT_SPAN_THRESH:
    a = 0                           # flat: no prediction to invert
    b = low_rtt
  else:
    a = min((high_rtt - low_rtt) / (high_cwnd - low_cwnd),
            low_rtt / low_cwnd)     # capped at the line through the origin
    a = a * MAX_SLOPE_RATIO         # then flattened about the low point
    b = low_rtt - a * low_cwnd
~~~

A lower round-trip time can arrive at a window at or beyond high_cwnd, which
would leave no span to fit across. The previous model is kept in that case:
drawing a line through the new point would absorb the very decline in round-trip
time that the model exists to detect.

The slope is capped at the line through the low point and the origin. On that
line the window is the bandwidth multiplied by the round-trip time; a steeper
one would put the window above what the path delivers in a round trip.

It is then flattened to MAX_SLOPE_RATIO of itself, the line rotating about the
low point. The slope is the reciprocal of the bandwidth, so two thirds of it
stands for a bandwidth one and a half times as large: the model predicts as
though that increase had already occurred, and {{increase}} accelerates only
where the path does better still.

## Beyond the Observed Range {#extrapolate}

Growing to high_cwnd * (2 - beta), about twice the window the
reduction left the sender with, without congestion reappearing says the path is
no longer the one that signalled it: its bandwidth may have risen, a competing
flow may have departed, or one that held much of the bandwidth may have yielded
and be reclaiming it more slowly than this sender is taking it. Past high_cwnd
the sender is already in CUBIC's convex region, probing upward for a new maximum
on the same reasoning ({{Section 4.5 of !CUBIC}}). In each case the capacity is
there to be taken, and where the competitor is still present, taking it is what
convergence consists of.

The model is therefore redrawn as the line from the origin through where the
sender now stands, the current window and rtt_floor(). Drawing from the origin
is what drops the intercept: no part of the round-trip time is credited to the
competing flows any longer. Where a fitted line predicted more round-trip time
than that, the sender has been chasing the higher value, and the line is drawn
through that instead.

~~~
if b != 0 and cwnd > high_cwnd * (2 - beta):
  rtt_target = rtt_floor()
  if b is set:
    rtt_target = max(rtt_target, a * cwnd + b)

  a = rtt_target / cwnd
  b = 0
~~~

The sender therefore goes on accelerating until the queue builds to meet the
line.


# Accelerated Increase {#increase}

To accelerate the increase, when all of the following conditions are met:

* the sender is in congestion avoidance,
* the acknowledgement lies outside a recovery period,
* the sender is limited by its congestion window,

the congestion window is adjusted on each acknowledgement as follows.

~~~
accel = 0
if a > 0 and latest_rtt < a * cwnd + b:
  w_ref = (latest_rtt - b) / a
  gain  = 2 / 3 * (1 - w_ref / cwnd)
  accel = min(segments_acked * gain, cwnd / 2)

cwnd = max(cwnd + accel, cwnd_cubic)
~~~

segments_acked is the number of segments the acknowledgement newly acknowledged.

The gain measures how far the path has moved from the one the model was measured
on. Where the round-trip time falls below what the model predicts for the window
in hand, the path has room the window is not using, and two thirds of that room
is taken each round trip.

The increase is capped at half the window per acknowledgement. The result
competes with what CUBIC would have set from the same window, so acceleration
can raise the congestion window but never lower it. Once the gain no longer applies, the
window holds, and once CUBIC's curve catches up, the increase is handled by
CUBIC.


# Properties

## Convergence and Fairness {#fairness}

Within the observed range, acceleration requires high_rtt to exceed low_rtt
by at least RTT_SPAN_THRESH. Until that span has been observed, window growth
is controlled entirely by CUBIC, unless the window exceeds the threshold for
extrapolation ({{extrapolate}}).

On a path whose bandwidth, idle delay, and queue capacity remain stable,
acceleration is expected to remain inactive when flows are near convergence
or when the sender is yielding share to competing traffic. Near convergence,
the fitted relationship should remain representative; when the sender is
yielding share, competing traffic increases the queueing its window encounters.
Retaining only two thirds of the fitted slope leaves a margin for variation
in that relationship.

Whether acceleration engages is decided by the round-trip time. Where it meets
or exceeds the model's prediction, there is no shortfall and window growth
remains controlled by CUBIC.

Where the round-trip time falls below that prediction, the path is carrying the
window with less queueing than the model expected. This can indicate unused
bandwidth or capacity released by competing traffic. Under additive increase
and multiplicative decrease, flows holding larger windows release more capacity
when they yield, while additive increase lets flows holding smaller windows
catch up. ABBA accelerates that acquisition where the round-trip time
observations permit it, supporting the same convergence process.

Where acceleration no longer engages, the window is held where it stands until
CUBIC's curve catches up. Subsequent growth is then controlled by CUBIC, unless
a new shortfall permits acceleration again. Every congestion event retains
CUBIC's reduction and discards the model.

Consequently, the convergence and fairness properties of ABBA resemble those of
CUBIC ({{Section 5.6 of !CUBIC}}), subject to the effects of acceleration
described above.


## Yielding under Sustained Congestion {#yield}

No congestion signal is suppressed or deferred. Every lost packet and every
ECN-CE mark produces the reduction the underlying controller specifies.

Under sustained congestion a signal arrives every round trip, so the increase
taken in the round trip that follows a reduction must not surpass that
reduction.

The gain is bounded by how far the window has run past the low point. In one
round trip, CUBIC never grows the window more than one and a half times
({{Section 4.2 of CUBIC}}), and ABBA cannot exceed that either. Relative to the
round-trip time, the discrepancy becomes the largest when it stays at the low
point; any acknowledgement carrying a lower one moves the low point of the model
and resets the discrepancy. At those extremes the line, passing through the low
point, places w_ref at low_cwnd, and one round trip after exiting recovery the
gain is 2/3 * (1 - 2/3), two ninths.

    TODO: this no longer holds for beta_ecn. The requirement is that the
    increase in one round trip not surpass the reduction, that is
    gain < 1 / beta - 1:

      beta_cubic 0.7  -> threshold 0.4286, (1 + 2/9) * 0.7  = 0.856  ok
      beta_ecn   0.85 -> threshold 0.1765, (1 + 2/9) * 0.85 = 1.039  fails

    At a gain of two thirds of the gap, the worst case is 2/9 = 0.222,
    above the 0.1765 that beta_ecn admits. It held at 1/6 when the gain
    was half the gap. Either the multiplier comes down, or the claim is
    scoped to loss-signalled reductions.


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
