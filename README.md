# AspNet Core: Tracing, Metrics (SPM), Logs Development/Demo Environment

Service Performance Monitoring (SPM) is an opt-in feature introduced to Jaeger that provides Request, Error and Duration (RED) metrics grouped by service name and operation that are derived from span data. These metrics are programmatically available through an API exposed by jaeger-query along with a "Monitor" UI tab that visualizes these metrics as graphs. For more details on this feature, please refer to the [tracking Issue](https://github.com/jaegertracing/jaeger/issues/2954) documenting the proposal and status.

The motivation for providing this environment is to allow developers to either test Jaeger UI or their own applications against jaeger-query's metrics query API, as well as a quick and simple way for users to bring up the entire stack required to visualize RED metrics from simulated traces (or their own).

Environment Components:
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/): vendor agnostic integration layer for traces and metrics. Its main role in this particular development environment is to receive Jaeger spans, forward these spans untouched to Jaeger All-in-one while simultaneously aggregating metrics out of this span data. To learn more about span metrics aggregation, please refer to the [spanmetrics processor documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/spanmetricsprocessor).
- [Jaeger All-in-one](https://www.jaegertracing.io/docs/1.24/getting-started/#all-in-one): the full Jaeger stack in a single container image.
- [Kibana](https://www.elastic.co/kibana/): a free and open user interface that lets you visualize your Elasticsearch data and navigate the Elastic Stack
- [ElasticSearch](https://www.elastic.co/elasticsearch/): a distributed, RESTful search and analytics engine capable of addressing a growing number of use cases
- [Prometheus](https://prometheus.io/): a metrics collection and query engine, used to scrape metrics computed by OpenTelemetry Collector, and presents an API for Jaeger All-in-one to query these metrics.

Open ports:
- Kibana (Logs): 5601
- Jaeger UI (Tracing / SPM): 16686
- Prometheus (Metrics): 9090
- OTLP (GRPC Receiver): 4317

# Getting Started

## Bring up/down the dev environment

```bash
docker-compose up
docker-compose down --remove-orphans
```

## Setup AspNetCore Project

Nuget Packages (Preview):
```
OpenTelemetry.Extensions.Hosting
OpenTelemetry.Instrumentation.AspNetCore
OpenTelemetry.Instrumentation.EntityFrameworkCore
OpenTelemetry.Instrumentation.Http
OpenTelemetry.Exporter.OpenTelemetryProtocol
OpenTelemetry.Exporter.OpenTelemetryProtocol.Logs
```

appsettings.json:
```
"OpenTelemetry": {
  "ServiceName": "Microservice Name",
  "ServiceNamespace": "Microservices",
  "ServiceVersion": "1.0.0",
  "Exporter": {
    "Endpoint": "http://localhost:4317"
  }
}
```

Program.cs / Startup.cs:
```
var otlpResourceBuilder = ResourceBuilder
        .CreateDefault()
        .AddService(
            configuration.GetValue<string>("OpenTelemetry:ServiceName"),
            configuration.GetValue<string>("OpenTelemetry:ServiceNamespace"),
            configuration.GetValue<string>("OpenTelemetry:ServiceVersion"));
var otlpExporterOptions = new Action<OtlpExporterOptions>(opt =>
{
    opt.Endpoint = new Uri(configuration.GetValue<string>("OpenTelemetry:Exporter:Endpoint"));
    opt.Protocol = OtlpExportProtocol.Grpc;
});
services.AddOpenTelemetryTracing(t =>
{
    t
    .AddSource(configuration.GetValue<string>("OpenTelemetry:ServiceName"))
    .SetResourceBuilder(otlpResourceBuilder)
    .AddAspNetCoreInstrumentation(o =>
    {
        o.Filter = (httpContext) => !httpContext.Request.Path.ToString().Contains("/_");
        o.Enrich = (activity, eventName, rawObject) =>
        {
            if (eventName.Equals("OnStartActivity"))
            {
                if (rawObject is HttpRequest httpRequest)
                {
                    activity.SetStartTime(DateTime.Now);
                    activity.SetTag("http.method", httpRequest.Method);
                    activity.SetTag("http.url", httpRequest.Path);
                }
            }
            else if (eventName.Equals("OnStopActivity"))
            {
                if (rawObject is HttpResponse httpResponse)
                {
                    activity.SetEndTime(DateTime.Now);
                    activity.SetTag("responseLength", httpResponse.ContentLength);
                }
            }
        };
    })
    .AddHttpClientInstrumentation(opt => opt.RecordException = true)
    .AddEntityFrameworkCoreInstrumentation(o => o.SetDbStatementForText = true)
    .AddOtlpExporter(otlpExporterOptions);
});

services.AddOpenTelemetryMetrics(t =>
{
    t
    .SetResourceBuilder(otlpResourceBuilder)
    .AddAspNetCoreInstrumentation()
    .AddOtlpExporter(otlpExporterOptions);
});

services.AddLogging(opt =>
{
    opt.AddOpenTelemetry(t =>
    {
        t
        .SetResourceBuilder(otlpResourceBuilder)
        .AddOtlpExporter(otlpExporterOptions);
    });
});

services.AddSingleton(TracerProvider.Default.GetTracer(
    configuration.GetValue<string>("OpenTelemetry:ServiceName")));
```

**Tip:** Let the application run for a couple of minutes to ensure there is enough time series data to plot in the dashboard. Navigate to Jaeger UI at http://localhost:16686/ and inspect the Monitor tab.

# Jaeger HTTP API

## Query Metrics

`/api/metrics/{metric_type}?{query}`

Where (Backus-Naur form):
```
metric_type = 'latencies' | 'calls' | 'errors'

query = services , [ '&' optionalParams ]

optionalParams = param | param '&' optionalParams

param =  groupByOperation | quantile | endTs | lookback | step | ratePer | spanKinds

services = service | service '&' services
service = 'service=' strValue
  - The list of services to include in the metrics selection filter, which are logically 'OR'ed.
  - Mandatory.

quantile = 'quantile=' floatValue
  - The quantile to compute the latency 'P' value. Valid range (0,1].
  - Mandatory for 'latencies' type.

groupByOperation = 'groupByOperation=' boolValue 
boolValue = '1' | 't' | 'T' | 'true' | 'TRUE' | 'True' | 0 | 'f' | 'F' | 'false' | 'FALSE' | 'False'
  - A boolean value which will determine if the metrics query will also group by operation.
  - Optional with default: false

endTs = 'endTs=' intValue
  - The posix milliseconds timestamp of the end time range of the metrics query.
  - Optional with default: now

lookback = 'lookback=' intValue
  - The duration, in milliseconds, from endTs to look back on for metrics data points.
  - For example, if set to `3600000` (1 hour), the query would span from `endTs - 1 hour` to `endTs`.
  - Optional with default: 3600000 (1 hour).

step = 'step=' intValue
  - The duration, in milliseconds, between data points of the query results.
  - For example, if set to 5s, the results would produce a data point every 5 seconds from the `endTs - lookback` to `endTs`.
  - Optional with default: 5000 (5 seconds).

ratePer = 'ratePer=' intValue
  - The duration, in milliseconds, in which the per-second rate of change is calculated for a cumulative counter metric.
  - Optional with default: 600000 (10 minutes).

spanKinds = spanKind | spanKind '&' spanKinds
spanKind = 'spanKind=' spanKindType
spanKindType = 'unspecified' | 'internal' | 'server' | 'client' | 'producer' | 'consumer'
  - The list of spanKinds to include in the metrics selection filter, which are logically 'OR'ed.
  - Optional with default: 'server'
```


## Min Step

`/api/metrics/minstep`

Gets the min time resolution supported by the backing metrics store, in milliseconds, that can be used in the `step` parameter.
e.g. a min step of 1 means the backend can only return data points that are at least 1ms apart, not closer.

## Responses

The response data model is based on [`MetricsFamily`](https://github.com/jaegertracing/jaeger/blob/main/model/proto/metrics/openmetrics.proto#L53).

For example:
```
{
  "name": "service_call_rate",
  "type": "GAUGE",
  "help": "calls/sec, grouped by service",
  "metrics": [
    {
      "labels": [
        {
          "name": "service_name",
          "value": "driver"
        }
      ],
      "metricPoints": [
        {
          "gaugeValue": {
            "doubleValue": 0.005846808321083344
          },
          "timestamp": "2021-06-03T09:12:06Z"
        },
        {
          "gaugeValue": {
            "doubleValue": 0.006960443672323934
          },
          "timestamp": "2021-06-03T09:12:11Z"
        },
...
  ```

If the `groupByOperation=true` parameter is set, the response will include the operation name in the labels like so:
```
      "labels": [
        {
          "name": "operation",
          "value": "/FindNearest"
        },
        {
          "name": "service_name",
          "value": "driver"
        }
      ],
```

# Disabling Metrics Querying

As this is feature is opt-in only, disabling metrics querying simply involves omitting the `METRICS_STORAGE_TYPE` environment variable when starting-up jaeger-query or jaeger all-in-one.

For example, try removing the `METRICS_STORAGE_TYPE=prometheus` environment variable from the [docker-compose.yml](./docker-compose.yml) file.

Then querying any metrics endpoints results in an error message:
```
$ curl http://localhost:16686/api/metrics/minstep | jq .
{
  "data": null,
  "total": 0,
  "limit": 0,
  "offset": 0,
  "errors": [
    {
      "code": 405,
      "msg": "metrics querying is currently disabled"
    }
  ]
}
```
