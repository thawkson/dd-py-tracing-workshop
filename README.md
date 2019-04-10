# Datadog Distributed Tracing Workshop

Content for a workshop on Distributed Tracing sponsored by [Datadog](http://www.datadoghq.com)

## Prerequisites

* Install ``docker`` and ``docker-compose`` on your system. Please, follow the instructions available
  in the [Docker website](https://www.docker.com/community-edition)
  For Linux environment, it should go like 
  1. ``sudo curl -sSL https://get.docker.com/ | sh``
  2. ``sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose``
* Get the workshop material with ``git clone https://github.com/DataDog/dd-py-tracing-workshop.git``
* Create a [Datadog Account](https://app.datadoghq.com/signup) and get an ``API_KEY`` for that account (you can create from the [Datadog API page](https://app.datadoghq.com/account/settings#api)). Remember to not share this key with anyone.

## Flask Application

Here's an app that does a simple thing. It tells you what donut to pair with your craft brew. While it is contrived in its purpose,
it probably has something in common with the apps you work on:
* It's a web application that exposes HTTP endpoints.
* To do its job, it must talk to datastores and external services.
* It may need performance improvements.

## Get Started

The application runs in many Docker containers that you can launch using the following command:

```bash
$ DD_API_KEY=<add_your_API_KEY_here> docker-compose up
```

Each Python application runs a Flask server with live-reload so you can update your code without restarting any container.
After executing the command above, you should have running:
* A Flask app ``web``, accepting HTTP requests
* A smaller Flask app ``taster``, also accepting HTTP requests
* Redis, the backing datastore
* Datadog agent, a process that listens for, samples and aggregates traces

You can run the following command to verify these are running properly.

```bash
$ docker-compose ps
```

If all containers are running properly, you should see the following:

```
            Name                           Command               State                          Ports
-----------------------------------------------------------------------------------------------------------------------------
ddpytracingworkshop_agent_1     /entrypoint.sh supervisord ...   Up      7777/tcp, 8125/udp, 0.0.0.0:8126->8126/tcp, 9001/tcp
ddpytracingworkshop_redis_1     docker-entrypoint.sh redis ...   Up      6379/tcp
ddpytracingworkshop_taster_1    python taster.py                 Up      0.0.0.0:5001->5001/tcp
ddpytracingworkshop_web_1       python app.py                    Up      0.0.0.0:5000->5000/tcp
```


## Step 0

Let's poke through the app and see how it works.

### Architecture

* Vital Business Info about Beers and Donuts lives in a SQL database.

* Some information about Donuts changes rapidly, with the waves of baker opinion, so
we store this time-sensitive information in a Redis-backed datastore called DonutStats.

* The `DonutStats` class abstracts away some of the gory details and provides a simple API

### HTTP Interface

* We can list the beers we have available
`curl -XGET "localhost:5000/beers"`

* The donuts we have available
`curl -XGET "localhost:5000/donuts"`

* We can grab a beer by name
`curl -XGET "localhost:5000/beers/ipa"`

* We can grab a donut by name
`curl -XGET "localhost:5000/donuts/jelly"`

So far so good.

Things feel pretty speedy. But what happens when we try to find a donut that pairs well with our favorite beer?

* `curl -XGET "localhost:5000/pair/beer?name=ipa"`

It feels slow! Slow enough that people might complain about it. Let's try to understand why




## Step 1 - Import Tracing Solution

In this first step, we'll use basic manual instrumentation to trace one single function from our application. Let's edit the `app.py` to do that.

First, we import and configure tracing capabilities...

```python
# app.py

from ddtrace import tracer
tracer.configure(hostname='agent', port=8126)
```

... and then, the datadog tracer decorator to beer function.
```python
# app.py

@app.route('/beers')
@tracer.wrap(service='beers')
def beers():
```

Now, when you call your webapp for beers `curl -XGET "localhost:5000/beers"`, you should be able to see the trace of the `beer()` function in Datadog Trace List [Datadog Trace List](https://app.datadoghq.com/apm/traces) 

When you click (View Trace ->) on your newly tracked service, you see the "details" of the trace. For now, the details are limited to the single span of the `beers` method you just instrumented. One valuable information you find here is the duration of the span.

If you access the [Beer Service Statistics](https://app.datadoghq.com/apm/service/beers/app.beers) page, you also find statistics about all the occurences of calls to that service (try `curl -XGET "localhost:5000/beers"` ten times in a row to populate that statistics. 

This is useful, but you'll need more to observe what's happening in your application and eventually fix or optimize things.


## Step 2 - Access full trace

A good tracing client will unpack, for instance, some of the layers of indirection in ORMs, and give
you a true view of the SQL being executed. This lets us marry the the nice APIs of ORMS with visibility
into what exactly is being executed and how performant it is.

We'll use Datadog's monkey patcher, a tool for safely adding tracing to packages in the import space:

```python
# app.py
patch_all(Flask=True)

```

Don't forget to remove the `tracer.wrap()` decorator from `beers()` function, which we added in Step 1 but which is useless now.

```python
@app.route('/beers')
# @tracer.wrap(service='beers')
def beers():
```

The middleware is operating by monkey patching the flask integration to ensure it is:
- Timing requests
- Collecting request-scoped metadata
- Pinning some information to the global request context to allow causal relationships to be registered

Now, if we hit our app, we can see that Datadog has begun to display some information for us. Meaning,
you should be able to see some data in the APM portion of the Datadog application.


## Step 3 - Correlate Traces and Logs

Traces are useful material, but sometimes troubleshoot starts with a line of log. Datadog magically enables correlation of traces and logs thanks to a `trace_id`. It's a unique identifier of every single trace, that you can easily report in any log written in that trace.

Let's append our logger format to inherit metadata from trace: `trace_id` and `span_id`.

```python
# app.py

patch_all(logging=True)
import logging

FORMAT = ('%(asctime)s %(levelname)s [%(name)s] [%(filename)s:%(lineno)d] '
          '[dd.trace_id=%(dd.trace_id)s dd.span_id=%(dd.span_id)s] '
          '- %(message)s')
logging.basicConfig(format=FORMAT)
log = logging.getLogger(__name__)
log.level = logging.INFO
```

We need to configure the agent to collect logs from the docker socket - refer to [agent documentation](https://docs.datadoghq.com/logs/log_collection/docker/?tab=dockercompose)
  
```
# docker-compose.yml

  agent:
    environment:
      - DD_LOGS_ENABLED=true
      - DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true
    volumes:
      - /opt/datadog-agent/run:/opt/datadog-agent/run:rw
  web:
    labels:
      com.datadoghq.ad.logs: '[{"source": "custom_python", "service": "web"}]'
  taster:
    labels:
      com.datadoghq.ad.logs: '[{"source": "custom_python", "service": "taster"}]'

```

And finally update the [Log pipelines](https://app.datadoghq.com/logs/pipelines/) to process these custom-format python logs. Note that there is no need to do it for Agent and Redis Logs, they are automatically recongnized and processes as such.

Create a new pipeline whose custom filter is `source:custotm_python`. Within that pipeline:

* Create a Grok Parser witht the following parsing rule `custom_python_trace %{date("yyyy-MM-dd HH:mm:ss,SSS"):timestamp} %{word:levelname} \[%{word}\] \[%{word}.%{word}:%{integer}] \[dd.trace_id=%{numberStr:dd.trace_id} dd.span_id=%{numberStr:dd.span_id}\] - %{data:message}`,

* Create a Date Remapper with the `timestamp` attribute,

* Create a Status remapper with the `levelname` attribute,

* Ceatet a Trace ID with the `dd.trace_id` attribute.


With that setup, you should now be able: 

* to access logs from a trace - see the Log Panel in the Trace Panel. [See Documentation](https://docs.datadoghq.com/tracing/visualization/trace/?tab=logs)

* to access a trace from a log - see the "Go to Trace" buttom in the Log Panel. [See Documentation](https://docs.datadoghq.com/logs/explorer/?tab=logstream#log-panel)  

## Step 4 - Trace Search

Trace search deactivated by default in Datadog. You should explictely enable it, service by service. 

Adding this environment variable in the datadog agent docker configures the agent to send trace events for the flask service.

```
# docker-compose.yml
  agent:
    image: "datadog/agent:latest"
    environment:
      - DD_APM_ANALYZED_SPANS=flask|flask.do_teardown_appcontext=1
```

After this, you can now search for specific traces in the [Trace Search](https://app.datadoghq.com/apm/search), and access advanced [Analytics](https://app.datadoghq.com/apm/search/analytics) capabilities as well.


## Step 5 - Distributed (work in Progress)!

Most of the hard problems we have to solve in our systems won't involve just one application. Even in our
toy app the ``best_match`` function crosses a distinct service boundary, making an HTTP call to the "taster" service.

For traditional metrics and logging crossing a service boundary is often a full stop for whatever telemetry you have
in place. But traces can cross these boundaries, which is what makes them so useful.

Here's how to make this happen in the Datadog client. First we configure the service that behaves as the client,
to propagate information about the active trace via HTTP headers:

```python
# app.py

def best_match(beer):
    # ...
    for candidate in candidates:
        try:
            # propagate the trace context between the two services
            span = tracer.current_span()
            headers = {
                "X-Datadog-Trace-Id": str(span.trace_id),
                "X-Datadog-Parent-Id": str(span.span_id),
            }

            resp = requests.get(
                "http://taster:5001/taste",
                params={"beer": beer.name, "donut": candidate},
                timeout=2,
                headers=headers,
            )
        except requests.exceptions.Timeout:
            continue
    # ...
```

We set up tracing on the server-side app (``taster``):

```python
# taster.py

import random

# import tracing functions
from ddtrace import tracer
from ddtrace.contrib.flask import TraceMiddleware

from flask import Flask, request, jsonify


app = create_app()

# trace the Flask application
tracer.configure(hostname='agent')
TraceMiddleware(app, tracer, service="taster")
```

Then we configure the server side of this equation to extract this information from the HTTP headers and continue the trace:
```python
# taster.py

@app.route("/taste")
def taste():
    # continue the trace
    trace_id = request.headers.get("X-Datadog-Trace-Id")
    parent_id = request.headers.get("X-Datadog-Parent-Id")
    if trace_id and parent_id:
        span = tracer.current_span()
        span.trace_id = int(trace_id)
        span.parent_id = int(parent_id)

    request.args.get("beer")
    # ...
```

Let's hit our pairing route a few more times now, and see what Datadog turns up:
``curl -XGET "localhost:5000/pair/beer?name=ipa"``

If everything went well we should see a distributed trace: 

![alt text](https://d1ax1i5f2y3x71.cloudfront.net/items/2C130u1S143T1f3p2H33/Image%202017-09-23%20at%202.20.42%20PM.png?X-CloudApp-Visitor-Id=2639901 "Distributed Trace")

## Step 6  - Optimize Endpoint

As we can see from that trace, we're spending a lot of time in the requests library,
especially relative to the amount of work being done in the taster application. This
looks like a place we can optimize.

If we look at the work being done in the ``best_match()`` and ``taste()``, we see
that we can probably move work from ``best_match()`` to ``taste()`` and eliminate
the number of requests we are making.

Let's do this and see if we can cut down the average latency of this code!

First, we'll refactor ``best_match`` to include all of the candidates in its request,
rather than making multiple requests.

```python
# app.py

def best_match(beer):
    # ...
    try:
        # propagate the trace context between the two services
        span = tracer.current_span()
        headers = {
            "X-Datadog-Trace-Id": str(span.trace_id),
            "X-Datadog-Parent-Id": str(span.span_id),
        }

        resp = requests.get(
            "http://taster:5001/taste",
            params={"beer": beer.name, "donuts": candidates},
            timeout=2,
            headers=headers,
        )
    except requests.exceptions.Timeout:
        # log the error
        return "not available"
    # ...
```

Then, we'll refactor the ``taste()`` function to accept this and return the donut
with the highest taste score.

```python
# taster.py

@app.route("/taste")
def taste():
    # ...
    # send the remaining candidates to our taster and pick the best
    matches = []
    beer = request.args.get("beer")
    candidates = request.args.getlist("donuts")

    for candidate in candidates:
        score = random.randint(1, 10)
        matches.append((score, candidate))

    best_match = max(matches)
    return jsonify(candidate=best_match[1], score=best_match[0])
```

Once we've done this, we can take a look at another trace and notice that we've
cut down the total time significantly, or about ~60 ms to ~20 ms: 

![https://cl.ly/3J3Y0B330p1P](https://d1ax1i5f2y3x71.cloudfront.net/items/1k02400O1X331A123036/Image%202017-09-23%20at%202.47.53%20PM.png?X-CloudApp-Visitor-Id=2639901)

## In conclusion

Datadog's tracing client gives you a lot of tools to extract meaningful insights from your Python apps.

We support an explicit approach, with tracing constructs embedded in your code, as well as more implicit ones,
with tracing auto-patched into common libraries or even triggered via our command line entrypoint like so:

```bash
$ ddtrace-run python my_app.py
```

We hope you can find an approach that fits for your app. More details at:
* https://pypi.datadoghq.com/trace/docs
* https://github.com/DataDog/dd-trace-py

Happy Tracing!
