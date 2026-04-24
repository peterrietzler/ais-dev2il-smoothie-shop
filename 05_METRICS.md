# DEV2IL: Observability

## Collecting Metrics with Prometheus

Logs are great for understanding what happened in your application. Metrics help you understand
how your application is performing. We use Prometheus to collect, store and query metrics.

- Download the file [docker-compose.yml](https://github.com/peterrietzler/ais-dev2il-smoothie-shop/blob/metrics/docker-compose.yml)
and overwrite the existing one in the root directory of the project. Make sure you understand it!
- Download the file [prometheus.yml](https://github.com/peterrietzler/ais-dev2il-smoothie-shop/blob/metrics/prometheus.yml)
and store it in the root directory of the project. Make sure you understand it!
- Start Prometheus by running `docker-compose up -d`
- Add the following lines to the `kitchen_service.py`, right after the creation of the `FastAPI` instance
```python
from prometheus_fastapi_instrumentator import Instrumentator
# Initialize Prometheus metrics instrumentation
Instrumentator().instrument(app).expose(app)
```

![Prometheus](prometheus.png)

The smoothie shop now exposes HTTP metrics automatically using the `prometheus-fastapi-instrumentator`
library. This library automatically instruments all HTTP endpoints and provides metrics like:
- `http_requests_total` - Total number of HTTP requests (counter)
- `http_request_duration_seconds_sum` - Duration of HTTP requests (counter)

Make sure that both services are reloaded and generate some traffic.

You can look at the latest state of the kitchen service's metrics by visiting: http://localhost:8001/metrics. This is the page that 
Prometheus scrapes regularly to collect metrics data. Search for `http_requests_total` to find the total 
number of HTTP requests that the kitchen service has received up until the point in time you loaded the page.

To view the metrics:
1. Open Prometheus at http://localhost:9090
1. Enter the query `http_requests_total` and inspect the table results. The table shows all available time series for this metric.
1. Filter down to one dedicated time series through a label, for example: `http_requests_total{handler="/prepare"}`
1. Use `sum(http_requests_total)` to get the total number of requests across all time series
1. Try these queries and have a look at the table and graph results:
   - Request rate per second: `rate(http_requests_total[1m])`
   - Average request duration: `http_request_duration_seconds_sum / http_request_duration_seconds_count` 
   - Average request duration calls to `/prepare`: `http_request_duration_seconds_sum{handler="/prepare"} / http_request_duration_seconds_count{handler="/prepare"}`
   
The query language that you are just using is called "PromQL". It is a powerful language to query and 
aggregate metrics data. You can find more information about it in the 
[Prometheus documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/).

We are now going to add our own business level metric. Consider, we want to know how many smoothies
are ordered per flavour. We can add a custom Prometheus counter metric to provide this information. 

In `kitchen_service.py` add the following right after the creation of the `Instrumentator`:
```python
# Custom metric: Count smoothies ordered by flavor
from prometheus_client import Counter
smoothies_ordered = Counter(
    'smoothies_ordered_total',
    'Total number of smoothies ordered',
    ['flavor']
)
```
Then add the following to the start of the `prepare_smoothie` function:
```python
# Increment the counter for this flavor
smoothies_ordered.labels(flavor=order.flavor).inc()
```

Make sure that the kitchen service is reloaded, generate some traffic and head over to 
Prometheus to answer the following questions:
- Total smoothies ordered by flavor: `smoothies_ordered_total`
- Most popular flavor: `topk(1, smoothies_ordered_total)`
- Rate of smoothies ordered per flavor: `rate(smoothies_ordered_total[5m])`

### 🚀 Level Up

#### Challenge 1: Understand `increase()`

The metric `smoothies_ordered_total` is a **counter**. A counter only goes up. It tells you the
total number of smoothies ordered since the service started.

Sometimes this total number is not what you want. Often, you want to know:

> How many new smoothies were ordered in the last 1 minute?

This is exactly what `increase()` does.

Try it out in Prometheus:
- Show the total number of ordered smoothies: `smoothies_ordered_total`
- Order a few smoothies and run: `increase(smoothies_ordered_total[1m])`
- Group the result by flavor: `sum by (flavor) (increase(smoothies_ordered_total[1m]))`

If the counter for Mango went from 12 to 17 during the last 1 minute, then 
`increase(smoothies_ordered_total{flavor="Mango"}[1m])` is `5`.

Try to answer these questions:
- What is the difference between `smoothies_ordered_total` and `increase(smoothies_ordered_total[1m])`?
- Which query would you use to show all smoothies ever ordered?
- Which query would you use to show only recent smoothie orders?

#### Challenge 2: Build Your Own Grafana Dashboard

Create your own Grafana dashboard for smoothie orders. Try to work you through Grafana intuitively or
use an assistant like Geminin in order to point you to the right places in Grafana.

Before you start building the dashboard, make sure that Grafana can talk to Prometheus. You are already
able to create a new connection with Prometheus as you already did it for Loki and you have the 
`docker-compose.yml` file that contains all the relevant information. 

<details>
<summary>Show hints for connecting Grafana to Prometheus</summary>
<ol>
  <li>Open Grafana at <a href="http://localhost:3000">http://localhost:3000</a></li>
  <li>Navigate to <em>Menu &gt; Connections &gt; Add new connection</em></li>
  <li>Search for the <em>Prometheus</em> data source and add it</li>
  <li>Set the connection URL to <code>http://prometheus:9090</code></li>
  <li>Click <em>Save &amp; Test</em></li>
</ol>
</details>

Your dashboard should contain:
1. A **Stat** panel that shows how many smoothies were ordered in the currently selected time range
1. A **Time series** panel that shows smoothie orders over time
1. A dashboard **variable** called `flavor` so that users can filter the dashboard

Useful PromQL queries:
- Variable query: `label_values(smoothies_ordered_total, flavor)`
- Stat panel: `sum(increase(smoothies_ordered_total{flavor=~"$flavor"}[$__range]))`
- Time series panel: `sum(increase(smoothies_ordered_total{flavor=~"$flavor"}[1m]))`
- Optional: show one line per flavor with `sum by (flavor) (increase(smoothies_ordered_total[1m]))`

Hints:
- Enable an **All** option for the `flavor` variable
- Use `increase(...)` for recent activity instead of the raw counter
- Check whether changing the `flavor` variable updates both panels

Try to answer these questions:
- Why is a **Stat** panel a good fit for a single important number?
- Why is a **Time series** panel a better fit for changes over time?
- Why is `increase(...)` easier to understand for a user in a dashboard than the raw counter value?
