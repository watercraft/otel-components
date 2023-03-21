# ScienceLogic Format Processor

| Status                   |                       |
| ------------------------ | --------------------- |
| Stability                | logs [beta]           |
| Supported pipeline types | logs                  |
| Distributions            | [contrib]             |

The ScienceLogic format processor accepts logs and places them into
batches annotated with the attributes required for processing by
other ScienceLogic components.  Each batch is forwarded as a
resource log entry with resource attributes that identify the log
stream, i.e. from an instance of an application running on a single
host.  The configuration describes where to find the appropriate
attributes and how to present them in the metadata attributes.
To work with different receivers, you can define multiple `profiles`
that match against incoming logs in the order configured.  A match
requires the following attributes:

- `service_group`: Domain of anomaly correlation
- `source`: Source within the domain, typically a host name
- `log_type`: Application generating the logs, e.g. Postgres

Additional optional attributes can be configured as `labels`.
These attributes can be derived from the following sources in the
incoming stream:

- `lit`: A literal injected from the configuration
- `rattr`: Resource attribute, also called resource label
- `attr`: Log record attribute
- `body`: The message body, when the body is a map, e.g. Windows Event Log stream

The syntax for associating metadata looks like:

```<destination>: <source>:<list of keys to try>:<optional replacement key>:<options>```

A special value `-` for the replacement key will omit the attribute
in the downstream ScienceLogic metadata attributes.  This can be used,
for example, to batch together logs of the same type from different hosts.

The following options are supported:

- `rmprefix`: To remove a prefix if matched, e.g. `rmprefix=Microsoft-Windows-`
- `alphanum`: Filter out all characters that are not letters or numbers
- `lc`: Transform to lowercase

The attributes assigned by this processor for consumption by
ScienceLogic commponents include the following resource attributes:

- `sl_logtype`: Resulting formated log type
- `sl_metadata`: An encoding of all log stream metadata

And the following log record attribute:

- `sl_msg`: The log message body formatted for consumption by ScienceLogic

The following configuration options can be modified:
- `send_batch_size` (default = 8192): Number of spans, metric data points, or log
records after which a batch will be sent regardless of the timeout.
- `timeout` (default = 200ms): Time duration after which a batch will be sent
regardless of size.
- `send_batch_max_size` (default = 0): The upper limit of the batch size.
  `0` means no upper limit of the batch size.
  This property ensures that larger batches are split into smaller units.
  It must be greater than or equal to `send_batch_size`.

Examples:

```yaml
processors:
  sllogformat:
    send_batch_size: 10000
    timeout: 10s
    profiles:
    - service_group: lit:default:ze_deployment_name # windows event log
      source: body:computer:zid_host
      log_type: body:provider.name:zid_log:rmprefix=Microsoft-Windows-:alphanum:lc
      labels:
      - body:channel:win_channel
      - body:keywords:win_keywords
      message: body:message|event_data|keywords
      format: event
    - service_group: lit:default:ze_deployment_name # docker logs
      source: rattr:host.name:zid_host
      log_type: attr:container_id:zid_log
      labels:
      - rattr:os.type
      - attr:log.file.path:zid_path
      message: body
      format: message
```

[beta]: https://github.com/open-telemetry/opentelemetry-collector#beta
[contrib]: https://github.com/open-telemetry/opentelemetry-collector-releases/tree/main/distributions/otelcol-contrib
[core]: https://github.com/open-telemetry/opentelemetry-collector-releases/tree/main/distributions/otelcol
