# loadshed #

**Proportional HTTP request rejection based on system load.**

## Options ##

This package exports a middleware via the `NewMiddleware()` method that returns
a `func(http.Handler) http.Handler` which should be compatible with virtually
any mux implementation or middleware chain management tool. By default, the
generated middleware is a passthrough. Load shedding based on system metrics
is enabled by passing to the constructor one or more options.

### CPU ###

The `CPU` option enables rejection of new requests based on CPU usage of the
host.

```golang
var lowerThreshold = 0.6
var upperThreshold = 0.8
var pollingInterval = time.Second
var windowSize = 10
var middleware = loadshed.NewMiddleware(
  loadshed.CPU(lowerThreshold, upperThreshold, pollingInterval, windowSize),
)
```

The above configures the load shedding middleware to record a 10 second, rolling
window of CPU usage data. As long as the average CPU usage within the window
is below the `lowerThreshold` value then all new requests pass through to the
wrapped handler. Once the rolling window exceed the `lowerThreshold` then the
middleware will begin rejecting requests with a `503` response at a rate
proportional to the distance of the average between the two thresholds. Once the
value exceed the upper threshold then all new requests are rejected until it
lowers again.

### Concurrency ###

The `Concurrency` option enables rejections of new requests when there are too
many requests currently in flight.

```golang
var lowerThreshold = 2500
var upperThreshold = 5000
var wg = loadshed.NewWaitGroup()
var middleware = loadshed.NewMiddleware(
  loadshed.Concurrency(lowerThreshold, upperThreshold, wg),
)
```

The above configures the load shedding middleware to track in-flight requests
being handled by the server. The middleware will begin rejecting a proportional
number of new requests between the lower and upper thresholds like the CPU
option above.

For convenience, this package exposes a wrapper around the
`sync.WaitGroup` feature in the standard library that wraps it in an interface
compatible with the metric aggregation system used by the middleware. The
`loadshed.WaitGroup.Add()` method will be called on every new request and the
corresponding `Done()` call as each request completes. This is intended to
act as a drop-in replacement for graceful shutdown uses of `sync.WaitGroup`.

### AverageLatency ###

The `AverageLatency` option enables rejection of new requests when the average
latency of all requests within a rolling time window is too high.

```golang
var lowerThreshold = .2
var upperThreshold = 1.0
var bucketSize = time.Second
var buckets = 10
var preallocationHint = 2000
var middleware = loadshed.NewMiddleware(
  loadshed.AverageLatency(lowerThreshold, upperThreshold, bucketSize, buckets, preallocationHint, requiredPoints),
)
```

The above configures the load shedding middleware to track the duration of
handling requests. It records the information into `bucketSize` segments of
time and keeps a rolling window of `buckets` number of segments. The above,
for example, keeps a 10 second rolling window with a granularity of 1 second
intervals.

The upper and lower thresholds are the time, in fractional seconds, that it
takes to execute the wrapped handler. As the average latency of all requests
in the window grows beyond the lower threshold then the middleware will begin
rejecting new requests. If the latency exceeds the upper threshold then all new
requests will be rejected until the average drops again. This will happen over
time either as outliers expire or until the entire window has rolled.

The `requiredPoints` value sets the minimum number of data points recorded in
the window before the filter takes effect. This is to help ensure that a
sufficient number of data points are collected to satisfy the aggregate before
a service begins denying new requests.

The `preallocationHint` is an optional optimisation for the internals of the
rolling window. It should be set to the projected number of data points that
will be contained within each bucket of the window. For example, the above
service expects to receive approximately 2,000 requests per second. This value
is only an optimisation and can be left as `0` if the projected rate is not
known.

### PercentileLatency ###

The `PercentileLatency` option works exactly the same as the `AverageLatency`
option except that it is based on a rolling percentile calculation rather than
an average.

```golang
var lowerThreshold = .2
var upperThreshold = 1.0
var bucketSize = time.Second
var buckets = 10
var preallocationHint = 2000
var percentile = 95.0
var middleware = loadshed.NewMiddleware(
  loadshed.PercentileLatency(lowerThreshold, upperThreshold, bucketSize, buckets, preallocationHint, requiredPoints, percentile),
)
```

### Callback ###

The `Callback` option enables redirection of rejected requests to a custom
`http.Handler`. This is for cases where the default `503` response is
insufficient.

```golang
var cb = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request){
  w.Header().Set("Content-Type", "application/json")
  w.WriteHeader(http.StatusServiceUnavailable)
  w.Write(`{"error": "service overloaded"}`)
})
var middleware = loadshed.NewMiddleware(
  loadshed.Callback(cb),
)
```

### Aggregator ###

The `Aggregator` enables injection of custom metrics that are not already
included in this package. The option relies on the `Aggregator` interface
provided by `bitbucket.org/atlassian/rolling` and the given aggregator must
return a value that is a percentage of requests to reject between 0.0 and 1.0.

```golang

// Inject a random amount of chaos when the system is not under load.
type chaosAggregator struct {
  amount float64
}

func (a *chaosAggregator) Aggregate() float64 {
  return a.percent
}

var middleware = loadshed.NewMiddleware(
  loadshed.Aggregator(&chaosAggregator{.01})
)
```

## Contributing ##

### License ###

This project is licensed under Apache 2.0. See LICENSE.txt for details.

### Contributing Agreement ###

Atlassian requires signing a contributor's agreement before we can accept a
patch. If you are an individual you can fill out the
[individual CLA](https://na2.docusign.net/Member/PowerFormSigning.aspx?PowerFormId=3f94fbdc-2fbe-46ac-b14c-5d152700ae5d).
If you are contributing on behalf of your company then please fill out the
[corporate CLA](https://na2.docusign.net/Member/PowerFormSigning.aspx?PowerFormId=e1c17c66-ca4d-4aab-a953-2c231af4a20b).
