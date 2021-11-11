---
title: "Trying Out Opentelemetry in Dotnet"
date: 2021-11-10T22:01:08+01:00
categories:
  - opentelemetry
summary: "Quick introduction of using Opentelemetry in C#/dotnet."
---

After testing Opentelemetry & Jaeger with Python, next step is now to try to set up a dotnet web app and to intrumentalize it as well. Note I don't know at all C#/dotnet environments, that is my pretty much first project with those technologies.

Also, I'm not using Visual Studio, so I won't be much helped by the environment.

# Setting up a project

After a quick look at the [official documentation](https://docs.microsoft.com/en-us/aspnet/core/tutorials/first-web-api), I'm setting up a new quick webapp.

```sh
$ dotnet new webapi -o fo
The template "ASP.NET Core Web API" was created successfully.
```

Then, after removing the default controller & everything useless for our test, get a brand new Controller of our choice:

```csharp
using Microsoft.AspNetCore.Mvc;

namespace fo.Controllers
{
    [ApiController]
    [Route("/")]
    public class FrontEndController : ControllerBase
    {
        [HttpGet("hello")]
        public string Get(string name = "world")
        {
            return "hello " + name;
        }
    }
}
```

Checking with some basic test:

```sh
$ dotnet run &
$ curl -k "https://127.0.0.1:5001/hello?name=patrick"
hello patrick
```

We're currently going through the controller as expected. Now would be a good time to start installing & enabling Opentelemetry


# Adding OpenTelemetry

With dotnet, applications can be instrumented using `System.Diagnostics.Activity` that is built into the standard .NET library, and those objects will be used by OpenTelemetry for distributed tracing & to export them to console or to Jaeger, among others.

Activity-ies here are more or less `spans` counters we worked with in Python.

But before writing our first Activity, it is required to install & enable the first required dependancies. First enables the opentelemetry tracing, then both AspNet Core intrumentation & Console exports are installed. Here all libraries are installed with the `--prerelease` flag as there is still no official/major release of the libraries.

```sh
$ dotnet add package --prerelease OpenTelemetry.Extensions.Hosting
$ dotnet add package --prerelease OpenTelemetry.Instrumentation.AspNetCore
$ dotnet add package --prerelease OpenTelemetry.Exporter.Console
```

Then, just add the OpenTelemetryTracing to the services in `Startup.cs`:

```csharp
using OpenTelemetry.Trace;
using OpenTelemetry.Resources;

        public void ConfigureServices(IServiceCollection services)
        {
            ...
            services.AddOpenTelemetryTracing(
                (builder) => builder
                    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("fo"))
                    .AddAspNetCoreInstrumentation()
                    .AddConsoleExporter()
            );
        }
```

And that's it to enable basic instrumentation of our basic AspNetCore API. After recompilation/restart and running a new http query, the first trace will show up on the console:

```sh
$ dotnet run
...
Activity.Id:          00-efdb432c3ace90f5c9d0b814a07637b1-4b985013537fdbe8-01
Activity.ActivitySourceName: OpenTelemetry.Instrumentation.AspNetCore
Activity.DisplayName: hello
Activity.Kind:        Server
Activity.StartTime:   2021-11-10T21:48:46.5182254Z
Activity.Duration:    00:00:00.0903551
Activity.TagObjects:
    http.host: 127.0.0.1:5001
    http.method: GET
    http.target: /hello
    http.url: https://127.0.0.1:5001/hello?name=patrick
    http.user_agent: curl/7.79.1
    http.route: hello
    http.status_code: 200
    otel.status_code: UNSET
Resource associated with Activity:
    service.name: fo
    service.instance.id: 3c5719bb-de80-4373-9336-0a1db913fc6f
```


## Enabling Jaeger exporter

As now tracing is working, we'll enabling Jaeger exporter & then send traces to our already running Jaeger instance. As for Console exporter, a new dependancy is required:

```sh
$ dotnet add package --prerelease OpenTelemetry.Exporter.Jaeger
```

And it is also required to enable it & configure it in `Startup.cs`:

```csharp
using OpenTelemetry;

            services.AddOpenTelemetryTracing(
                (builder) => builder
...
                    .AddJaegerExporter(opts => {
                        opts.ExportProcessorType = ExportProcessorType.Simple;
                        opts.AgentHost = "localhost";
                        opts.AgentPort = 6831;
                    })
            );
```

After a new HTTP request to the API endpoint, the trace appeared on Jaeger:

![First trace in Jaeger](images/jaeger_dotnet_basic.png)


# Creating more Activity-ies.

The trace generated is the result of the default AspNetCore instrumentation, but as applications are getting more complex, we might want to generate our own spans, and as OpenTelemetry is using the internal `System.Diagnostics.Activity` objects provided by dotnet, it is possible to define ours & intrumentalize in a finer way our application.

Before setting up our new Activity in the Controller, it is required to inform the list of the source(s) Activity will be declared in. This requires a new change to define those sources in the `Startup.cs` in the OpenTelemetry service. The Activity source name used here will be `Tracing`:

```csharp
                    .AddAspNetCoreInstrumentation()
                    .AddSource("Tracing")
                    .AddConsoleExporter()
```

This done, next is to add some new instrumentation code in our Controller, by defining an `ActivitySource` & using some Activity created thanks to this source around work units. After those changes, our controller looks like the following:

```csharp
using System.Diagnostics;

...
        private static readonly ActivitySource Activity = new("Tracing", "1.0.0");

        [HttpGet("hello")]
        public string Get(string name = "world")
        {
            using (var activity_foo = Activity.StartActivity("foo", ActivityKind.Producer)) {
                using (var activity_bar = Activity.StartActivity("bar", ActivityKind.Producer)) {
                    if (name == "foo") {
                        name = "world";
                    }
                };
            };

            using (var activity_fizz = Activity.StartActivity("fizz", ActivityKind.Producer)) {
                using (var activity_buzz = Activity.StartActivity("buzz", ActivityKind.Producer)) {
                    if (name == "fizz") {
                        name = "buzz";
                    }
                };
            };
            return "hello " + name;
        }
```

Running a new HTTP query against this new compiled code will result in jaeger by:

![Jaeger trace with multiple spans](images/jaeger_dotnet_activities.png)


# Populating Activities

Like OpenTelemtry's spans in Python, dotnet Activity-ies objects are pretty much customizable. It is possible to set a status (using the OpenTelemetry tags), 

## Adding tags

Adding tags to the activity is simple as using the `SetTag` function in the activity.

```csharp
    activity_foo?.SetTag("foo", "bar");
    activity_foo?.SetTag("hello", "world");
```

## Setting an error status

Adding a status to an Activity is simple as adding the `otel.status_code` & `otel.status_description` tags. `status_code` can be "UNSET", "OK" or "ERROR". Description is optional and can be used to describe the status code reason.

As an example:

```csharp
                if (name == "world") {
                    activity_foo?.SetTag("otel.status_code", "ERROR");
                    activity_foo?.SetTag("otel.status_description", "Name is world. Username can't be world.");
                }
```

In Jaeger, the Activity will be showing an error icon in the UI as expected, with the error description in tags:

![Jaeger trace with an error](images/jaeger_dotnet_error.png)


## Adding events

Finally, it is possible to attach events to activities. Each event is described by a mandatory message, an optional timestamp and a set of tags. The `AddEvent` Activity function allow to attach an event.

```csharp
            using (var activity_fizz = Activity.StartActivity("fizz", ActivityKind.Producer)) {
                using (var activity_buzz = Activity.StartActivity("buzz", ActivityKind.Producer)) {
                    activity_buzz?.SetTag("tag1", "this is a tag attached to the Activity");

                    if (name == "fizz") {
                        var activity_tags = new ActivityTagsCollection();
                        activity_tags.Add("tag1", "this is a tag attached to the ActivityEvent.");

                        activity_buzz?.AddEvent(new ActivityEvent(
                            "Name was changed!",
                            default,
                            activity_tags
                        ));

                        name = "buzz";
                    }
                };
            };
```

As a result, tags & events will be then showing in Jaeger like the following screenshot. Activity' tags & events' tags are clearly viewable:

![Jaeger trace with tags & events](images/jaeger_dotnet_tags_events.png)



# Instrumentalizing HTTP client

Now that our component is fully instrumentalized, we now want distributed tracing among several components. Easier first way is to go with using basic HTTP client, and Opentelemetry provides an instrumentalization for it.

To illustrate HTTP client, I'll create a brand new AspNetCore project called `bo` with a simple controller, pretty much similar than `fo`. I'll no redo the same steps than above as they are pretty much similar. Backend main controller looks like the following:

```csharp
    public class BackEndController : ControllerBase
    {
        private static readonly ActivitySource Activity = new("Tracing", "1.0.0");

        [HttpGet("reverse")]
        public string Get(string name = "world")
        {
            using (var activity_foo = Activity.StartActivity("foo", ActivityKind.Producer)) {
                activity_foo?.SetTag("input", name);

                char[] nameArray = name.ToCharArray();
                Array.Reverse(nameArray);

                var output = new string(nameArray);

                var event_tags = new ActivityTagsCollection();
                event_tags.Add("request.input", name);
                event_tags.Add("request.output", output);
                
                activity_foo?.AddEvent(
                    new ActivityEvent("Got http response for the reverse on backend request.", default, event_tags)
                );

                activity_foo?.SetTag("output", name);

                return output;
            };
        }
    }
```

After modifying `launchSettings.json` to bind ports 6000 (http only, too lazy to set up ssl ceritficates on my setup) for this new webservice, I start the service & check if this new backend service is able to send traces to Jaeger.

```json
...
      "applicationUrl": "http://localhost:6000",
...
```

This done, let's modify our frontend code to execute an http request to backend.

First, it is required to add the new Http client intrumentalization dependancy.

```sh
$ dotnet add package --prerelease OpenTelemetry.Instrumentation.Http
```

Then, enable it in Startup.cs:

```csharp
                    .AddHttpClientInstrumentation()
```

Finally, do some code to call the backend webservice:

```csharp
using System.IO;
using System.Net;

...
            using (var activity_foo = Activity.StartActivity("foo", ActivityKind.Producer)) {
                activity_foo?.AddEvent(new ActivityEvent("Starting request to backend."));

                HttpWebRequest request = (HttpWebRequest)WebRequest.Create("http://localhost:6000/reverse?name=patrick");
                HttpWebResponse response = (HttpWebResponse)request.GetResponse();

                var response_body = new StreamReader(response.GetResponseStream()).ReadToEnd();

                var event_tags = new ActivityTagsCollection();
                event_tags.Add("response.length", response_body.Length);
                event_tags.Add("response.body", response_body.ToString());
                
                activity_foo?.AddEvent(
                    new ActivityEvent("Got http response for the request.", default, event_tags)
                );

                return "hello " + response_body;
            };

...
```

All this done, start both `bo` & `fo` component, and run a request against frontend API:

```sh
$ mycroft@saisei ~> curl -k "https://127.0.0.1:5001/hello?name=fizz"
hello kcirtap
```

The trace will then show up in Jaeger as the following:

![Jaeger trace with http query and subcomponent](images/jaeger_dotnet_http.png)


# Using Baggage to propagage context information

Baggages are key/values and can be used to propagage context informations among processes involved into a single trace.

Adding a k/v into the current's activity baggage is simple as that:

On frontend:

```csharp
            var remoteIpAddress = HttpContext.Connection.RemoteIpAddress;
            var remoteIpPort = HttpContext.Connection.RemotePort;

            using (var activity_foo = Activity.StartActivity("foo", ActivityKind.Producer)) {
                activity_foo?.SetTag("client_port", remoteIpAddress + ":" + remoteIpPort);
                activity_foo?.AddEvent(new ActivityEvent("Starting request to backend."));

                activity_foo.AddBaggage("client_port", remoteIpAddress + ":" + remoteIpPort);
...
            };

```

... and the bagages values can be extracted out an `IEnumerable` as the following on the backend part of the application stack:

```csharp
using System.Linq;

...

        public string Get(string name = "world")
        {
            // Sending all baggages into this activity tags
            foreach (var (key, value) in Activity.Current?.Baggage)
            {
                Activity.Current?.SetTag(key, value);
            }

            using (var activity_foo = Source.StartActivity("foo", ActivityKind.Producer)) {
                activity_foo?.SetTag("input", name);

                // Extracting a single baggage item & inserting it as a tag.
                var baggage = Activity.Current?.Baggage.ToDictionary(item => item.Key);

                if (baggage.ContainsKey("client_port")) {
                    activity_foo?.SetTag("client_port", baggage["client_port"].Value);
                }        
...
        }
```

The next possible thing to do is to send the context & propagage the trace using a new way to communicate among processes, such as a message queue.


# Propagating traces by exporting & importing context to workers using message queues 

I'll be using rabbitmq to send messages with all the require information to continue the trace. The information will be exported from dotnet Activities to Rabbitmq properties in a message, then re-imported in the worker process in order to complete the whole task.


First before all, enable rabbitmq using an official docker image:

```sh
$ podman run --name rabbitmq -d -p 5672:5672 -p 15672:15672 rabbitmq
```

On the backend process, let's make sure the context information is correctly sent through RabbitMQ. The new `RabbitMQ.Client` needs to be installed.

```sh
$ dotnet add package RabbitMQ.Client
```

Then, the backend code needs to be enhanced to send a message on each request to RabbitMQ.

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Text;

using RabbitMQ.Client;

using OpenTelemetry;
using OpenTelemetry.Context.Propagation;

...

        private void InjectContextIntoHeader(IBasicProperties props, string key, string value)
        {
            try
            {
                props.Headers ??= new Dictionary<string, object>();
                props.Headers[key] = value;
            }
            catch (Exception)
            {
                Console.WriteLine("Failed to inject trace context.");
            }
        }

...

            using (var activity = Source.StartActivity("Sending message to Rabbitmq", ActivityKind.Producer))
            {
                var factory = new ConnectionFactory { HostName = "localhost" };
                using (var connection = factory.CreateConnection())
                using (var channel = connection.CreateModel())
                {
                    activity?.SetTag("messaging.system", "rabbitmq");
                    activity?.SetTag("messaging.destination_kind", "queue");
                    activity?.SetTag("messaging.rabbitmq.queue", "sample");

                    var props = channel.CreateBasicProperties();
                    Propagator.Inject(
                        new PropagationContext(
                            activity.Context,
                            Baggage.Current), 
                        props,
                        InjectContextIntoHeader);

                    channel.QueueDeclare(queue: "sample",
                        durable: false,
                        exclusive: false,
                        autoDelete: false,
                        arguments: null);

                    channel.BasicPublish(exchange: "",
                        routingKey: "sample",
                        basicProperties: props,
                        body: Encoding.UTF8.GetBytes(output));
                }
            };

```

Then, we need a worker to receive the messages, and we then want to continue & complete the execution trace with this component. This last component will be named ... `worker`:

```sh
$ dotnet new console -o worker
$ cd worker
$ dotnet add package RabbitMQ.Client
$ dotnet add package --prerelease OpenTelemetry.Extensions.Hosting
$ dotnet add package --prerelease OpenTelemetry.Exporter.Jaeger
```

The worker code is a bit long, as we need to setup OpenTelemetry, RabbitMq, un-queue messages, extract metadata, then emit a trace. Source code in a brief looks like:

```csharp
        private static void ProcessMessage(BasicDeliverEventArgs ea, IModel rabbitMqChannel)
        {
            var parentContext = Propagator.Extract(
                default,
                ea.BasicProperties,
                ExtractTraceContextFromBasicProperties);

            Baggage.Current = parentContext.Baggage;

            using (var activity = Source.StartActivity("Worker: Processing Message", ActivityKind.Consumer, parentContext.ActivityContext))
            {
                try
                {
                    var body = ea.Body.ToArray();
                    var message = Encoding.UTF8.GetString(body);

                    foreach(var (key, value) in Baggage.Current) {
                        activity?.SetBaggage(key, value);
                    }

                    activity?.SetTag("messaging.system", "rabbitmq");
                    activity?.SetTag("messaging.destination_kind", "queue");
                    activity?.SetTag("messaging.rabbitmq.queue", "sample");

                    var baggage = activity?.Baggage.ToDictionary(item => item.Key);
                    var client = "unknown";

                    if (baggage.ContainsKey("client_port")) {
                        activity?.SetTag("client_port", baggage["client_port"].Value);
                        client = baggage["client_port"].Value;
                    } 

                    activity?.AddEvent(new ActivityEvent($"Recieved a message \"{message}\" from client:{client}"));

                    rabbitMqChannel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
                }
                catch(Exception ex)
                {
                    Console.Write($"Got an error while processing message: {ex}");
                }
            }
        }
```

As a result, after an execution, we have the whole process: It goes from frontend, then to backend, then to worker process run async. We've sent required baggage when needed as well, and the information is still in the trace at the end of the process execution.

![Jaeger trace using async component](images/jaeger_dotnet_with_queue.png)


In this tutorial, we enabled tracing in a dotnet application, using already implemented instrumentalization but also handing it manually when going through rabbitmq. We've seen how to custom Activity as spans, set a status code, events, baggages...



As I'm not a dotnet developer, it took me a while to write all this. Nothing would have been possible without those references:

- https://docs.microsoft.com/en-us/aspnet/core/tutorials/first-web-api
- https://docs.microsoft.com/fr-fr/dotnet/core/diagnostics/distributed-tracing-instrumentation-walkthroughs

- https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry.Api/README.md
- https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry.Instrumentation.Http/README.md

- https://opentelemetry.lightstep.com/csharp/
- https://www.mytechramblings.com/posts/getting-started-with-opentelemetry-and-dotnet-core/