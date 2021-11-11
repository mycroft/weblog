---
title: "Testing Jaeger With Opentelemetry"
date: 2021-11-09T10:27:04+01:00
categories:
  - opentelemetry
summary: "Testing Python & Golang opentelemetry & send traces to Jaeger"
---

This posts is a quick reminder of my testing of opentelemetry SDKs (Python & Golang implementations), working with traces (span, events propagation).

It covers:

- Running a Jaeger instance;
- Sending traces using a short Python debug script;
- Writing logs lines with opentelemetry correlation;
- Working with http backend/frontend processes;

# Starting Jaeger

[Jaeger](https://www.jaegertracing.io) is a distributed trace collector, ingestor & UI to search & view the traces. It uses Cassandra (or scylladb) or Elasticsearch for trace persistence.

It comes with an all-in-one docker image that will run all the mandatory components (ie: both collector, API, UI & storage) for development & testing purposes.

## Running the all-in-one Jaeger docker image

Starting the image is pretty much straightforward (as long as you don't forget to expose ports), and the following should work out of the box:

```sh
podman run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.27
```

(Note I'm using podman because fedora, but you're free to use your prefered container runtime tech.).

Once started, Jaeger should be available on localhost, on UI port 16686: [http://localhost:16686](http://localhost:16686).

All ports are not required for what you want to do, so you can shrink the list a bit. Most importants ones in my opinion are 6831, 6832 (udp, to accept traces in jaeger.thrift formats), 5778 http port to serve config, 16686 http port to serve UI & finally 14268 to accept jaeger.thrift formats using http.

# First Opentelemetry Python SDK usage

I'll use the Python SDK to send a single trace. Before send it to jaeger, the following script will send it to the console for debug purpose.

The snippet comes from the [official documentation page](https://opentelemetry-python.readthedocs.io/en/latest/getting-started.html), with some inline comments of mine.

```python
# trace is the base component that will use a tracer to control the execution context of tracing.
# it must be given a tracer API (here TracerProvider), that will make use of a processor & an exporter
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
    ConsoleSpanExporter,
)

# Create the provider, processor, exporter, and initialize the execution context
provider = TracerProvider()
processor = BatchSpanProcessor(ConsoleSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)


# Retrieve the instanciated API
tracer = trace.get_tracer(__name__)

# Create a span creating a span creating a span.
with tracer.start_as_current_span("foo"):
    with tracer.start_as_current_span("bar"):
        with tracer.start_as_current_span("baz"):
            print("Hello world from OpenTelemetry Python!")
```

At this point, the script should run as it. Don't forget to initialize a virtual environment & install the required deps or, well, it might not work:

```sh
$ virtualenv .venv
$ source .venv/bin/activate
$ pip install opentelemetry-api opentelemetry-sdk
$ python test1.py
Hello world from OpenTelemetry Python!
{
    "name": "baz",
    "context": {
        "trace_id": "0x9677d965376d21835a4b03d2dc226920",
        "span_id": "0x1283356771d6ab84",
        "trace_state": "[]"
    },
    "kind": "SpanKind.INTERNAL",
    "parent_id": "0x6f28074b7bf2ac3e",
    "start_time": "2021-11-09T09:50:34.541857Z",
    "end_time": "2021-11-09T09:50:34.541882Z",
    "status": {
        "status_code": "UNSET"
    },
    "attributes": {},
    "events": [],
    "links": [],
    "resource": {
        "telemetry.sdk.language": "python",
        "telemetry.sdk.name": "opentelemetry",
        "telemetry.sdk.version": "1.6.2",
        "service.name": "unknown_service"
    }
}
...
```

You should see the `print` message, then on exit, the console dump of the trace spans. As we defined three spans (`foo`, `bar`, `baz`), you'll have three spans as well.
Each spans has several information, as `trace_id` (should be unique during the whole `tracer` execution process, multiple `span_id`s, kind, start/end times, status, etc.) Some of those information are generated during execution, others can be changed (like `status`, `events`...).

This said, next step is to send traces to Jaeger. We modify the above script a little bit to add more processing information, like a service name, or modify a bit spans during execution process, by adding some error happening (without any error message yet).

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.trace import Status, StatusCode

# Create a provider by providing a service name.
trace.set_tracer_provider(
    TracerProvider(
        resource=Resource.create({SERVICE_NAME: "testaroo-service"})
    )
)

# Create and use a JaegerExporter (no longer use the ConsoleSpanExporter anymore)
exporter = JaegerExporter(
    collector_endpoint="http://localhost:14268/api/traces"
)

trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(exporter)
)

# Retrieve the instanciated API
tracer = trace.get_tracer(__name__)

# Create a span creating a span creating a span.
with tracer.start_as_current_span("foo"):
    with tracer.start_as_current_span("bar") as span:
        with tracer.start_as_current_span("error_baz") as span:
            # for some reason, this span had some bad stuff happening
            span.set_status(Status(StatusCode.ERROR))
        with tracer.start_as_current_span("baz"):
            print("Hello world from OpenTelemetry Python!")
```


Before running, don't forget to install the new required dependancy `opentelemetry-exporter-jaeger`.

```sh
$ pip install opentelemetry-exporter-jaeger
$ python test.py
Hello world from OpenTelemetry Python!
$
```

The script is no longer very talkative as the trace is no longer sent to the console. Instead, the trace appeared in Jaeger:

![First trace in Jaeger](images/jaeger1.png)

You can also see the span marked as errored.

In a second part, I'll add more information in spans, as events/logs, add some logs & make sure logs are correctly correlated with traces, and in a final part, test trace propagation between a python frontend & a golang backend.

In this part, you found out how to start jaeger in development mode, and send your first traces using the Opentelemetry Python SDK.

References:

* https://opentelemetry-python.readthedocs.io/en/stable/index.html
* https://github.com/open-telemetry/opentelemetry-specification/tree/main/specification/trace
