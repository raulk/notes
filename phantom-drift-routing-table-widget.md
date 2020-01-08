# Phantom Drift: thoughts on routing table widget

> _status: draft._

* **Motivation.** We want to visualise the structure of the routing table, which
  is split in `N` logical `k`-buckets where `N` = (0,256], and `k` is a
  configuration value (default: 20) representing the bucket capacity in terms of
  number of peers. We seek to capture bucket members, various peer-scoped stats,
  activity, and changes.
    * To properly construct the viz structure, the introspection protocol must
      report the current value of `k`.
    * The table is unfolded lazily. We start with a single catch-all bucket 0,
      and as we find peers at different XOR distances and buckets overflow, we
      split the catch-all bucket to store same-distance peers.
    * It is extremely unprobable, if not outright unfeasible, to entirely unfold
      the routing table to its maximum stretch. Realistically, a routing table
      for a healthy long-lived peer contains 11-16 dedicated buckets + the
      catch-all bucket. This is because each bucket represents log2n region of
      the global network tree. The subset represented by each bucket halves the
      previous bucket, so by bucket 15, the probability of finding a member is
      2^(256-15)/2^256 = 0.001525878906%.

* **Layout.** Assuming a horizontal layout where buckets range from
  left-to-right, we need to decide if the left-most bucket represents the
  nearest or the furthest distance.
    * Choosing the _nearest_ makes this bucket the catch-all bucket, containing
      a mixture of peers with different distances not covered by dedicated
      buckets. An unfold operation would slide all buckets to the right.
    * Choosing the _furthest_ entails that the table unfolds by appending new
      buckets to the right, which might not be visible and may go unnoticed.
    * Given this is a standalone viz, we can consider full-screen layouts (e.g.
      grid).
    * Regardless, the catch-all bucket contains a variety of distances, so we
      should display the distance of each peer inside its square.

* **Bucket membership & churn.** As peers arrive and depart (churn), and as we
  traverse the network, we will evict existing peers to replace them with new
  ones (or not). Bucket membership changes need to be brought to attention.

* **Activity cues.** As with the original design, flash peers based on inbound
  and outbound DHT messages.
    * How do we differentiate between inbound and outbound requests?
    * Consider varying the intensity/vibrancy of the flash in proportion to the
      delta between the previous read and the incoming read (such that peers
      with more traffic exchanged flash brighter).

* **Fault depiction.** Display faults as red fill flashes, potentially, or red
  borders.

* **Peer stats.** We want to display the following info, possibly on hover or in
  a secondary pane on click:
    * Peer score, which affects eviction.
    * Age (time in routing table).
    * Timestamp of last activity.
    * Histogram of messages exchanged over 5m, 15m, 30m, 1h, etc.
    * Total number of messages exchanged (in/out) per message type.
    * Total number of faults.
    * Latency.
    * more.
     
* **Sorting.** Capability to sort bucket elements by: age, last activity,
  cumulative activity.

* **Movement.** The routing table is both a read and write-intensive data
  structure. Events that lead to changes in its content are evictions,
  disconnections, faults, etc. Faithful to the "drift" principle, we'd like to
  register those changes in the UI in a semi-persistent manner, such that
  elements don't disappear or change abruptly with no apparent reason.