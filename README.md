# Zipkin-SkyWalking
Prepare to accept Zipkin tracr data at SkyWalking collector. 

The primary goal of Zipkin-SkyWalking bridge is mapping the spans, which collects by Zipkin SDKs and its ecosystem, to 
SkyWalking Netwok protocols

**But be attention, right now, I think there is no way to let SkyWalking agent works with Zipkin SDKs in the same traced application cluster, now. You only can choose one of them.** [W3C distributed-tracing specification](https://github.com/apache/incubator-skywalking) may help some day, still need further discussion.

* Zipkin v2: https://zipkin.io/zipkin-api/#/default/post_spans
* SkyWalking 5.x: https://github.com/apache/incubator-skywalking/tree/master/apm-protocol/apm-network/src/main/proto En documents are still missing.

# Data Models Mapping
## Trace 
A set of spans that share a single root span. TraceId maps the GlobalTraceId in SkyWalking.

## TraceSegment
No TraceSegment concept in Zipkin. Require [Trace and TraceSegment Rebuid mechanism](#trace-and-tracesegment-rebuid-mechanism).

## Span
In SkyWalking, Span is just a concept, not real. There are three sub-concept, and real classes are EntrySpan, LocalSpan and ExitSpan.

LocalSpan and ExitSpan relate RPC and MQ, mapping Zipkin's Span with RemoteEndpoint.

### TraceId, SpanId, ParentSpanId
All these three IDs are String in Zipkin, and format is 
```
maxLength: 32
minLength: 16
pattern: [a-z0-9]{16,32}
```

In SkyWalking, TraceId is a combination by three Integers. SpanId and ParentSpanId are small Integers, start with 0 in each TraceSegment. There two IDs need to be generated in [Trace and TraceSegment Rebuid](#trace-and-tracesegment-rebuid-mechanism) process.

### Span's timestamp
Epoch microseconds of the start of this span. Equal to SkyWalking Span's startTime, but the Unit of SkyWalking is milliseconds.

### Span's duration
**duration** is in microseconds. For notice, 1 millisecond is 1000 microseconds. SkyWalking has only endTime in milliseconds. Zipkin-Span timestamp + duration = SkyWalking-Span endTime.

### Span's LocalEndpoint
Application listening address, includes IPv4, IPv6 and port. usually the link local IP.

**SkyWalking Mapping**: Server identify and InstanceDiscoveryService, including register, heartbeat. 

### Span's RemoteEndpoint
RemoteEndpoint represents the peer network address and service name for a RPC or MQ(broker). In SkyWalking, service name maps ExitSpan's operationName, and ip(v4/v6) + port maps ExitSpan's peer by combining these two as a String with `:`.

### Annotations
Annotation can be considered as Log in SkyWalking, but no key. So convert it to log with the default log.key=`za` (Zipkin Annotation).

### Tags
Tags included in both Zipkin and SkyWalking. And for further analysis and aggregation. Here is the mapping table based on [Zipkin core define](https://github.com/openzipkin/zipkin-api/blob/master/thrift/zipkinCore.thrift).

| Data required by SkyWalking | Possible Keys in Zipkin |
|----|-----|
|Error occurs in this Span scope.| **error** | 
|Componenet library name. e.g. Feign, Resttemplate| **lc** |
|Layer of this Span. e.g. Database, RPCFramework, Http, MQ, Cache | Presume non, must be conjectured by Component library name or other tags|
|SQL statement| **sql.query** |
|Database Type| |
|HTTP URL| **http.url** |
|HTTP Status Code| **http.status_code** |
|HTTP method| **http.method** |
|MQ topic/queue name | No |

# Trace and TraceSegment Rebuid mechanism
## Why need rebuild mechanism
SkyWalking analysis traces by trace segment, not span. So the rebuild process is about to buffer the upload spans, rebuild them into severl segments, based on a trace segment rebuild timeout thredhold.

- The reason of need **timeout threhold**: No one can tell when, where and how a trace end, even for a same service, it changes time to time, because codes upgrade, different parameter or status data. And also this is why SkyWalking use a complex HEAD/SegmentRef to allow analysis and aggregation program didn't expect a full trace to analysis. But This is impossible for rebuilding process.

## Process flow
```
+-------------------------------+       +-------------------------------+                  +-------------------------------+
| Application under Zipkin trace|       | Zipkin-SkyWalking Rebuilder   |                  |  SkyWalking Collector         |
+-------------------------------+       +-------------------------------+                  +-------------------------------+
              |     /Spans (Zipkin v2)                  |                                                   |
              +---------------------------->    1. Rebuild segment                                          |
                                with same traceid and applicationCode(Requirement¹)                         |
                                            2. Buffer unfinished segment                                    |
                                                    into files                                              |
                                                        |   SkyWalking gRPC services                        |
                                                        +------------------------------------>  Analysis and Aggregation
```

## Requirement
An additional **Tag**, key is `app.code`. Every application under traced should be an explicit application code, in order to show in SkyWalking Topology, Application list etc.

## Rebuild steps
