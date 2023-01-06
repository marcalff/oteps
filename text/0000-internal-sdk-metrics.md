# Internal SDK Metrics

Provide metrics on the internal SDK and exporter execution.

## Motivation

Assume a system instrumented with OpenTelemetry.

Whenever a failure happens in the OtenTelemetry implementation itself,
the failure is not returned to the application.

This is the desired behavior, to avoid creating instability in the production
workload instrumented.

The problem is that internal failures tend to be silent, and go unnoticed,
which typically hides:

- configuration errors (for example, no endpoint defined)
- runtime errors (for example, endpoint is not responding)

From an external point of view:

- the instrumented application is running,
- the OpenTelemetry implementation appear to be working,
- some events may be delivered downstream to an OpenTelemetry collector
  process, so the full pipeline appear to be working.

However, there is no way to assess:

- if all expected events are delivered,
- if not, the actual reason why some events are not delivered.

This feature provides observability on the OpenTelemetry implementation code itself,
by providing execution metrics on every part involved in the delivery
pipeline, to account for the fate of an incoming event.

Once an event is created from the application instrumented code (say, a trace span
is created), provide metrics for:

- the number of events instrumented (incoming flow)
- the number of events discarded (noop tracer)
- the number of events sampled out
- the number of events discarded due to limits (number of attributes, links,
  ...)
- the number of events dropped due to lack of space (buffer full)
- the number of events dropped due to a time out
- the number of events delivered by an exporter (outgoing flow)

In addition, provide metrics to assess resource usage inside the
OpenTelemetry implementation:

- current usage (gauge) for an internal buffer
- max usage (high water mark) for an internal buffer
- number of retry for operations that support restarts

The metrics are available locally, using dedicated APIs, to the instrumented
application, which may want to show details in its own logs,
and/or raise alerts on the most severe errors (no endpoint to talk to).

The metrics are available remotely, exported like regular metrics,
to a downstream system.

Implementing this change is valuable:

- to troubleshoot an application instrumented with OpenTelemetry,
  to assess if events are correctly processed.

- to tune the configuration parameters affecting OpenTelemetry (buffer
  sizes, timeout values, ...) according to the application workload,
  for better resource usage and performances.

In short, this feature is observability on the observability code.

We all know how important observability is, right ?

## Explanation

For each component of the system:

- traces
  - tracer provider
  - tracer
  - sampler
  - processor
  - exporter
- metrics
  - metrics provider
  - xxx
- logs
  - logs provider
  - xxx

the component has an associated statistics object,
which collects internal execution metrics of the component.

The instrumented application can obtain the associated statistics of any
component it uses, and inspect statistics details.

For example, when using an OTLP GRPC exporter for the trace signal,
the application can inspect the associated OTLP GRPC exporter statistics,
and find out how many times the exporter failed to connect downstream,
how many span records were exported, etc.

Every statistics buffer accessible can also be used as a source of data for
async instruments, generating "internal" metrics.

For example, the OTLP GRPC trace exporter statistics can be exported
as metrics, using async instruments, to send these statistics downstream,
using the OTLP HTTP metrics exporter.

The downstream system can be the same system that collects the application
workload events, or it can be different.

In the downstream system (say, in prometheus), an admin can inspect remotely
how the OTLP GRPC trace exporter is performing, when collecting trace data for an
instrumented application.

This investigation is not limited to only the trace exporter,
any component involved in the pipeline can be inspected:

- the tracer provider
- the tracer
- the trace sampler
- the trace processor
- the trace exporter

Visualizing metrics in prometheus, an admin can determine that trace events
for a given application are not delivered due to:

- a bottleneck in the batch processor,
  which drops spans due to an insufficient buffer size for the workload.
- a few connection errors in the exporter, dropping more data

Likewise for metrics and logs.

## Internal details

TODO

From a technical perspective, how do you propose accomplishing the proposal? In particular, please explain:

* How the change would impact and interact with existing functionality
* Likely error modes (and how to handle them)
* Corner cases (and how to handle them)

While you do not need to prescribe a particular implementation - indeed, OTEPs should be about **behaviour**, not implementation! - it may be useful to provide at least one suggestion as to how the proposal *could* be implemented. This helps reassure reviewers that implementation is at least possible, and often helps them inspire them to think more deeply about trade-offs, alternatives, etc.

## Trade-offs and mitigations

TODO

What are some (known!) drawbacks? What are some ways that they might be mitigated?

Note that mitigations do not need to be complete *solutions*, and that they do not need to be accomplished directly through your proposal. A suggested mitigation may even warrant its own OTEP!

## Prior art and alternatives

TODO

What are some prior and/or alternative approaches? For instance, is there a corresponding feature in OpenTracing or OpenCensus? What are some ideas that you have rejected?

## Open questions

TODO

What are some questions that you know aren't resolved yet by the OTEP? These may be questions that could be answered through further discussion, implementation experiments, or anything else that the future may bring.

## Future possibilities

TODO

What are some future changes that this proposal would enable?
