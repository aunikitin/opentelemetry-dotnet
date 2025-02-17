# OpenTelemetry .NET Metrics

## Best Practices

### Instruments should be singleton

Instruments SHOULD only be created once and reused throughout the application
lifetime. This [example](../../docs/metrics/getting-started-console/Program.cs)
shows how an instrument is created as a `static` field and then used in the
application. You could also look at this ASP.NET Core
[example](../../examples/AspNetCore/Program.cs) which shows a more Dependency
Injection friendly way of doing this by extracting the `Meter` and an instrument
into a dedicated class called
[Instrumentation](../../examples/AspNetCore/Instrumentation.cs) which is then
added as a `Singleton` service.

### Ordering of Tags

When emitting metrics with tags, DO NOT change the order in which you provide
tags. Changing the order of tag keys would increase the time taken by the SDK to
record the measurement.

```csharp
// If you emit the tag keys in this order: name -> color -> taste, stick to this order of tag keys for subsequent measurements.
MyFruitCounter.Add(5, new("name", "apple"), new("color", "red"), new("taste", "sweet"));
...
...
...
// Same measurement with the order of tags changed: color -> name -> taste. This order of tags is different from the one that was first encountered by the SDK.
MyFruitCounter.Add(7, new("color", "red"), new("name", "apple"), new("taste", "sweet")); // <--- DON'T DO THIS
```

### Use TagList where appropriate

For the best performance, it is highly recommended to pass in tags in certain
ways so allocations are only happening on the stack rather than the heap,
which eliminates pressure on the GC (garbage collector):

- When reporting measurements with 3 tags or less,
  emit the tags individually.
- When reporting measurements with more than 3 tags, use
  [`TagList`](https://learn.microsoft.com/dotnet/api/system.diagnostics.taglist?view=net-7.0#remarks)
  for better performance.

```csharp
var tags = new TagList
{
    { "DimName1", "DimValue1" },
    { "DimName2", "DimValue2" },
    { "DimName3", "DimValue3" },
    { "DimName4", "DimValue4" },
};

// Uses a TagList as there are more than three tags
counter.Add(100, tags); // <--- DO THIS

// Avoid the below mentioned approaches when there are more than three tags
var tag1 = new KeyValuePair<string, object>("DimName1", "DimValue1");
var tag2 = new KeyValuePair<string, object>("DimName2", "DimValue2");
var tag3 = new KeyValuePair<string, object>("DimName3", "DimValue3");
var tag4 = new KeyValuePair<string, object>("DimName4", "DimValue4");

counter.Add(100, tag1, tag2, tag3, tag4); // <--- DON'T DO THIS

var readOnlySpanOfTags = new KeyValuePair<string, object>[4] { tag1, tag2, tag3, tag4};
counter.Add(100, readOnlySpanOfTags); // <--- DON'T DO THIS
```

- When emitting metrics with more than eight tags, the SDK allocates memory on
the hot-path. You SHOULD try to keep the number of tags less than or equal to
eight. If you are exceeding this, check if you can model some of the tags as
Resource, as [shown here](#modeling-static-tags-as-resource).

### Modeling static tags as Resource

Tags such as `MachineName`, `Environment` etc. which are static throughout the
process lifetime should be be modeled as `Resource`, instead of adding them to
each metric measurement. Refer to this
[doc](./customizing-the-sdk/README.md#resource) for details and examples.

## Common issues that lead to missing metrics

- The `Meter` used to create the instruments is not added to the
  `MeterProvider`. Use `AddMeter` method to enable the processing for the
  required metrics.
- Instrument name is invalid. When naming instruments, ensure that the name you
  choose meets the criteria defined in the
  [spec](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/metrics/api.md#instrument-name-syntax).
  A few notable characters that are not allowed in the instrument name: `/`
  (forward slash), `\` (backward slash), any space character in the name.
- MetricPoint limit is reached. By default, the SDK limits the number of maximum
  MetricPoints (unique combination of keys and values for a given Metric stream)
  to `2000`. This limit can be configured using
  `SetMaxMetricPointsPerMetricStream` method. Refer to this
  [doc](../../docs/metrics/customizing-the-sdk/README.md#changing-maximum-metricpoints-per-metricstream)
  for more information. The SDK would not process any newer unique key-value
  combination that it encounters, once this limit is reached.
- MeterProvider is disposed. You need to ensure that the `MeterProvider`
  instance is kept active for metrics to be collected. In a typical application,
  a single MeterProvider is built at application startup, and is disposed of at
  application shutdown. For an ASP.NET Core application, use `AddOpenTelemetry`
  and `WithMetrics` methods from the `OpenTelemetry.Extensions.Hosting` package
  to correctly setup `MeterProvider`. Here's a [sample ASP.NET Core
  app](../../examples/AspNetCore/Program.cs) for reference. For simpler
  applications such as Console apps, refer to this
  [example](../../docs/metrics/getting-started-console/Program.cs).
