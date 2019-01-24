# HTTP/3 Prioritization Proposal

## Design Goals
* A priority scheme that can provide the appropriate scheduling without needing the full context of other streams.
* Ordering of streams (more important delivered before lower importance).
* Specifying concurrency of download for requests to allow for both sequential and concurrent transfer of streams.

## Proposed Prioritization Scheme

The prioritization will be packed into a single byte (with no other stream ID's or data required for prioritization).

```
 0               
 0 1 2 3 4 5 6 7 
+-+-+-+-+-+-+-+-+
| Priority  | C |
+-+-+-+-+-+-+-+-+
```

In all cases, prioritization of stream delivery is based on what streams have data available to send.

Priority (6 bits, 0-63): 
* Streams at a higher priority are delivered in their entirety before streams at a lower priority.
* Streams at the same priority level are delivered as defined by their concurrency.

Concurrency (2 bits, 0-4):

0. Default (unspecified)
1. "Exclusive Sequential" : Streams with a concurrency of 1 are delivered first at a given priority level and without sharing bandwidth with any other streams. This is optimal for things like blocking scripts and CSS.
2. "Shared Sequential" : Streams with a concurrency of 2 are grouped together and delivered sequentially within the group. The group as a whole splits bandwidth evenly with the streams with concurrency 3 (50% to the one stream at a time in the concurrency 2 group and 50% to the 3-concurrency group as a whole). This is optimal for things like async or deferred scripts where you may want to load them quickly but not exclusively and where they are optimally delivered completely and in order.
3. "Shared" : Streams with a concurrency of 3 are grouped together in a single group. The allocated bandwidth for the group is split evenly across all streams in the group.

When the priority and concurrency are both set to zero, it should be assumed that prioritization is not specified and default prioritization should be applied (TBD - likely priority 0 with concurrency 3).

![Priority Levels and Concurrency](images/priorities.png)

### Bandwidth Splitting
Bandwidth splitting is done at a frame-by-frame level where the next frame to be sent is evaluated as every frame of data is being prepared to be sent (assuming the data streams are chunked into frames on the server side for processing):

* Only consider responses where data is available to be sent.
* Select from the responses with the highest priority level.
* If "Concurrency 1" responses are available, fill the frame with the "Concurrency 1" response with the lowest stream ID.
* Otherwise, round robin between the "Concurrency 2" and "Concurrency 3" groups, picking one frame from each group.
* Within the "Concurrency 2" group, fill the frame with the response with the lowest stream ID.
* Within the "Concurrency 3" group, round robin across all of the available responses.

### Example Browser Prioritization

Based on Chromium's prioritization scheme:

![Sample Prioritization](images/sample.svg)