# Kyverno Design Proposal - Opentelemetry Support in Kyverno

Author: [Tathagata Paul](https://github.com/4molybdenum2)

PR: [#3514](https://github.com/kyverno/kyverno/pull/3514)

## Contents

- [Kyverno Design Proposal - Opentelemetry support in Kyverno](#kyverno-design-proposal---opentelemetry-support-in-kyverno)
  * [Project Goals](#project-goals)
  * [Benefits](#benefits)
  * [Tradeoffs](#tradeoffs)
  * [Implementation](#implementation)
    + [Metrics](#metrics)
    + [Traces](#traces)
    + [Logs](#logs) 

## Project Goals

- Replace the current Prometheus implementation for metrics with OpenTelemetry instrumentation for Kyverno.
- Add support for tracing which calls some external API's in Kyverno
- Add support for exporting logs to a logging backend from Kyverno

## Benefits:

- A single, vendor-agnostic instrumentation library for metrics, logs and traces.
- This essentially means that we can instrument Kyverno to support not only Prometheus but export metrics to other monitoring tools over the course of time.
- Decoupling the instrumentation and implementing an OpenTelemetry Collector will allow us to export metrics in different formats just by changing the configuration of the collector.
- High-quality metrics without comprosing application performance. This can reduce memory issues for Kyverno in large clusters. 
- We can add tracing support for Kyverno, to trace and debug errors in functions that make calls to external APIs.
- TLS while exporting telemetry data to a backend.

## TradeOffs:

- The current Metrics API is an unstable Alpha stage, therefore with the change in the implementation of OpenTelemetry codebase, it will become necessary for us to maintain it over time.
- Currently there is minimal support for logging instrumentation in Opentelemetry.
- Documentation is scarce for metrics and logging APIs in Opentelemetry.


## Implementation:

Current requirements are that for metrics and traces support.
### Metrics
- [x] Remove the current Prometheus instrumentation for metrics.
- [x] Replace the Prometheus implementation with the Opentelemetry instrumentation for exporting metrics.
- [x] There must be support such that the metrics can be directly fetched by Prometheus or an Opentelemetry collector, based upon user's choice of exporting the metrics (can be decided using flags).
- [x] The new implementation must still support the older configurations of exporting metrics directly to Prometheus such that the clients don't need to change anything on their end.
- [x] The GRPC configurations must support TLS.
- [x] Proper documentation update for configuring a Opentelemetry Collector and exporting metrics using the new implementation.

Example Metric Instrumentation:
```go
m.admissionRequestsMetric, err = meter.SyncInt64().Counter("kyverno_admission_requests_total", instrument.WithDescription("can be used to track the number of admission requests encountered by Kyverno in the cluster"))
if err != nil {
    m.Log.Error(err, "Failed to create instrument")
    return nil, err
}
```

The above is a counter metric which counts the number of admission requests made by a client.

### Traces

Traces for external calls from Kyverno (for example Cosign) and webhook calls are to be instrumented for generating traces. The traces are exported to a tracing backend such as Jaeger.

Example Tracing Instrumentation:

```go
tracing.DoInSpan(context.Background(), "cosign", "verify_image_signatures", func(ctx context.Context) {
		signatures, bundleVerified, err = client.VerifyImageSignatures(ctx, ref, cosignOpts)
	}
```
