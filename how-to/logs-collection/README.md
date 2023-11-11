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

Logging to a file along with the OTel collector to read and parse its content using
the [filelog receiver][filelog].

## Yoda level

Using the OTel logs bridge API to directly ingest OTLP (Yoda level)

## References

* [OpenTelemetry Logging][otel-log-spec]
* [A language-specific implementation of OpenTelemetry in Python][otel-python]
  (OTel docs)
* [OpenTelemetry Logging Instrumentation][py-docs-logging] (Python docs)
* [OpenTelemetry Logs SDK example][py-docs-logging-example] (Python docs)

[repo-baby-grogu]: https://github.com/mhausenblas/ref.otel.help/tree/main/how-to/logs-collection/baby-grogu
[filelog]: https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver
[otel-log-spec]: https://opentelemetry.io/docs/specs/otel/logs/
[otel-python]: https://opentelemetry.io/docs/instrumentation/python/
[py-docs-logging]: https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/logging/logging.html
[py-docs-logging-example]: https://opentelemetry-python.readthedocs.io/en/latest/examples/logs/README.html

