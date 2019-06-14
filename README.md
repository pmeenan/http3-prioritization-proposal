# HTTP/3 Prioritization Proposal

The HTTP/2 prioritization scheme provides a lot of flexibility but is extremely complex to implement (at least for a Browser and to deliver resources in an optimal order). The dependency tree management requires the tree state to be synchronized on both ends of the connection. Scheduling both sequential downloads (for things like render-blocking scripts and css) and concurrent downloads (for things like progressive images) and coordinating between the different resource types is complex enough that no browser has implemented a fully optimal tree. As things stand today:

**Chrome:** Builds a linear list with each request exclusively dependent on the resource before it. Resources internally are given a 1-5 priority level and they are scheduled in the order they are processed (with higher-priority items being inserted ahead of lower-priority items as necessary). This works well for the resources that benefit from sequential downloads but not for those that would benefit from concurrency.

**Firefox:** Groups resources into groups that are scheduled as a whole with groups loading sequentially after each other and resources within a group downloading concurrently. This tends to be better for images but worse for resources that would benefit from sequential loading.

**Safari:** All resources download concurrently (attached to the root) with a weight that is set based on the internal 1-5 priority.

**Edge:** No prioritization specified (may be moot with the move to Chromium).

## Design Goals
* A priority scheme that can provide the appropriate scheduling without needing the full context of other streams.
* Ordering of streams (more important delivered before lower importance).
* Specifying concurrency of download for requests to allow for both sequential and concurrent transfer of streams.
* Simple to implement for both clients and servers.

## Proposed Prioritization Scheme

The prioritization will be packed into a single byte (with no other stream ID's or data required for prioritization).

```
 0               
 0 1 2 3 4 5 6 7 
+-+-+-+-+-+-+-+-+
|  Priority   |C|
+-+-+-+-+-+-+-+-+
```

In all cases, prioritization of stream delivery is based on what streams have data available to send.

### Priority (7 bits, 0-127): 

* Streams at a higher priority are delivered in their entirety before streams at a lower priority.
* Streams at the same priority level are delivered as defined by their concurrency.

Implementations may not be able to use the full bit space and should drop the least significant bits when mapping from the 7-bit space to their implementation's priority levels. It is also possible to negotiate other uses for the space, such as using the 4 most significant bits for priority and the 3 least significant bits for defining groups.

### Concurrency (1 bit):

At a given priority, all requests are grouped based on their concurrency. The overall available bandwidth is split evenly between the two groups and then allocated within each group:

* **0**: "Sequential" : Streams with a concurrency of 0 are delivered sequentially within the group. This is optimal for things like async or deferred scripts where you may want to load them quickly but not exclusively and where they are optimally delivered completely and in order.
* **1**: "Concurrent" : Streams with a concurrency of 1 share bandwidth across all streams in the group.

Exclusive resource delivery with no bandwidth sharing can be achieved by using a higher priority and setting the concurrency to 0 (with no resources at the same priority level using a concurrency of 1).

![Priority Levels and Concurrency](images/priorities.png)

### Bandwidth Splitting
Bandwidth splitting is implementation-specific. One possible implementation is when the server makes a decision on a frame-by-frame basis:

* Only consider responses where data is available to be sent.
* Select from the responses with the highest priority level.
* Round robin between the "Concurrency 0" and "Concurrency 1" groups, picking one frame from each group.
* Within the "Concurrency 0" group, fill the frame with the response with the lowest stream ID.
* Within the "Concurrency 1" group, round robin across all of the available responses.

### Example Browser Prioritization

Based on Chromium's prioritization scheme:

![Sample Prioritization](images/sample.svg)
