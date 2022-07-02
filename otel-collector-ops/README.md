# Good practices operating OpenTelemetry collectors

> Author: Michael Hausenblas
>
> Contact: `mh9@o11y.engineering`

This document describes good practices operating [OpenTelemetry collectors][otelcol]. The target audience are platform operators and devops/SRE roles that deploy OTel collectors for the telemetry of their workloads such as containerized microservices. I assume you have a basic familiarity with OpenTelemetry and cloud native environments including but not limited to Kubernetes and AWS Lambda.

--- 

## Introduction

TBD.

## Distributions

When it comes to [distributions][otel-distro-main] you have three options:

1. Use the upstream, [pre-built distro][otel-distro-prebuilt]
1. Choose a vendor distro like [ADOT][otel-distro-aws]
1. Roll your own, using `ocb`, the [OpenTelemetry Collector Builder][otel-distro-builder]

Let's have a look at pros and cons for each option:

## Health and Performance Monitoring

An [example dashboard][example-dashboard] might look as follows:

![Screen shot of Grafana dashboard for OTel collector monitoring](example-otel-collector-dashboard.png)

## Statefulness

Consider running the collector stateless.

## Security

### VMs

### Kubernetes

## Further references

* [Open Agent Management Protocol (OpAMP) specification][opamp]
* [OpenTelemetry: beyond getting started][otel-beyond-gettingstarted] by _Sergey Kanzhelev_, 01/2020
* [OpenTelemetry Collector & Exporters][containiq-col]


[otelcol]: https://opentelemetry.io/docs/collector/
[example-dashboard]: otel-collector-dashboard.json
[otel-distro-main]: https://opentelemetry.io/docs/collector/distributions/
[otel-distro-prebuilt]: https://github.com/open-telemetry/opentelemetry-collector-releases/releases
[otel-distro-aws]: https://github.com/aws-observability/aws-otel-collector
[otel-distro-builder]: https://github.com/open-telemetry/opentelemetry-collector/tree/main/cmd/builder
[opamp]: https://github.com/open-telemetry/opamp-spec
[otel-beyond-gettingstarted]: https://medium.com/opentelemetry/opentelemetry-beyond-getting-started-5ac43cd0fe26
[containiq-col]: https://www.containiq.com/post/opentelemetry-collector-and-exporters
