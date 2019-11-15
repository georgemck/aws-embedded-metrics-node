# aws-embedded-metrics

![](https://codebuild.us-west-2.amazonaws.com/badges?uuid=eyJlbmNyeXB0ZWREYXRhIjoiRWFRdGtyUGw4a0JyaUR3THF4cTZxU2J6aEE1RVJFdmpxcUk5ekFHdUwzMnJXa1dYRmpzKzBCZlBNMU41cVkwNTNsQjZieUVGc3FGbUw1eHovTERrMStVPSIsIml2UGFyYW1ldGVyU3BlYyI6IjFuQ0VXN2l4YnNVMVpYMHIiLCJtYXRlcmlhbFNldFNlcmlhbCI6MX0%3D&branch=master)
[![](https://img.shields.io/npm/v/aws-embedded-metrics.svg)](https://www.npmjs.com/package/aws-embedded-metrics)

A new package from Amazon CloudWatch that allows you to generate CloudWatch Metrics from structured log events. The embedded metrics will be extracted so you can visualize and alarm on them for real-time incident detection. This allows you to monitor aggregated values while preserving the detailed event context that generated them.

## Use Cases

- **Asynchronous emission of metrics from Lambda functions**
  There are two natively supported options for emitting metrics from Lambda today: executing synchronous calls to CloudWatch via PutMetricData or extracting metrics from your function logs through CloudWatch Logs Metric Filters. The first couples the TPS of your function to your PutMetricData TPS and also blocks function execution while waiting on a response from CloudWatch Metrics. The second requires you to make a control plane call, forces you to keep code and configuration synchronized. These are no longer problems if you use CloudWatch Embedded Metrics since your metric definitions are included in the log data.
- **Linking metrics to high cardinality context**
  Using the Embedded Metric Format, you will be able to extract metrics and configure alarms on those metrics, but also be able to jump back to the logs—using [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)—that emitted those metrics to view high cardinality event context.

## Installation

```
npm install aws-embedded-metrics
```

## Usage

To get a metric logger, you can either decorate your function with a metricScope, or manually create and flush the logger.

Using the metricScope decorator:

```js
const { metricScope, Unit } = require("aws-embedded-metrics");

const myFunc = metricScope(metrics => 
  async () => {
    metrics.putDimensions({ Service: "Aggregator" });
    metrics.putMetric("ProcessingLatency", 100, Unit.Milliseconds);
    metrics.setProperty("RequestId", "422b1569-16f6-4a03-b8f0-fe3fd9b100f8");
    // ...
  });

await myFunc();
```

Manually constructing and flushing the logger:

```js
const { createMetricsLogger, Unit } = require("aws-embedded-metrics");

const myFunc = async () => {
  const metrics = createMetricsLogger();
  metrics.putDimensions({ Service: "Aggregator" });
  metrics.putMetric("ProcessingLatency", 100, Unit.Milliseconds);
  metrics.setProperty("RequestId", "422b1569-16f6-4a03-b8f0-fe3fd9b100f8");
  // ...
  metrics.flush();
};

await myFunc();
```

### Lambda

If you are running on Lambda, export your function like so:

```js
const { metricScope } = require("aws-embedded-metrics");

const myFunc = metricScope(metrics => 
  async () => {
    // ...
  });

exports.handler = myFunc();
```

## API

### MetricLogger

The `MetricLogger` is the interface you will use to publish embedded metrics.

- **putMetric**(String name, Double value, Unit? unit)

Adds a new metric to the current logger context. Multiple metrics using the same key will be appended to an array of values. The Embedded Metric Format supports a maximum of 100 values per key. If more metric values are added than are supported by the format, the logger will be flushed to allow for new metric values to be captured.

Requirements:
- Name Length 1-255 characters
- Name must be ASCII characters only
- Values must be in the range of 8.515920e-109 to 1.174271e+108. In addition, special values (for example, NaN, +Infinity, -Infinity) are not supported.
- Units must meet CW Metrics unit requirements, if not it will default to None.

Examples:

```js
putMetric("Latency", 200, Unit.Milliseconds)
```

- **setProperty**(String key, Object value)

Adds or updates the value for a given property on this context. This value is not submitted to CloudWatch Metrics but is searchable by CloudWatch Logs Insights. This is useful for contextual and potentially high-cardinality data that is not appropriate for CloudWatch Metrics dimensions.

Requirements:
- Length 1-255 characters
    
Examples: 
```js
setProperty("RequestId", "422b1569-16f6-4a03-b8f0-fe3fd9b100f8")
setProperty("InstanceId", "i-1234567890")
setProperty("Device", {
  Id: "61270781-c6ac-46f1-baf7-22c808af8162",
  Name: "Transducer",
  Model: "PT-1234"
});
```
- **putDimensions**(Record<String, String> dimensions)

Adds a new set of dimensions that will be associated to all metric values.

**WARNING**: Every distinct value will result in a new CloudWatch Metric. 
If the cardinality of a particular value is expected to be high, you should consider
using `setProperty` instead.

Requirements:
- Length 1-255 characters
- ASCII characters only

Examples: 
```js
putDimensions({ Operation: "Aggregator" })
putDimensions({ Operation: "Aggregator", DeviceType: "Actuator" })
```

- **setDimensions**(Record<String, String>[] dimensions...)

Explicitly override all dimensions. This will remove the default dimensions.

**WARNING**: Every distinct value will result in a new CloudWatch Metric. 
If the cardinality of a particular value is expected to be high, you should consider
using `setProperty` instead.

Requirements:
- Length 1-255 characters
- ASCII characters only

Examples:

```js
setDimensions(
  { Operation: "Aggregator" },
  { Operation: "Aggregator", DeviceType: "Actuator" })
```

- **setNamespace**(String value)

Sets the CloudWatch [namespace](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html#Namespace) that extracted metrics should be published to. If not set, a default value of aws-embedded-metrics will be used.

Requirements:

- Name Length 1-255 characters
- Name must be ASCII characters only

Examples:

```py
setNamespace("MyApplication");
```

- **flush**()

Flushes the current MetricsContext to the configured sink and resets all properties, dimensions and metric values. The namespace and default dimensions will be preserved across flushes.

### Configuration

All configuration values can be set using environment variables with the prefix (`AWS_EMF_`). Configuration should be performed as close to application start up as possible.

**ServiceName**: Overrides the name of the service. For services where the name cannot be inferred (e.g. Java process running on EC2), a default value of Unknown will be used if not explicitly set.

Requirements:

- Name Length 1-255 characters
- Name must be ASCII characters only

Example:

```js
// in process
const { Configuration } = require("aws-embedded-metrics");
Configuration.serviceName = "MyApp";

// environment
AWS_EMF_SERVICE_NAME=MyApp
```

**ServiceType**: Overrides the type of the service. For services where the type cannot be inferred (e.g. Java process running on EC2), a default value of Unknown will be used if not explicitly set.

Requirements:

- Name Length 1-255 characters
- Name must be ASCII characters only

Example: 

```js
// in process
const { Configuration } = require("aws-embedded-metrics");
Configuration.serviceType = "NodeJSWebApp";

// environment
AWS_EMF_SERVICE_TYPE=NodeJSWebApp
```

**LogGroupName**: For agent-based platforms, you may optionally configure the destination log group that metrics should be delivered to. This value will be passed from the library to the agent in the Embedded Metric payload. If a LogGroup is not provided, the default value will be derived from the service name: <service-name>-metrics

Requirements:
- Name Length 1-512 characters
- Log group names consist of the following characters: a-z, A-Z, 0-9, '\_' (underscore), '-' (hyphen), '/' (forward slash), and '.' (period). Pattern: [\.\-_/#A-Za-z0-9]+

Example:

```js
// in process
const { Configuration } = require("aws-embedded-metrics");
Configuration.logGroupName = "LogGroupName";

// environment
AWS_EMF_LOG_GROUP_NAME=LogGroupName
```

**LogStreamName**: For agent-based platforms, you may optionally configure the destination log stream that metrics should be delivered to. This value will be passed from the library to the agent in the Embedded Metric payload. If a LogGroup is not provided, the default value will be derived by the agent (this will likely be the hostname).

Requirements:

- Name Length 1-512 characters
- The ':' (colon) and '\*' (asterisk) characters are not allowed. Pattern: [^:]*

Example:

```js
// in process
const { Configuration } = require("aws-embedded-metrics");
Configuration.logStreamName = "LogStreamName";

// environment
AWS_EMF_LOG_STREAM_NAME=LogStreamName
```

## Examples

Check out the [examples](https://github.com/awslabs/aws-embedded-metrics-node/tree/master/examples) directory to get started.

## Development

### Building

This project uses [Volta](https://volta.sh/) to pin the currently supported version of node.

```
npm i && npm run build
```

### Running Locally

If you are running the CW agent locally, you can test the workflow:

```
npm i && npm link
cd examples/agent && npm link aws-embedded-metrics
```

After linking you'll need to rebuild any changes:

```
npm run build
```

### Testing

We have 2 different types of tests:
1. Unit tests which can be run using the following commands:
  ```sh
  npm test
  # or
  npm run watch
  ```
2. Integration tests. These tests require Docker to run the CloudWatch Agent and valid AWS credentials. Tests can be run by:
  ```
  export AWS_ACCESS_KEY_ID=
  export AWS_SECRET_ACCESS_KEY=
  export AWS_REGION=us-west-2
  npm run integ
  ```

### Formatting

We use [Prettier](https://prettier.io/) for auto-formatting.
You should install the plugin for your editor-of-choice and enabled format-on-save.

## License

This project is licensed under the Apache-2.0 License.
