# Good practices operating OpenTelemetry collectors

> Author: Michael Hausenblas
>
> Contact: `mh9@o11y.engineering`
>
> Last update: 2022-07-02
> 
> This document describes good practices operating [OpenTelemetry collectors][otelcol]. 
> The target audience for this document includes platform operators and devops/SRE roles 
> that deploy OTel collectors for the telemetry of their workloads, such as containerized
> microservices or on-premises monoliths. 
>
> I assume you have a basic familiarity with OpenTelemetry and cloud native environments
> like Kubernetes or AWS Lambda. Corrections or suggestions are welcome!


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

extensions:
  pprof:
    endpoint: localhost:1777
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

You can consume the `pprof` formatted profiles the OTel collector exposes with one of the Continuos Profiling tools (e.g. CNCF Pixie, Parca, Pyroscope).

Using [Parca][parca-bin] with the following configuration stored in a file called `parca.yaml`:

```yaml
debug_info:
  bucket:
    type: "FILESYSTEM"
    config:
      directory: "./tmp"
  cache:
    type: "FILESYSTEM"
    config:
      directory: "./tmp"

scrape_configs:
  - job_name: "default"
    scrape_interval: "2s"
    static_configs:
      - targets: ["127.0.0.1:1777"]
```

You can now run Parca like so (assuming you've done a port-forward for the OTel collector on port `1777` to the machine where you run Parca):


```shell
$ parca --config-path="parca.yaml"
ooooooooo.
`888   `Y88.
 888   .d88'  .oooo.   oooo d8b  .ooooo.   .oooo.
 888ooo88P'  `P  )88b  `888""8P d88' `"Y8 `P  )88b
 888          .oP"888   888     888        .oP"888
 888         d8(  888   888     888   .o8 d8(  888
o888o        `Y888""8o d888b    `Y8bod8P' `Y888""8o

level=info name=parca ts=2022-05-24T13:43:18.645322Z caller=badger.go:100 msg="Set nextTxnTs to 0"
level=info name=parca ts=2022-05-24T13:43:18.64803Z caller=factory.go:49 component=debuginfod msg="loading bucket configuration"
level=info name=parca ts=2022-05-24T13:43:18.648642Z caller=factory.go:49 msg="loading bucket configuration"
level=info name=parca ts=2022-05-24T13:43:18.649058Z caller=server.go:92 msg="starting server" addr=:7070
...

```

When you now open `localhost:7070` in your browser of choice, you should see something like:

![The ADOT collector profiles in Parca](adot-col-in-parca.png)


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
[parca-bin]: https://www.parca.dev/docs/binary
[opamp]: https://github.com/open-telemetry/opamp-spec
[otel-beyond-gettingstarted]: https://medium.com/opentelemetry/opentelemetry-beyond-getting-started-5ac43cd0fe26
[containiq-col]: https://www.containiq.com/post/opentelemetry-collector-and-exporters
