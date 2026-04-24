# DEV2IL: Observability

## Distributed Tracing with Jaeger

Logs show you what happened. Metrics show you how your system performs. Traces help you understand
the flow of requests across multiple services. When a customer orders a smoothie, the request flows from
the order service to the kitchen service. Distributed tracing helps you see this entire journey.

We use OpenTelemetry to automatically create traces for all HTTP requests and also to connect 
traces between services with each other. We use Jaeger to collect and visualize these traces.

- Download the file [docker-compose.yml](https://github.com/peterrietzler/ais-dev2il-smoothie-shop/blob/tracing/docker-compose.yml)
and overwrite the existing one in the root directory of the project. Make sure you understand it!
- Start Jaeger by running `docker-compose up -d`
- Add the following code to the top of `kitchen_service.py`
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.sdk.resources import Resource

# Configure OpenTelemetry tracing
resource = Resource.create({"service.name": "kitchen-service"})
trace.set_tracer_provider(TracerProvider(resource=resource))
# This is going to export the tracing data to Jaeger
otlp_exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
trace.get_tracer_provider().add_span_processor(BatchSpanProcessor(otlp_exporter))
```
- Add the following code to `kitchen_service.py` after the creation of the `FastAPI` instance
```python
# This is going to hook into FastAPI and automatically create traces for all HTTP requests
# We exclude "receive" and "send" spans because they are not relevant for us and just add noise to the traces
FastAPIInstrumentor.instrument_app(app, exclude_spans=["receive", "send"])
# This is going to hook into HTTPX to automatically create traces for all outgoing HTTP requests and to 
# connect traces between services with each other
HTTPXClientInstrumentor().instrument()
```
- Insert the same code blocks into `order_service.py` as well. Make sure to change the `service.name` to `order-service`.

![Jaeger](jaeger.png)

The smoothie shop now uses OpenTelemetry to automatically create traces for all HTTP requests and 
connect traces between services. This means you can follow a single order from the moment it
arrives at the order service through to when the kitchen prepares it.

To view traces:
1. Make sure that both services are reloaded and generate some traffic
1. Open Jaeger UI at http://localhost:16686
1. Select `order-service` from the _Service_ dropdown and `POST /order` from the _Operation_ dropdown
1. Click _Find Traces_ to see all traces
1. Click on any trace to see the detailed view

When you open a trace, you'll see:
- **Timeline**: Visual representation of when each span started and how long it took
- **Service Dependencies**: How the order service calls the kitchen service
- **Error Traces**: If the kitchen is busy (503 error), you'll see red spans indicating failures
- **Tags**: Additional information about the span, e.g. the HTTP method or status code

Try to answer these questions using Jaeger:
- How many services are involved in a single smoothie order?
- Can you identify the slowest part of the request flow?
- When the kitchen returns a 503 error, how much time was spent before the error occurred?

The used instrumentors hook into libaries automatically and create spans. You can also create spans manually
using the OpenTelemetry API. Consider, we want to know how long we have to wait for a cook to become available
and how long it actually takes to prepare a smoothie. In `kitchen_service.py`, replace the `prepare_smoothie` function with:
```python
@app.post("/prepare")
async def prepare_smoothie(order: SmoothieOrder):
    logger.info(f"Received order to prepare a smoothie with flavor {order.flavor}")

    # Increment the counter for this flavor
    smoothies_ordered.labels(flavor=order.flavor).inc()

    # Get a tracer named after this module (same pattern as getLogger(__name__) for loggers).
    # The name identifies which part of the code created the tracer and will show up
    # as `otel.library.name` in Jaeger, so you can see which module produced a span.
    tracer = trace.get_tracer(__name__)

    # Custom span: Waiting for cook to become available
    with tracer.start_as_current_span("wait_for_cook") as wait_span:
        wait_span.set_attribute("flavor", order.flavor)
        wait_span.set_attribute("num_cooks", NUM_COOKS)
        try:
            logger.debug(f"Waiting for a cook to become available")
            await asyncio.wait_for(cook_semaphore.acquire(), timeout=2.0)
        except asyncio.TimeoutError:
            logger.error(f"Can't process the order: {NUM_COOKS} cooks are currently busy. Consider increasing NUM_COOKS.")
            raise HTTPException(status_code=503, detail="All cooks are currently busy")

    try:
        # Custom span: Preparing the smoothie
        with tracer.start_as_current_span("prepare_smoothie") as prep_span:
            prep_span.set_attribute("flavor", order.flavor)
            preparation_time = random.uniform(1.5, 2.5)
            await asyncio.sleep(preparation_time)
            logger.debug(f"Smoothie with flavor {order.flavor} prepared")

        return {"status": "done", "flavor": order.flavor}
    finally:
        cook_semaphore.release()
```

The kitchen service now includes two custom spans that show internal operations:
- **wait_for_cook**: Shows how long an order waits for an available cook
- **prepare_smoothie**: Shows the actual smoothie preparation time

These custom spans help you understand:
- **Queue time vs work time**: Is most time spent waiting or working?
- **Bottlenecks**: If `wait_for_cook` is long, you need more cooks (increase NUM_COOKS)
- **Business metrics**: Each span includes the flavor as an attribute, which would allow you to spot differences in preparation times between flavors

To view traces:
1. Make sure that both services are reloaded and generate some traffic
1. Find traces in the Jager UI and inspect the span hierarchy

### 🚀 Level Up

#### Challenge 1: Jump from a Jaeger Trace to the Matching Logs (Trace ID)

You will get the most out of all this information if you can correlate different signals together.
Correlating by timestamps is possible, but it breaks down quickly when multiple requests happen at the same time.

**Goal:** When you see a slow or failing trace in Jaeger, instantly find the **exact log lines** for that same request.

**Step 1: Put `trace_id` into every log line**

Add the following code block to both, `order_service.py` and `kitchen_service.py`:

```python
from opentelemetry.instrumentation.logging import LoggingInstrumentor

# Instrument logging to automatically inject trace context into all log records

def log_hook(span, record):
    if not hasattr(record, "tags"):
        record.tags = {}
    record.tags["service_name"] = resource.attributes["service.name"]
    record.tags["trace_id"] = format(span.get_span_context().trace_id, "032x")

LoggingInstrumentor().instrument(log_hook=log_hook)
```

This adds two labels to all your log records:
- `service_name`
- `trace_id`

**Step 2: Search for Logs of a Trace**

1. Reload both services and generate some traffic
2. Open Jaeger UI at http://localhost:16686 and open any trace
3. Copy the trace id from the URL
4. Go to Grafana and search for logs with `trace_id=<the value you copied>`

#### Challenge 2: Zero-code Tracing via CLI Auto-Instrumentation

In this challenge, you will add tracing **without changing a single line of application code**.

**Setup**
- Commit all your changes, so you don't lose any work. You can also stash your changes if you want to keep them but don't want to commit them yet.
- Keep your current `docker-compose` services running (Jaeger, Grafana, etc.)
- Checkout the `metrics` branch

**Important:** The `metrics` branch does **not** contain any tracing code.

**Step 1: Install auto-instrumentation libraries**

```bash
uv add opentelemetry-distro opentelemetry-exporter-otlp opentelemetry-instrumentation-fastapi
```
**Step 2: Restart the services using `opentelemetry-instrument`**

> 💡 Hint: remove the `--reload` flag. Auto-instrumentation + reload often does not behave well.

In one terminal:

```bash
export OTEL_SERVICE_NAME=order-service
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
uv run opentelemetry-instrument uvicorn order_service:app --port 8000 --log-config logging_config.yaml
```

In a second terminal:

```bash
export OTEL_SERVICE_NAME=kitchen-service
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
uv run opentelemetry-instrument uvicorn kitchen_service:app --port 8001 --log-config logging_config.yaml
```

**Step 3: Verify**
1. Generate traffic (order a few smoothies)
2. Open Jaeger again (http://localhost:16686)
3. You should now see traces even though the code contains no tracing

More information:
- https://opentelemetry.io/docs/zero-code/python/
