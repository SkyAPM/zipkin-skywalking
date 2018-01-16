# Zipkin-SkyWalking
Prepare to accept Zipkin tracr data at SkyWalking collector. 

The primary goal of Zipkin-SkyWalking bridge is mapping the spans, which collects by Zipkin SDKs and its ecosystem, to 
SkyWalking Netwok protocols

**But be attention, right now, I think there is no way to let SkyWalking agent works with Zipkin SDKs in the same traced application cluster, now. You only can choose one of them.**

* Zipkin v2: https://zipkin.io/zipkin-api/#/default/post_spans
* SkyWalking 5.x: https://github.com/apache/incubator-skywalking/tree/master/apm-protocol/apm-network/src/main/proto En documents are still missing.

# Data Models Mapping
## Trace 
A set of spans that share a single root span. TraceId maps the GlobalTraceId in SkyWalking.

## Span
In SkyWalking, Span is just a concept, not real. There are three sub-concept, and real classes are EntrySpan, LocalSpan and ExitSpan.

LocalSpan and ExitSpan relate RPC and MQ, mapping Zipkin's Span with RemoteEndpoint.

### Span's duration
**duration** is in microseconds, and SkyWalking is in milliseconds. For notice, 1 millisecond is 1000 microseconds.

### Span's LocalEndpoint
Application listening address, includes IPv4, IPv6 and port. usually the link local IP

**SkyWalking Mapping**: Server identify and InstanceDiscoveryService, including register, heartbeat. 
