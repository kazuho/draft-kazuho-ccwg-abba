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
whose available bandwidth changes rapidly during a connection. ABBA models the
round-trip time of the path as a function of the congestion window. A round-trip
time below what that model predicts indicates an increase in the bandwidth
available at the bottleneck, and ABBA then permits window growth faster than
CUBIC would, in proportion to the discrepancy. No congestion signal is
suppressed: every loss and every ECN-CE mark produces the reduction CUBIC
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
model is moreover discarded at every congestion event, and what replaces it
accelerates only once the path has shown the round-trip time falling materially
below the value at which congestion was signalled, or the window has grown well
past the one that signalled it. A sender under sustained congestion reaches
neither, and does not accelerate at all.

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


## Relationship to Low-Latency Mechanisms {#low-latency}

Even though low queueing delay is not an objective, it is not precluded either;
the responsibility for it rests with the network. Active queue management limits
persistent queueing {{?AQM=RFC7567}}, and flow isolation confines the delay
queue-building flow imposes to that flow, as in FQ-CoDel {{?FQ-CODEL=RFC8290}}.
These mechanisms act on queueing in the network, independently of how quickly a
sender acquires the bandwidth available to it. ABBA addresses the latter, when
the congestion window is insufficient to use the available bandwidth.
{{managed}} analyses how it behaves where such a bottleneck is deployed.

L4S {{?L4S=RFC9330}} combines network support with scalable congestion control
to achieve low queueing delay and high utilization. Prague can be implemented as
a modification to CUBIC, changing the response to loss and the growth that
follows ({{Section 2.4.1 of
?PRAGUE=I-D.briscoe-iccrg-prague-congestion-control}}), as can ABBA. The two
extensions can be combined, with Prague controlling the response to L4S
congestion signals and ABBA overriding the increase rate while the path delivers
better than its model predicts. This combination addresses Prague's concern
about slow adaptation following an increase in available capacity {{Section
3.1.2 of PRAGUE}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses min_rtt, the lowest round-trip time observed over the
lifetime of the connection, and latest_rtt as maintained by the transport.
beta_cubic is as defined in {{!CUBIC}}, beta is beta_cubic or, where the
congestion event was signalled by an ECN-CE mark and the sender reduces by a
different factor after such a mark ({{?ABE=RFC8511}}), that factor; and
cwnd_cubic denotes the congestion window that CUBIC's congestion avoidance sets
on an acknowledgement, whichever of its regions applies ({{Section 4.3 of
!CUBIC}} through {{Section 4.5 of !CUBIC}}).


# Overview {#overview}

ABBA models the round-trip time of the path as a function of the congestion
window, and accelerates the increase of that window while the path delivers
better than the model predicts.

A round-trip time below what the model predicts means the path is no longer the
one the model was fitted to. Bandwidth may have been added, competing traffic
may have departed, or the window may have been reduced by a loss that was not
congestion. In each case the sender holds a window the path now carries with
less queueing than the model expects, and ABBA increases the window faster than
CUBIC alone would, by an amount taken from that discrepancy.

Two points too close in round-trip time do not define a slope, and the model is
left flat until the path has shown more variation than that. Once the window has
grown well past the one that signalled congestion, the fitted line is left
behind for a simpler assumption: that the queue is occupied now, so round-trip
time should rise with the window from here on, faster still where competing
traffic is holding part of the queue. Where it does not rise, the path is
underutilized, and the sender accelerates until the round-trip time does.

A further condition covers the round-trip time returning to the lowest the
connection has seen. Where congestion was signalled well above that floor, the
sender accelerates toward a window whose round-trip time would exceed the floor
by a small fixed margin.

Neither candidate replaces CUBIC's own increase. The greater of the accelerated
window and the window CUBIC sets applies, so the sender is back on CUBIC's curve
as soon as the round-trip time rises to what the model predicts, and every
congestion signal produces the reduction CUBIC specifies.

The sections below use two constants and one derived quantity. MIN_RTT_SPAN,
5ms, is the range of round-trip time the path must have demonstrated before
either form of acceleration becomes available. MIN_RTT_SHORTFALL, 2ms, is how
far the round-trip time must fall below the model's prediction before that
prediction is acted upon. rtt_floor() is the lowest round-trip time observed
over the round trip preceding the call.


# The Round-Trip Time Model {#model}

The model is a line through two points, both observed within one period between
congestion events: the high point, the window and round-trip time at the
congestion event that began the period, where the queue was built up; and the
low point, the lowest round-trip time observed since, where it was at its
emptiest. The line between them says what round-trip time a window in that range
should produce.

An ABBA sender therefore holds four values in addition to the state of CUBIC:
high_cwnd and high_rtt ({{high}}); low_cwnd and low_rtt ({{low}}); and the
coefficients a and b of the line, which predicts a round-trip time of a * cwnd +
b ({{fit}}). All are established anew at each congestion event, and where such
an event is later determined to have been spurious ({{Section 4.9 of !CUBIC}}),
they are restored to the values they held before.

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
starts flat and remains so until the path shows a round-trip time MIN_RTT_SPAN
below it.

## Fitting the Line {#fit}

Whenever the low point is set or moves, the line is reevaluated from the two
points then in hand. Where those points do not support a line, either because
the low point has reached or passed the high one or because their round-trip
times lie within MIN_RTT_SPAN of each other, the previous model is kept or a
flat one is left in its place.

~~~
fit():
  if high_cwnd <= low_cwnd:
    return                          # keep the previous model
  if high_rtt - low_rtt < MIN_RTT_SPAN:
    a = 0                           # flat: no prediction to invert
    b = low_rtt
  else:
    a = min((high_rtt - low_rtt) / (high_cwnd - low_cwnd),
            low_rtt / low_cwnd)     # capped at the line through the origin
    b = low_rtt - a * low_cwnd
~~~

A lower round-trip time can arrive at a window at or beyond high_cwnd, which
would leave no span to fit across. The previous model is kept in that case:
drawing a line through the new point would absorb the very decline in round-trip
time that the model exists to detect.

The slope is capped at the line through the low point and the origin. On that
line the window is the bandwidth multiplied by the round-trip time; a steeper
one would put the window above what the path delivers in a round trip.

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
if cwnd > high_cwnd * (2 - beta):
  rtt_target = rtt_floor()
  if b is set:
    rtt_target = max(rtt_target, a * cwnd + b)
  a = rtt_target / cwnd
  b = 0
~~~

The sender therefore goes on accelerating until the queue builds to meet the
line.


# Accelerated Increase {#increase}

To accelerate the increase, the congestion window is adjusted on each
acknowledgement for which all of the following conditions are met:

* the sender is in congestion avoidance,
* the acknowledgement lies outside a recovery period,
* the sender is limited by its congestion window.

~~~
gain = 0
if a > 0 and b >= 0 and a * cwnd + b - latest_rtt >= MIN_RTT_SHORTFALL:
  w_ref = (latest_rtt - b) / a
  gain  = max(gain, (1 - w_ref / cwnd) / 2)
if high_rtt - min_rtt >= MIN_RTT_SPAN and latest_rtt < min_rtt + 2ms:
  gain  = max(gain, (min_rtt + 2ms) / latest_rtt - 1)
increase = min(bytes_acked * gain, cwnd / 2)
cwnd = max(cwnd + increase, cwnd_cubic)
~~~

bytes_acked is the data the acknowledgement newly acknowledged. The accelerated
window governs only where it exceeds cwnd_cubic, so acceleration can raise the
congestion window but never lower it.

The first candidate measures how far the path has moved from the one the model
was fitted to: the path is carrying the window with the queueing the model
predicts for a smaller one, and half that distance is taken per round trip, each
acknowledgement contributing its share of the window. The second is the case the
model cannot describe, the round-trip time having returned to the floor of the
connection rather than merely fallen below a prediction; scaling the window by
(min_rtt + 2ms) / latest_rtt targets the window whose round-trip time would be
2ms above that floor, the window and the round-trip time standing in proportion
at a fixed bandwidth. Requiring high_rtt to exceed min_rtt by MIN_RTT_SPAN keeps
it out of paths whose queue never showed enough depth for the floor to mean
anything. The greater of the two applies and not their sum: they are two
readings of one quantity, not two effects to be added.


# Properties

## Where Acceleration Engages {#fairness}

Every congestion event discards the model and the low point, and the low point
is reinitialized at high_rtt when the recovery period ends. The span is zero
there, so the model is flat and {{increase}} finds nothing to invert: the first
candidate is unavailable until the path has shown MIN_RTT_SPAN of decline from
the round-trip time at which congestion was signalled, or the window has reached
high_cwnd * (2 - beta) and the proportional model of {{extrapolate}} has taken
over. The second candidate is unavailable until the round-trip time comes within
2ms of the floor of the connection, from a high point at least
MIN_RTT_SPAN above that floor. A path that keeps signalling congestion supplies
none of these, and ABBA's window is CUBIC's throughout.

Where the decline does arrive, four things bound what follows.

* The model is fitted to this flow's own observations, and a competing flow's
  queue raises both of the points it is fitted through. What the model describes
  is the round-trip time this path produces as this window grows, whatever else
  is using the bottleneck.

* Acceleration requires the round-trip time to fall MIN_RTT_SHORTFALL below the
  prediction, so a round-trip time merely equal to it yields nothing, and the
  gain grows only as the shortfall does.

* Extrapolation stops at high_cwnd * (2 - beta) ({{extrapolate}}). Beyond it the
  model predicts the smoothed round-trip time at the window in hand, which is
  the most conservative prediction available from a sample.

* The window that acceleration produces competes with CUBIC's rather than
  replacing it, and no single acknowledgement may raise the window by more than
  half of itself.

## Yielding under Sustained Congestion {#yield}

No congestion signal is suppressed or deferred. Every lost packet and every
ECN-CE mark produces the reduction the underlying controller specifies.

A sender under sustained congestion does not accelerate at all. Each signal
resets the model to flat, and the round-trip time that would refit it,
MIN_RTT_SPAN below the round-trip time at which congestion was last signalled,
is by definition not what a congested path is producing. The proportional model
of {{extrapolate}} is the other way a flat model becomes usable, and it requires
the window to reach high_cwnd * (2 - beta), which is the window that signalled
congestion plus the whole of the reduction taken from it; a path that keeps
signalling congestion interrupts that growth long before. The second candidate
of {{increase}} is gated on the same span as the first and additionally requires
the round-trip time to be within 2ms of the floor of the connection. A sender
whose round-trip time signal is misleading therefore increases too quickly for
as long as the misreading lasts, and yields as soon as the path signals.

## Behavior at a Managed Bottleneck {#managed}

An actively managed bottleneck typically combines three things: isolation
between flows, congestion signalled from a shallow queue using ECN, and a buffer
far deeper than that queue. Isolation distributes bandwidth among flows
irrespective of how aggressively each sends, and confines the delay a
queue-building flow creates to that flow. Aggressive signalling of congestion
keeps the delay small, while the depth of the buffer and the use of ECN-CE
minimize the packet drops that would cost the endpoints recovery delay.

FQ-CoDel {{FQ-CODEL}} is the common instance. It hashes flows into separate
queues and signals congestion using ECN once the queue has stood above 5ms for
100ms (the defaults defined in {{Section 5.3 of ?CODEL=RFC8289}}). The buffer
behind is sized for the link, 10240 packets by default, a limit the algorithm's
own congestion signalling keeps it from reaching.

The span such a bottleneck offers is its own setpoint. The first congestion
event of a connection ends slow start and carries its overshoot, so high_rtt
comes out well above the floor; every event after that is raised from the queue
the bottleneck permits to stand, and the decline available afterwards is about
that queue's depth. At FQ-CoDel's default target of 5ms the span sits exactly at
MIN_RTT_SPAN, and where the target is larger, as {{Section 5.2.2 of FQ-CODEL}}
recommends for slow links, it clears it comfortably.

{::comment}
The marginality at the 5ms default deserves a second opinion: MIN_RTT_SPAN is
5ms and CoDel's target is 5ms, so whether the model fits at all at such a
bottleneck turns on where the samples land either side of the setpoint. Worth
checking against the CoDel traces.
{:/comment}

A queue that shallow is where CUBIC is least able to keep the path busy, and
where acceleration is of most use. Each signal removes (1 - beta) of the window,
and where the queue the signal was raised over holds less than the reduction
removes, the sender comes out below the bandwidth-delay product and the
bottleneck idles until the window is restored. Against a 5ms setpoint that is
every path whose idle round-trip time exceeds 5ms * beta / (1 - beta): 11.7ms
where beta is 0.7, and 28.3ms where the sender applies the ECN factor of 0.85
that {{ABE}} recommends. The time CUBIC takes to restore the window grows with
the cube root of the amount removed ({{Section 4.2 of CUBIC}}), so on a large
window it is measured in seconds, and a further signal arriving before then
leaves the sender lower still. Scalable congestion controls address this by
reducing in proportion to the extent of the marking; ABBA addresses it from the
other side, the round-trip time falling below the model being exactly what a
window below the bandwidth-delay product produces.

Therefore, at such a bottleneck, ABBA retains CUBIC's behavior while shortening
periods of underutilization, including those caused separately by packet losses.
Where the accelerated increase does carry the window past the marking threshold,
on a longer path or after an overshoot, the bottleneck marks sooner and the
congestion avoidance period is correspondingly shorter. Congestion being
signalled by ECN-CE, that only costs a reduction of the window, which is quickly
fixed by the accelerated increase. The shorter period does not reach other
flows: isolation confines the queue to the flow that built it, and apportions
bandwidth irrespective of how aggressively each sends.


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
