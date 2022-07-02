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

The OTel collector provides built-in mechanism for monitoring and troubleshooting. It provides rich telemetry, configurable via the [`service` section in the config][otelcol-config-service]:

```yaml
service:
  telemetry:
    logs:
      level: debug
    metrics:
      level: detailed
      address: 0.0.0.0:8888

...
exporters:
  logging:
    loglevel: debug
...
```

### Logs

### Metrics

You can get metrics in Prometheus exposition format, for example, in a Kubernetes setup you can forward traffic locally:

```shell
$ kubectl port-forward svc/adot-collector 8888:8888   
```

In a different session:

```shell
$ curl localhost:8888/metrics
# HELP otelcol_exporter_enqueue_failed_log_records Number of log records failed to be added to the sending queue.
# TYPE otelcol_exporter_enqueue_failed_log_records counter
otelcol_exporter_enqueue_failed_log_records{exporter="awsemf",service_instance_id="f4e35993-b0df-4bd4-80ab-554c847f6156",service_version="latest"} 0
otelcol_exporter_enqueue_failed_log_records{exporter="awsprometheusremotewrite",service_instance_id="f4e35993-b0df-4bd4-80ab-554c847f6156",service_version="latest"} 0
otelcol_exporter_enqueue_failed_log_records{exporter="awsxray",service_instance_id="f4e35993-b0df-4bd4-80ab-554c847f6156",service_version="latest"} 0
otelcol_exporter_enqueue_failed_log_records{exporter="logging",service_instance_id="f4e35993-b0df-4bd4-80ab-554c847f6156",service_version="latest"} 0
...
```

If you use Prometheus to scrape the collector and visualize them in Grafana, an [example dashboard][example-dashboard] might look as follows:

![Screen shot of Grafana dashboard for OTel collector monitoring](example-otel-collector-dashboard.png)

### Profiling

You can consume the `pprof` formatted profiles with CP like Parca:

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
[otelcol-config-service]: https://opentelemetry.io/docs/collector/configuration/#service
[example-dashboard]: otel-collector-dashboard.json
[otel-distro-main]: https://opentelemetry.io/docs/collector/distributions/
[otel-distro-prebuilt]: https://github.com/open-telemetry/opentelemetry-collector-releases/releases
[otel-distro-aws]: https://github.com/aws-observability/aws-otel-collector
[otel-distro-builder]: https://github.com/open-telemetry/opentelemetry-collector/tree/main/cmd/builder
[opamp]: https://github.com/open-telemetry/opamp-spec
[otel-beyond-gettingstarted]: https://medium.com/opentelemetry/opentelemetry-beyond-getting-started-5ac43cd0fe26
[containiq-col]: https://www.containiq.com/post/opentelemetry-collector-and-exporters
