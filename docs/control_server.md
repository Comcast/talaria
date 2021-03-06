---
title: Control Talaria Server
sort_rank: 2
---

Talaria exposes a built-in control server that can be used to adjust certain features.

## Device Gate
Talaria can be set to disallow incoming websocket connections.
When the gate is closed, all incoming websocket connection requests are rejected with a **503** status.
Talaria always starts with the gate open, allowing new websocket connections.
Already connected websockets are not affected by closing the gate.

The RESTful endpoint that controls this is `/api/v2/device/gate`.
Note that the control_port for the control server is not the same as the port for the websocket server.

* `GET host:control_port/api/v2/device/gate` returns a JSON message indicating the status of the gate.
For example, `{"open": true, "timestamp": "2009-11-10T23:00:00Z"}` indicates that the gate
is open and has been open since the given timestamp.  
Similarly, `"open": false` indicates that the gate is closed.

* `POST/PUT/PATCH host:control_port/api/v2/device/gate?open=<boolean>` raises or lowers the gate.
The value of the open parameter may be anything that `golang` will parse as a boolean (see [ParseBool](https://godoc.org/strconv#ParseBool)).
This endpoint is idempotent.
Any attempt to open the gate when it is already open or close it when it is already closed results in a **200** status.
If this endpoint did change the status of the gate, a **201** status is returned.

### Gate Filtering
In addition to disallowing all incoming websocket connection requests, Talaria can also reject incoming websocket connection requests that match specific device metadata parameters.

* `GET host:control_port/api/v2/device/gate/filter` returns a JSON message with the current filters that are in place.
For example, 

```
{
  "filters": {
    "partner-id": [
      "test",
      "comcast",
    ]
  },
  "allowedFilters": ["partner-id"]
}
```

indicates that all incoming websocket connection requests that have partner-ids of either test or comcast are not allowed. Furthermore, `allowedFilters` shows that this talaria only allows requests to be filtered by `partner-id`. If `allowedFilters` is null, that means that this talaria accepts any filter keys to filter requests by.

* `POST/PUT host:control_port/api/v2/device/gate/filter` adds or updates to the list of filters in place. The request body must be in JSON format with the following attributes: 
  * `key` - Required. The parameter to filter connection requests by (can think of this as the metadata key)
  * `values` - Required. This is an array of strings. These are the metadata values to filter requests by 

An example request:

```
{
  "key": "partner-id",
  "values": ["comcast", "sky"]
}
```

This will prevent any devices with a partner-id of comcast or sky from connecting. Example responses:

HTTP/1.1 201 Created

Response Body:
```{
  "filters": {
    "partner-id": [
      "comcast",
      "sky"
    ]
  },
  "allowedFilters": ["partner-id"]
}
```
The above response would indicate a filter key has been created. The response body shows the updated state of the filters.

HTTP/1.1 200 OK

Response Body:
```{
  "filters": {
    "partner-id": [
      "comcast",
      "sky"
    ]
  },
  "allowedFilters": ["partner-id"]
}
```
The above response would indicate the filter key already exists and has been updated. The response body shows the updated state of the filters.

Note that this request completely upserts the values connected to a filter key, if the filter key already previously existed.


* `DELETE host:control_port/api/v2/device/gate/filter` Deletes a filter key and the values associated with it from the list of filters. The request body must be in JSON format with the following attribute: 
  * `key` - Required. The filter key to delete (can think of this as the metadata key)

An example request:

```
{
  "key": "partner-id",
}
```

The request would mean that the talaria will no longer gate device connection requests by the partner-id parameter

An example response:

HTTP/1.1 200 OK

Response Body:
```{
  "filters": {},
  "allowedFilters": ["partner-id"]
}
```

This shows a successful delete request. This specific response body means that there are currently no filters in place after the `DELETE` request.

### Metrics
`xmidt_talaria_gate_status` is the exposed Prometheus metric that indicates the status of the gate.
When this gauge is 0.0, the gate is closed.  When this gauge is 1.0, the gate is open.

## Connection Drain
Talaria supports the draining of websocket connections.
Essentially, this means shedding load in a controlled fashion.
Websocket connections can be drained at a given rate, e.g. `100 connections/minute`,
or can be drained as fast as possible.

Only 1 drain job is permitted to be running at any time.
Attempts to start a drain when one is already active results in an error.

**IMPORTANT**:  The device gate may be open when a drain is started.
That means that devices can connect and disconnect during a drain.
In order to prevent a situation where the drain job cannot ever complete,
computations about the number of devices are done once when the job is started.
For example, if a drain job is instructed to drain all devices, the count of devices
is computed at the start of the job and exactly that many devices are drained.
This may mean that connections are left at the end of a drain when the gate is not closed.
If this behavior is not desired, *close the device gate before starting a drain.*

The RESTful endpoint that controls the connection drain is `/api/v2/device/drain`.
Note that the control_port for the control server is not the same as the port for the websocket server.

* `GET host:control_port/api/v2/device/drain` returns a JSON message indicating whether
a drain job is active and the progress of the active job if one is running.
If a drain has previously completed, the information about that job will be
available via this endpoint until a new drain job is started. This will not
start a new drain.
```json
{
    "active": false,
    "job": {
        "count": 0
    },
    "progress": {
        "visited": 0,
        "drained": 0,
        "started": "0001-01-01T00:00:00Z"
    }
}
```

* `POST/PUT/PATCH host:control_port/api/v2/device/drain` attempts to start a drain job.
This endpoint returns the same JSON message as a `GET` when it starts a drain job,
along with a **200** status.  If a drain job is already running, this endpoint returns a **429 Conflict** status.
If no parameters are supplied, all devices are drained as fast as possible.
Several parameters may be supplied to customize the drain:

    + `count`: The maximum number of websocket connections to close.
    If this value is larger than the number of connections at the start of the job,
    the current count of connections is used instead.
    + `percent`: The percentage of connections to close.
    The computation of how many connections is done once when the job is started.
    If both `count` and `percent` are supplied, `count` is ignored.
    + `rate`: The number of connections per unit time (tick) to close.
    If this value is not supplied, connections are closed as fast as possible.
    + `tick`: The unit of time for `rate`.
    If `rate` is supplied and `tick` is not supplied, a `tick` of `1s` is used.
    If `rate` is not supplied and `tick` is supplied, `tick` is ignored.
    The value of `tick` may be anything parseable by the `golang` standard library (see [ParseDuration](https://godoc.org/time#ParseDuration)).

  For example:
  ```json
    {
      "tick": "2s",
      "rate": 4
    }
  ```
  will drain 4 devices every 2 seconds. This is the same as saying 2 devices
  every second. Another example is with the request `?tick=1m&rate=30`, meaning
  Every minute 30 devices will be remove. Since the draining of devices are
  spread out over the tick period, the 1m tick at a rate of 30 is the same as
  a tick of 2s and a rate of 1.

  If attempting to drain by a specific device parameter in the metadata, such as `partner-id`, supply a body to the request, like so:

  ```
  {
    "key": "partner-id",
    "values": ["comcast"]
  }
  ```

  This request will drain all devices that have comcast as the partner-id.


* `DELETE host:control_port/api/v2/device/drain` attempts to cancel any running drain job.
Note that a running job may not cancel immediately.
If no drain job is running, **429 Conflict** is returned.

### Metrics

Two Prometheus metrics are exposed to monitor the drain feature:

* `xmidt_talaria_drain_count` is the total number of connections that were
closed due to a drain since the server was started.
* `xmidt_talaria_drain_status` is a gauge indicating whether a drain is running.
This gauge will be `0.0` when no drain job is running, and `1.0` when a drain job is active.
