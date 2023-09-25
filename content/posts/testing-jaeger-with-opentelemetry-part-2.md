---
title: "Testing Jaeger With Opentelemetry Part 2"
date: 2021-11-09T12:05:15+01:00
summary: "Using Opentelemetry logging, extending spans, propagating traces in a multiple component project"
tags:
  - opentelemetry
---

In this post, I'll show how to customize spans, by adding logs & events & propagate traces in a multiple components application.

This is a follow-up of the [Testing Jaeger with Opentelemetry]({{< ref "testing-jaeger-with-opentelemetry" >}}) post.


# Adding logs

Strictly speaking, logs are no part of traces. However, thanks to Opentelemetry, you can add the relevant information to correlate logs to traces. By just initializing otel's logging instrumentalization, logs will contain trace ids, making correlation much easier.

To achieve this, after adding the required `opentelemetry-instrumentation-logging` dependency to your project, just initialize the LoggingInstrumentor:

```python
...

from opentelemetry.instrumentation.logging import LoggingInstrumentor
import logging

LoggingInstrumentor().instrument(
    set_logging_format=True,
    log_level=logging.DEBUG
)
logger = logging.getLogger(__name__)

...

# Adding some log in some spans:
logger.info("First span will start")
with tracer.start_as_current_span("foo"):
    logger.info("In the first span!")

    with tracer.start_as_current_span("bar") as span:
        logger.info("In the second span!")

        with tracer.start_as_current_span("error_baz") as span:
            # for some reason, this span had some bad stuff happening
            span.set_status(Status(StatusCode.ERROR))
        with tracer.start_as_current_span("baz"):
            print("Hello world from OpenTelemetry Python!")

logger.info("end of the trace")

```

The result after the change, in the console:

```sh
2021-11-09 13:02:16,035 INFO [__main__] [test2.py:39] [trace_id=0 span_id=0 resource.service.name=testaroo-service] - First span will start
2021-11-09 13:02:16,052 INFO [__main__] [test2.py:41] [trace_id=3db23ca07ef0f16d870e5ffe549a803a span_id=6c700be9e46bb6be resource.service.name=testaroo-service] - In the first span!
2021-11-09 13:02:16,052 INFO [__main__] [test2.py:44] [trace_id=3db23ca07ef0f16d870e5ffe549a803a span_id=16f3a5e3e285bc7a resource.service.name=testaroo-service] - In the second span!
Hello world from OpenTelemetry Python!
2021-11-09 13:02:16,053 INFO [__main__] [test2.py:51] [trace_id=0 span_id=0 resource.service.name=testaroo-service] - end of the trace
```

In my sample, the trace id is `3db23ca07ef0f16d870e5ffe549a803a` (aka `3db23ca`), which correlates to the trace collected by Jaeger, as shown in the UI:

![Trace during log test without any log](images/jaeger2.png)


However, those are log lines. While they contain the `trace_id` of the trace, they will be no part of traces.

# No logs in trace, here come events

As said, there is no log strictly speaking in traces. However, as we already seen, we can set a status to traces using `set_status`, and it is also possible to add `events` in span, which can be used as logs to describe how spans are behaving.

To do so, we'll make use of the `add_event` function in spans. An event behaves as a log message, and some attributes can be added to it. Events will be part of the span they are added, and can be viewed in Jaeger UI. Attributes/values can also be searched, which make it easy to find out when something failed.

For example, by adding some code that change our testing program, we can generate & track some unexpected behavior:

```python
...
import random
...

# In the span contexts...

    with tracer.start_as_current_span("bar") as span:
        logger.info("In the second span!")
        random_number = random.randint(1,10)
        if random_number % 2 == 0:
            message = f"Picked up number is even"
        else:
            message = f"Picked up number is odd"

        span.add_event(
            message,
            attributes={
                'random_number': random_number,
            }
        )

        if random_number == 7:
            span.set_status(Status(StatusCode.ERROR))

...
```

After a few executions, to get the `random_number==7`, we can track the trace with this event & then take a deeper look at it:

![Trace with search](images/jaeger_search.png)

Details of the found trace:

![Trace with search](images/jaeger_error.png)

We found out our trace with the event matching the `random_number=7` filter that generated an error.


# Propagating traces between requests

Now that we master traces & spans, the most useful mechanism is to propagate traces during a multiple component workflow. The goal is to make easier developers & sysops to identify where errors happen or what component is unexpectedly slow.

To illustrate this, I'll start with 2 small webapps, a frontend & a backend, both running flask (as the Opentelemetry integration for flask is very simple to set-up), and generate traces involving both components.

## The Backend

Let's start by creating our backend. It is very small, as it will respond to `/hello` with a `name` parameter, and will write "Hello <name>". The biggest part of the backend will be actually to startup trace instrumentalization, which doesn't differ much from previous samples.

The backend code:

```python
import flask
import requests
from flask import request

from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
    ConsoleSpanExporter,
)
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.trace import Status, StatusCode

trace.set_tracer_provider(TracerProvider(
    resource=Resource.create({SERVICE_NAME: "backend-service"})
))
exporter = JaegerExporter(
    collector_endpoint="http://localhost:14268/api/traces"
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(exporter)
)

app = flask.Flask(__name__)
FlaskInstrumentor().instrument_app(app)

tracer = trace.get_tracer(__name__)

@app.route("/hello")
def hello():
    name = request.args.get('name', default = "", type = str)
    with tracer.start_as_current_span("backend-hello") as span:
        span.add_event("Doing backend here.", {'name': name})

        if name == 'joe':
            span.set_status(Status(StatusCode.ERROR))
            span.add_event("I don't like joe", {'anger': 'high'})

    return "hello " + name

app.run(port=5000)
```

Running this requires more deps, so don't forget to install them before testing out our backend:

```sh
$ pip install flask requests opentelemetry-instrumentation-flask
$ python backend.py
 * Serving Flask app 'backend' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
127.0.0.1 - - [09/Nov/2021 14:13:24] "GET /hello HTTP/1.1" 200 -
```

To test the backend:

```sh
$ curl "http://127.0.0.1:5000/hello?name=world"
hello world
```

Now the frontend counterpart. Our frontend service will act as a proxy and run some requests on the backend. It will make use of the `requests` module to perform this, and as Opentelemetry is nice for already providing a specific instrumentalization for this module, we'll make use of it as well.

The RequestInstrumentalization will populate the http requests with custom headers to propagate the trace. As backend server is already instrumentalized using opentelemetry, the modules will find out that trace contexts exist, and will make use of it. 

The code will then be:

```python
import flask
from flask import request
import requests

from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
    ConsoleSpanExporter,
)
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

trace.set_tracer_provider(TracerProvider(
    resource=Resource.create({SERVICE_NAME: "web-frontend"})
))
exporter = JaegerExporter(
    collector_endpoint="http://localhost:14268/api/traces"
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(exporter)
)

app = flask.Flask(__name__)
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

tracer = trace.get_tracer(__name__)

@app.route("/")
def hello():
    name = request.args.get('name', default = "", type = str)
    with tracer.start_as_current_span("frontend-hello") as span:
        span.add_event("Calling backend.", attributes={'name': name})
        result = requests.get("http://localhost:5000/hello?name=" + name)
        return str(result.text)
    return "default response" # This should not happen.

app.run(port=8000)
```

Finally, install the pretty much last required package and run the frontend:

```sh
# pip install opentelemetry-instrumentation-requests
# python frontend.py
 * Serving Flask app 'frontend' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:8000/ (Press CTRL+C to quit)
```

This done, run a single query and watch the magic in jaeger UI:

```sh
# curl "http://127.0.0.1:8000/?name=patrick"
hello patrick
```

![Trace involving 2 components](images/jaeger_multiple.png)

We can clearly see the request going through the frontend (in brown), then the backend (in salmon), with the argument given. With this, it is possible to debug more easily complex multi-component projects.


# Adding more information in the context

As traces are now propagating among multiple components, one last bit to add is to send more informations in the context. Tracing & opentelemetry propose a "baggage" API to this, allowing to annotate a bit traces. Baggages can also be sent through API requests from one component to another.

On the frontend, you need to create a new context involving the databag, and to inject the context in the API request.

Frontend code:

```python
# Used to retrieve hostname
import socket
from opentelemetry.context import get_current
from opentelemetry.propagate import inject

...

# In the spans:

        with tracer.start_as_current_span('frontend-with-baggage') as span:
            # Retrieve current otel context to build the context to inject
            current_context = get_current()
            current_context = baggage.set_baggage('frontend_hostname', socket.gethostname(), current_context)
            headers = {}
            # Inject the context into headers
            inject(headers, context=current_context)
            
            # Send the context through the API query:
            span.add_event("Calling backend.", attributes={'name': name})
            result = requests.get("http://localhost:5000/hello?name=" + name, headers=headers)
            return str(result.text)
```

On backend, it is required to extract the given context to extract the bag:

Backend code:

```python

from opentelemetry import baggage
from opentelemetry.propagate import extract
import opentelemetry.instrumentation.wsgi as otel_wsgi

...

    extracted_context = extract(request.environ, getter=otel_wsgi.wsgi_getter)
    bag_value = baggage.get_baggage('frontend_hostname', extracted_context)
    
    with tracer.start_as_current_span("backend-hello") as span:
        span.add_event("Doing backend here.", {
            'name': name,
            'frontend_hostname': bag_value, 
        })  


```

Don't forget to install `opentelemetry-instrumentation-wsgi` before running, and to restart the services.

As a result, in our example, the retrieved hostname on the frontend is corrently sent to backend & can be added in traces as well.

![Trace with propagated baggage](images/jaeger_baggage.png)


In this post, we discovered how to add logs in our app and make sure they can be correlated to traces, how to add events in spans, trace propagation across multiple services & finally, how to send data/context between different components in a single trace.

Final source code: [frontend](src/jaeger/frontend.py), [backend](src/jaeger/backend.py).

References:
- https://medium.com/dzerolabs/observability-journey-understanding-logs-events-traces-and-spans-836524d63172
- https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/logging/logging.html
- https://opentelemetry-python.readthedocs.io/en/latest/api/baggage.html
- https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/logs/overview.md
