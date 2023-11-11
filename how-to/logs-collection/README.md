# Logs collection using OpenTelemetry

In the following, we will walk through how to do logs collection with
OpenTelemetry (OTel). To keep things simple, we will use Python as the demonstration
programming language, however note that at time of writing the logs support
there is still early days so things might need some updating.

We will show the evolution from using print statements for logging 
(Baby Grogu level) to logging to a file along with the OTel collector (Expert 
Grogu level) to using the OTel logs bridge API to directly ingest OTLP (Yoda level).

## Baby Grogu level

First, change into the [`baby-grogu/`][repo-baby-grogu] directory.

We're using the Python code in `baby-grogu/main.py` as an example, with the
interesting part located in the `practice()` function:

```python
start_time = time.time()
try:
    how_long_int = int(how_long)
    print(f"Starting to practice The Telemetry for {how_long_int} second(s)")
    while time.time() - start_time < how_long_int:
        next_char = random.choice(string.punctuation)
        print(next_char, end="", flush=True)
        time.sleep(0.5)
    print("\nDone practicing")
except ValueError as ve:
    print(f"I need an integer value for the time to practice: {ve}")
    return False
except Exception as e:
    print(f"An unexpected error occurred: {e}")
    return False
return True
```

The above Python code doesn't really do anything useful, just printing out
random punctuation for the specified time, however, notice the different 
semantics of the `print()` function. For example, when we say
`print(next_char, end="", flush=True)` we're actually performing work whereas
when we say `print("\nDone practicing")` that's just an informational message
that the work is done. This would be a great candidate for a log message!
The same is true for `print(f"I need an integer value for the time to practice: {ve}")`
which is really an error log line.

To execute you can either directly run it with `python3 main.py 3` to have
baby Grogu practice for 3 seconds, or you can use a containerized version.

For this, we're using the following `Dockerfile`:

```yaml
FROM python:3
WORKDIR /usr/src/app
COPY . .

```
In the context of the following Docker Compose file:

```yaml
version: '3'
services:
  baby-grogu:
    build: .
    command: python main.py 3
    volumes:
    - .:/usr/src/app
```

And now you can run it with `docker-compose -f docker-compose.yaml` and you
should see something like:

```shell
[+] Running 2/0
 ✔ Network baby-grogu_default         Created                               0.0s
 ✔ Container baby-grogu-baby-grogu-1  Created                               0.0s
Attaching to baby-grogu-baby-grogu-1
baby-grogu-baby-grogu-1  | Starting to practice The Telemetry for 2 second(s)
baby-grogu-baby-grogu-1  | /)||
baby-grogu-baby-grogu-1  | Done practicing
baby-grogu-baby-grogu-1  | Practicing The Telemetry completed: True
baby-grogu-baby-grogu-1 exited with code 0
```

Now let's up the game and use OTel!

## Expert Grogu level

First, change into the [`expert-grogu/`][repo-expert-grogu] directory.

In this scenario we're logging to a file (in JSON format) from the Python app.
Then, we're using the OTel collector to read and parse the logs file content using
the [filelog receiver][filelog].

Overall, we have the following setup:

```
( python main.py ) --> exgru.log --> ( OTel collector ) --> stdout
```

We are using the following `Dockerfile` (installing the one dependecy we have,
`python-json-logger==2.0.7`):

```
FROM python:3
WORKDIR /usr/src/app
COPY requirements.txt requirements.txt
RUN pip3 install --no-cache-dir -r requirements.txt
COPY . .
```

With the following OTel collector config:

```yaml
receivers:
  filelog:
    include: [ /usr/src/app/*.log ]
    operators:
      - type: json_parser
        timestamp:
          parse_from: attributes.asctime
          layout: '%Y-%m-%dT%H:%M:%S'
        severity:
          parse_from: attributes.levelname
exporters:
  logging:
    verbosity: detailed
service:
  pipelines:
    logs:
      receivers: [ filelog ]
      exporters: [ logging ]
```

In the Docker Compose file that looks as follows, we bring all above together:

```yaml
version: '3'
services:
  collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
    - ./otel-config.yaml:/etc/otelcol-contrib/config.yaml
    - ./:/usr/src/app
    command: ["--config=/etc/otelcol-contrib/config.yaml"]
    ports:
    - "4317:4317"
  baby-grogu:
    build: .
    command: python main.py 10
    volumes:
    - .:/usr/src/app
```

Which you can run it with `docker-compose -f docker-compose.yaml` and you
should see something like:

```shell
Attaching to expert-grogu-baby-grogu-1, expert-grogu-collector-1
expert-grogu-collector-1   | 2023-11-11T15:01:41.581Z   info    LogsExporter    {"kind": "exporter", "data_type": "logs", "name": "logging", "resource logs": 1, "log records": 1}
expert-grogu-collector-1   | 2023-11-11T15:01:41.581Z   info    ResourceLog #0
expert-grogu-collector-1   | Resource SchemaURL:
expert-grogu-collector-1   | ScopeLogs #0
expert-grogu-collector-1   | ScopeLogs SchemaURL:
expert-grogu-collector-1   | InstrumentationScope
expert-grogu-collector-1   | LogRecord #0
expert-grogu-collector-1   | ObservedTimestamp: 2023-11-11 15:01:41.485663255 +0000 UTC
expert-grogu-collector-1   | Timestamp: 2023-11-11 15:01:41 +0000 UTC
expert-grogu-collector-1   | SeverityText: INFO
expert-grogu-collector-1   | SeverityNumber: Info(9)
expert-grogu-collector-1   | Body: Str({"asctime": "2023-11-11T15:01:41", "levelname": "INFO", "message": "Starting to practice The Telemetry for 10 second(s)", "taskName": null})
expert-grogu-collector-1   | Attributes:
expert-grogu-collector-1   |      -> log.file.name: Str(exgru.log)
expert-grogu-collector-1   |      -> asctime: Str(2023-11-11T15:01:41)
expert-grogu-collector-1   |      -> levelname: Str(INFO)
expert-grogu-collector-1   |      -> message: Str(Starting to practice The Telemetry for 10 second(s))
expert-grogu-collector-1   |      -> taskName: Str(<nil>)
expert-grogu-collector-1   | Trace ID:
expert-grogu-collector-1   | Span ID:
expert-grogu-collector-1   | Flags: 0
expert-grogu-collector-1   |    {"kind": "exporter", "data_type": "logs", "name": "logging"}
expert-grogu-baby-grogu-1  | <(=~#`!`_<\&*|~:):.^
expert-grogu-baby-grogu-1 exited with code 0
expert-grogu-collector-1   | 2023-11-11T15:01:51.581Z   info    LogsExporter    {"kind": "exporter", "data_type": "logs", "name": "logging", "resource logs": 1, "log records": 2}
expert-grogu-collector-1   | 2023-11-11T15:01:51.581Z   info    ResourceLog #0
expert-grogu-collector-1   | Resource SchemaURL:
expert-grogu-collector-1   | ScopeLogs #0
expert-grogu-collector-1   | ScopeLogs SchemaURL:
expert-grogu-collector-1   | InstrumentationScope
expert-grogu-collector-1   | LogRecord #0
expert-grogu-collector-1   | ObservedTimestamp: 2023-11-11 15:01:51.484268468 +0000 UTC
expert-grogu-collector-1   | Timestamp: 2023-11-11 15:01:51 +0000 UTC
expert-grogu-collector-1   | SeverityText: INFO
expert-grogu-collector-1   | SeverityNumber: Info(9)
expert-grogu-collector-1   | Body: Str({"asctime": "2023-11-11T15:01:51", "levelname": "INFO", "message": "Done practicing", "taskName": null})
expert-grogu-collector-1   | Attributes:
expert-grogu-collector-1   |      -> log.file.name: Str(exgru.log)
expert-grogu-collector-1   |      -> asctime: Str(2023-11-11T15:01:51)
expert-grogu-collector-1   |      -> levelname: Str(INFO)
expert-grogu-collector-1   |      -> message: Str(Done practicing)
expert-grogu-collector-1   |      -> taskName: Str(<nil>)
expert-grogu-collector-1   | Trace ID:
expert-grogu-collector-1   | Span ID:
expert-grogu-collector-1   | Flags: 0
expert-grogu-collector-1   | LogRecord #1
expert-grogu-collector-1   | ObservedTimestamp: 2023-11-11 15:01:51.48437876 +0000 UTC
expert-grogu-collector-1   | Timestamp: 2023-11-11 15:01:51 +0000 UTC
expert-grogu-collector-1   | SeverityText: INFO
expert-grogu-collector-1   | SeverityNumber: Info(9)
expert-grogu-collector-1   | Body: Str({"asctime": "2023-11-11T15:01:51", "levelname": "INFO", "message": "Practicing The Telemetry completed: True", "taskName": null})
expert-grogu-collector-1   | Attributes:
expert-grogu-collector-1   |      -> log.file.name: Str(exgru.log)
expert-grogu-collector-1   |      -> asctime: Str(2023-11-11T15:01:51)
expert-grogu-collector-1   |      -> levelname: Str(INFO)
expert-grogu-collector-1   |      -> message: Str(Practicing The Telemetry completed: True)
expert-grogu-collector-1   |      -> taskName: Str(<nil>)
expert-grogu-collector-1   | Trace ID:
expert-grogu-collector-1   | Span ID:
expert-grogu-collector-1   | Flags: 0
expert-grogu-collector-1   |    {"kind": "exporter", "data_type": "logs", "name": "logging"}
```

## Yoda level

First, change into the [`yoda/`][repo-yoda] directory.

In this scenario we're using the OTel logs bridge API in the Python app to directly 
ingest logs in OTLP format into the OTel collector.

Overall, we have the following setup:

```
( python main.py ) - OTLP -> ( OTel collector ) --> stdout
```

With the following OTel collector config:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
exporters:
  logging:
    verbosity: detailed
service:
  pipelines:
    logs:
      receivers: [ otlp ]
      exporters: [ logging ]
```

Now run the setup with `docker-compose -f docker-compose.yaml` and you
should see something akin to below:

```shell
Attaching to yoda-baby-grogu-1, yoda-collector-1
yoda-collector-1   | 2023-11-11T15:10:31.135Z   info    service@v0.88.0/telemetry.go:84 Setting up own telemetry...
yoda-collector-1   | 2023-11-11T15:10:31.135Z   info    service@v0.88.0/telemetry.go:201        Serving Prometheus metrics      {"address": ":8888", "level": "Basic"}
yoda-collector-1   | 2023-11-11T15:10:31.135Z   info    exporter@v0.88.0/exporter.go:275        Deprecated component. Will be removed in future releases.       {"kind": "exporter", "data_type": "logs", "name": "logging"}
yoda-collector-1   | 2023-11-11T15:10:31.136Z   info    service@v0.88.0/service.go:143  Starting otelcol-contrib...     {"Version": "0.88.0", "NumCPU": 4}
yoda-collector-1   | 2023-11-11T15:10:31.136Z   info    extensions/extensions.go:33     Starting extensions...
yoda-collector-1   | 2023-11-11T15:10:31.136Z   warn    internal@v0.88.0/warning.go:40  Using the 0.0.0.0 address exposes this server to every network interface, which may facilitate Denial of Service attacks        {"kind": "receiver", "name": "otlp", "data_type": "logs", "documentation": "https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/security-best-practices.md#safeguards-against-denial-of-service-attacks"}
yoda-collector-1   | 2023-11-11T15:10:31.136Z   info    otlpreceiver@v0.88.0/otlp.go:83 Starting GRPC server    {"kind": "receiver", "name": "otlp", "data_type": "logs", "endpoint": "0.0.0.0:4317"}
yoda-collector-1   | 2023-11-11T15:10:31.136Z   info    service@v0.88.0/service.go:169  Everything is ready. Begin running and processing data.
yoda-baby-grogu-1  | #}>&;!|.<:;++:{.?[>\
yoda-baby-grogu-1 exited with code 0
```

May The Telemetry be with you, young Padawan!

## References

* [OpenTelemetry Logging][otel-log-spec]
* [A language-specific implementation of OpenTelemetry in Python][otel-python]
  (OTel docs)
* [OpenTelemetry Logging Instrumentation][py-docs-logging] (Python docs)
* [OpenTelemetry Logs SDK example][py-docs-logging-example] (Python docs)


[repo-baby-grogu]: https://github.com/mhausenblas/ref.otel.help/tree/main/how-to/logs-collection/baby-grogu/
[repo-expert-grogu]: https://github.com/mhausenblas/ref.otel.help/tree/main/how-to/logs-collection/expert-grogu/
[repo-yoda]: https://github.com/mhausenblas/ref.otel.help/tree/main/how-to/logs-collection/yoda/
[filelog]: https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver
[otel-log-spec]: https://opentelemetry.io/docs/specs/otel/logs/
[otel-python]: https://opentelemetry.io/docs/instrumentation/python/
[py-docs-logging]: https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/logging/logging.html
[py-docs-logging-example]: https://opentelemetry-python.readthedocs.io/en/latest/examples/logs/README.html

