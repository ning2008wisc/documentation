---
title: OpenTelemetry and Jaeger
description: Integration between Jaeger and OpenTelemetry
weight: 11
---

From the [recent Jaeger blog](https://medium.com/jaegertracing/jaeger-and-opentelemetry-1846f701d9f2), Jaeger planned to collaborate with OpenTelemetry (a.k.a, OTEL). With the most recent Jaeger release, OpenTelemetry Collector can run in the roles of Jaeger Agent, Collector and Ingester out-of-the-box. More practically, [OpenTelemetry-Collector](https://github.com/open-telemetry/opentelemetry-collector) has an implementation of Tail-based Sampling and this page will describe how to scale it in Jaeger ecosystem without losing the correctness of Tail-based Sampling.

## Jaeger OpenTelemetry Agent, Collector and Ingester

All of the three new OTEL-based components are practically based from the same binary of OTEL-Collector. By default, they can be the drop-in replacements of the current Jaeger Agent, Collector and Ingester respectively for most use cases. For example, the ports and flags that work with Jaeger components are still valid in their OTEL counterparts.


## Scale Tail-based Sampling in OpenTelemetry Collector

OTEL-Collector already implemented [a version of tail-based sampling](https://github.com/open-telemetry/opentelemetry-collector/tree/master/processor/samplingprocessor/tailsamplingprocessor), however it only works in a single-node mode.

The fundamental challenge is: For high availability and scalability, multiple OTEL-Collector instances need to be deployed. The spans, reported by different clients, with the same `Trace ID` could be received and processed by different OTEL-Collector instance, unless the load balancing of OTEL-Collector is Trace ID “aware", meaning that the spans with same Trace ID will be always routed to the same OTEL-Collector instance.

Without changing the existing OTEL-Collector implementation, the following deployment can scale OTEL-Collector by solving the above challenge.

![Architecture](/img/otel-architecture-tail-sampling-v1.png)
*Illustration of OpenTelemetry Collector and Agent deployment with Kafka in Jaeger ecosystem*

The reasons why it scales correctly: (1) in  [Jaeger-OTEL-Agent](https://github.com/jaegertracing/jaeger/tree/master/cmd/opentelemetry-collector/cmd/agent), the [Kafka OTEL exporter](https://github.com/jaegertracing/jaeger/tree/master/cmd/opentelemetry-collector/app/exporter/kafka) sends spans to a Kafka topic keyed on `Trace ID`, so the spans with the same `Trace ID` will always be seen on one partition of Kafka topic. (2) Multiple [Jaeger-OTEL-Collector](https://github.com/jaegertracing/jaeger/tree/master/cmd/opentelemetry-collector/cmd/collector) instances, or [Jaeger-OTEL-Ingester](https://github.com/jaegertracing/jaeger/tree/master/cmd/opentelemetry-collector/cmd/ingester) with tail-based sampling processor, will collectively use the same consumer group when consuming from the topic by [Kafka OTEL receiver](https://github.com/jaegertracing/jaeger/tree/master/cmd/opentelemetry-collector/app/receiver/kafka). Therefore all spans with the same `Trace ID` will only be processed by exactly one OTEL tail-based sampling processor.

Kafka itself is a highly available, fault-tolerant messaging system. Both the Jaeger-OTEL-Agent and Jaeger-OTEL-Collector can scale horizontally. Thus there is no Single Point of Failure. Kafka here serves two purposes: (1) A scalable and fault-tolerant buffer to temporarily stage the data with tunable data retention, (2) a way of sharding by taking advantaging of: (a) messages with same key will be sent to the same partition. (b) each partition is only consumed by one consumer thread in a consumer group.


## Alternative Design and Trade-off

The above design has the chance of spans with the same `Trace ID` showing up on more than one instances of Jaeger-OTEL-Collector, when Kafka is rebalancing possibly caused by broker failure. This is because when rebalancing, the spans with the same Trace ID may be temporarily produced to a different partition. After the failed broker recovers and joins back to Kafka, the spans will show up again on the original partition.

During the rebalancing and recovery phases, it may break the original statement “all spans with the same Trace ID will always be on the same partition”, so multiple Tail-based Sampling processors may process the spans with the same `Trace ID` for a short period of time, leading to incomplete or incorrect sampling.

An improvement or alternative to solve the above issue is: each Jaeger-OTEL-Agent instance sends spans to a differently named topic, and each Jaeger-OTEL-Collector instance consumes from a different topic, so the spans with the same `Trace ID` will always be on one topic. But the downside is to maintain and monitor multiple Kafka topics.