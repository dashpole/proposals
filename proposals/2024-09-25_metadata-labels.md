## Type and Unit Metadata Labels

* **Owners:**
  * David Ashpole [@dashpole](https://github.com/dashpole)

* **Implementation Status:** `Not implemented`

* **Related Issues and PRs:**
  * https://github.com/open-telemetry/opentelemetry-specification/issues/2497

* **Other docs or links:**
  * Survey Results: https://opentelemetry.io/blog/2024/prometheus-compatibility-survey/
  * Slack thread: https://cloud-native.slack.com/archives/C01AUBA4PFE/p1726399373207819
  * Doc with Options: https://docs.google.com/document/d/1t4ARkyOoI4lLNdKb0ixbUz7k7Mv_eCiq7sRKHAGZ9vg
  * Prometheus PoC: https://github.com/prometheus/prometheus/compare/main...dashpole:prometheus:type_and_unit_labels

This document proposes adding the metric type and unit as labels on metrics.

## Why

Per [dev-summit consensus](https://docs.google.com/document/d/1uurQCi5iVufhYHGlBZ8mJMK_freDFKPG0iYBQqJ9fvA/edit#bookmark=id.q6upqm7itl24), we would like to avoid adding type and unit suffixes to metric names when translating from OpenTelemetry to Prometheus. These suffixes are currently required by the [compatibility specification](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/compatibility/prometheus_and_openmetrics.md). However, OpenTelemetry currently considers the metric type and unit identifying, whereas Prometheus does not. Simply removing suffixes might result in "collisions" between distinct OpenTelemetry metrics which have the same name, but different types (less commonly) or units.

### Pitfalls of the current solution

Roughly half of OpenTelemetry users surveyed preferred keeping metric names unmodified when translating to Prometheus. With our goal of being the default choice to store OpenTelemetry metrics, we should find a way to preserve metric names without introducing potential issues for users.

## Goals

Goals and use cases for the solution as proposed in [How](#how):

* [Required] OpenTelemetry users can query for the original names of their metrics.
* [Required] Users can filter by the metric type or unit in PromQL queries.
* [Required] PromQL returns a warning when querying across metrics with different units or types.
* [Nice to have] OpenTelemetry users can grep for the original names in the text exposition.

### Audience

This document is for Prometheus server maintainers, PromQL maintainers, and anyone interested in furthering compatibility between Prometheus and OpenTelemetry.

## Non-Goals

* Prometheus will not attempt to auto-convert between units when there is a conflict.
* Prometheus will not attempt to auto-convert between types (e.g. native histogram vs float series).
* Prometheus client libraries will not allow mixing metrics with the same name, but different type or unit in the exposition format. See potential future extensions.

## User Experience

When querying for a metric, users can filter for a type or unit by specifying a filter on the `__unit__` or `__type__` labels, which use the reserved `__` prefix to ensure they do not collide with user-provided labels.

For example:

* `my_metric{}` returns all series with any type or unit, including `__type__` and `__unit__` labels.
* `my_metric{__unit__="seconds", __type__="counter"}` returns only series with the specified type and unit.

When a query for a metric returns multiple metrics with a different `__type__` or `__unit__` label, but the same `__name__`, users see a warning in the UI.

Users don't see the `__type__` or `__unit__` labels in the Prometheus UI next to other labels, but the unit displayed next to the value.

Users see no difference to exposition formats.

## How

### PromQL Changes

When a query for a metric returns multiple metrics with a different `__type__` or `__unit__` label, but the same `__name__`, a warning annotation will be returned with the PromQL response.

### Prometheus UI Changes

When displaying a metric's labels in the table or in the graph views, the UI will hide the `__type__` and `__unit__` labels, similar to the current handling of `__name__`. If present, the `__unit__` label will be displayed to the right of the value of the metric: e.g. "1.25 seconds". The type will not be displayed.

### Prometheus Server Ingestion

When receiving OTLP or PRW 2.0, or when scraping the text, OM, or proto formats, the type and unit of the metric are added as the `__type__` and `__unit__` labels, if not already present.

PRW 1.0 is omitted because metadata is sent separately from timeseries, making it infeasible to add the labels at ingestion time.

### Implementation Plan

#### Milestone 1: Feature flag for adding labels

Add a feature flag: `--enable-feature=metadata-labels`. When enabled `__type__` and `__unit__` labels are added when receiving OTLP or PRW 2.0, or when scraping the text, OM, or proto formats.

#### Milestone 2: UI and PromQL changes

During this stage, implement the UI and PromQL changes above. Iterate based on feedback. Changes should have no effect unless the metadata-labels feature flag is enabled.

#### Milestone 3: Add NoNameChanges option for OTLP translation

Add an option, `NoNameChanges` for the OTLP translation strategy. When enabled, it disables UTF-8 sanitization and the addition of suffixes. In the documentation for this option, recommend that `--enable-feature=metadata-labels` is enabled.

## Alternatives

### “Real” Type and Unit suffixes in **name**

Similar to suffixes of _<unit> and _<type/total> but make it an explicit suffix using a delimiter not currently permitted in metric names. Specifying suffixes is optional when querying for a metric. When the type or unit suffix is omitted from a query, it would (design TBD) return results which include any type or unit suffix which exists for that name.

NOTE: Dot for units and slash for type is just one example, there might be better operators/characters to use.

Writing queries that include the type and unit would be recommended as a best-practice by the community.

For example:

* Querying for foo/histogram would return results that include both foo.seconds/histogram and foo.milliseconds/histogram.
* Querying for foo.seconds would return results that include both foo.seconds/histogram and foo.seconds/counter.
* Querying for http_server_duration would return results that include both foo.seconds/histogram and foo.milliseconds/counter.
* Querying for an OpenTelemetry metric, such as http.server.duration, with suffixes would require querying for ”http.server.duration”.seconds/histogram. Note that suffixes are outside of quotes.

This solution is not chosen because:

* Requires PromQL changes (intrusive), touches on “dot” operator ideas.
* Adding suffixes outside of quotes looks strange: {“http.server.duration”.seconds/histogram}
* Rolling this out would be breaking for existing Prometheus users: E.g. {foo_seconds} becomes {foo.seconds/histogram}. Could this be part of OM 2.0?
  * Mitigation: users just stay with {foo_seconds.seconds/histogram}
* Users might be surprised by, or dislike the additional suffixes and delimiters in the metric name results
  * Mitigation: Opt-in for query engines?

### "Hide" __type__ and __unit__ labels in PromQL, instead of UI

Existing UIs don't handle the `__type__` and `__unit__` labels. To mitigate this, PromQL could omit the `__type__` and `__unit__` labels from the query response. Doing this would avoid requiring UIs to update to handle the new labels.

This solution is not chosen because:

* It deviates from the current handling of the `__name__` label.
* We expect metadata, like type and unit to be useful to display in the UI, and want to enable these use-cases.
* It should be a small amount of effort to hide these labels.

## Potential Future Extensions

### __type__ and __unit__ from client libraries

One obvious extension of this proposal would be for Prometheus clients to start sending `__type__` and `__unit__` labels with the exposition format. This would:

* Allow mixing metrics with the same name, but different types and units in the same endpoint (an explicit non-goal of this proposal).
* Allow sending unit metadata in the text exposition format, which doesn't support `# UNIT` metadata.

This is excluded from this proposal because:

* OpenTelemetry users can use [Views](https://opentelemetry.io/docs/specs/otel/metrics/sdk/#view) to resolve collisions. That should be "good enough".
* Doing this would require some changes to client libraries, which is significantly more work, and is harder to experiment with.
* There can be conflicts between `# TYPE` and `# UNIT` metadata for a metric, and `__type__` and `__unit__` labels. This adds complexity to understanding the exposition format, and requires establishing rules for dealing with conflicts.

### More metadata labels

OpenTelemetry has lots of other metadata that may be a good fit for this "metadata label" pattern. For example, OpenTelemetry's scope name and version, or the schema URL are technically identifying for a time series, but are unlikely to be something that we want to display prominently in the UI.

If the pattern of adding `__type__` and `__unit__` works well for this metadata, we could consider making the pattern more generic:

* UIs should hide labels starting with `__` by default, with an option to show them.
* Introduce a mechanism to allow OTel libraries to provide additional metadata labels. However, this has the potential to introduce collisions, since `__` has been reserved thus far.  Maybe a specific allowlist could work.

## Action Plan

* [ ] Feature flag for adding labels
* [ ] UI and PromQL changes
* [ ] Add NoNameChanges option for OTLP translation