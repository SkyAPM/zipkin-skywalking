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
Tags included in both Zipkin and SkyWalking. And for further analysis and aggregation.

| Data required by SkyWalking | Possible Keys in Zipkin |
|----|-----|
|Error occurs in this Span scope.|  | 
|Componenet library name. e.g. Feign, Resttemplate|  |
|Layer of this Span. e.g. Database, RPCFramework, Http, MQ, Cache | Presume non, must be conjectured by Component library name.|
|SQL statement| **sql.query** |


# Trace and TraceSegment Rebuid mechanism
