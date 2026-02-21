# Advanced Observability

This reference covers logging infrastructure configuration and telemetry handler setup. For everyday logging usage (`Rage.logger.info`, `with_context`, `tagged`), see the Structured Logging section in the main guide.

## Logging Infrastructure

### External Logger

Replace the built-in logger with a custom one by passing a callable to `config.logger`. The callable receives structured data for each log entry:

```ruby
class MyExternalLogger
  def call(severity:, tags:, context:, message:, request_info:)
    # severity: log level (e.g., :info, :error)
    # tags: array of string tags
    # context: hash of structured context
    # message: the log message string (nil for request logs)
    # request_info: hash with request details (nil for non-request logs)
  end
end

Rage.configure do
  config.logger = MyExternalLogger.new
end
```

**Sentry integration example:**

```ruby
class SentryLogger
  def call(severity:, tags:, context:, message:, request_info:)
    data = context
    tags.each do |tag|
      data = data.merge("tags.#{tag}" => "true")
    end

    if request_info
      data[:path] = request_info[:env]["PATH_INFO"]
      message = "Request processed"
    end

    Sentry.logger.log(severity, message, parameters: [], **data)
  end
end

Rage.configure do
  config.logger = SentryLogger.new
end
```

### Global Log Context

Add key-value pairs to every log entry across all requests. Useful for app version, deployment info, etc.:

```ruby
# config/application.rb
Rage.configure do
  # Static context
  config.log_context << { version: ENV["MY_APP_VERSION"] }

  # Dynamic context via callable (evaluated per request)
  config.log_context << proc do
    { trace_id: MyObservabilitySDK.trace_id, span_id: MyObservabilitySDK.span_id }
  end
end
```

Callables should return a hash or `nil`.

### Global Log Tags

Add identifiers to every log entry:

```ruby
# config/application.rb
Rage.configure do
  # Static tag
  config.log_tags << Rage.env

  # Dynamic tag via callable
  config.log_tags << proc { Fiber[:request_source] }
end
```

Tags appear as `[production][web]` (or `{ tags: ["production", "web"] }` for JSON logs) in log output.

### Request Log Enhancement

Extend the built-in request completion log with custom data using `append_info_to_payload` in controllers:

```ruby
class ApplicationController < RageController::API
  private

  def append_info_to_payload(payload)
    payload[:user_id] = current_user&.id
    payload[:tenant_id] = current_tenant&.id
    payload[:response_size] = response.body.size
  end
end
```

## Telemetry

Built-in span-based instrumentation for observing application behavior. Use telemetry to integrate with monitoring platforms, track performance metrics, debug production issues, or build custom observability solutions.

### Spans

Spans are measurement points wrapping framework operations. Span names follow a `component.entity.action` pattern:

| Span Name | Description |
|-----------|-------------|
| `controller.action.process` | HTTP request processing |
| `cable.action.process` | WebSocket channel action handling |
| `cable.connection.process` | WebSocket connection handling |
| `cable.stream.broadcast` | Cable stream broadcasting |
| `deferred.task.enqueue` | Background task enqueuing |
| `deferred.task.process` | Background task execution |
| `events.event.publish` | Event publishing |
| `events.subscriber.process` | Event subscriber handling |

See [Rage::Telemetry::Spans](https://api.rage-rb.dev/Rage/Telemetry/Spans) for the full list of spans and the parameters each one passes to handlers.

### Creating Handlers

Handlers observe spans and execute custom logic. Inherit from `Rage::Telemetry::Handler`:

```ruby
class SlowRequestHandler < Rage::Telemetry::Handler
  handle "controller.action.process", with: :monitor_duration

  def self.monitor_duration(controller:)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start

    if duration > 2.0
      Rage.logger.warn "Slow request detected",
        controller: controller.class.name,
        action: controller.action_name,
        duration_seconds: duration.round(3)
    end
  end
end
```

### Handler Mechanics

- Code before `yield` runs before the operation
- `yield` transfers control to the observed operation
- Code after `yield` runs after completion
- If you don't call `yield`, Rage calls it automatically
- Rage inspects the handler method signature and only passes the parameters you request — only declare the parameters you need
- **Do not use `**` (double-splat) to collect and forward all parameters** — some spans pass large objects (like controller instances) that should not be forwarded as is
- See [Rage::Telemetry::Spans](https://api.rage-rb.dev/Rage/Telemetry/Spans) for all available spans and the parameters each one provides

### Registering Handlers

```ruby
# config/application.rb or config/environments/*.rb
Rage.configure do
  # use `after_initialize` if referencing an app-level constant
  config.after_initialize do
    config.telemetry.use SlowRequestHandler
  end
end
```

### Handling Multiple Spans

Use separate handler methods when spans provide different parameters:

```ruby
class PerformanceHandler < Rage::Telemetry::Handler
  handle "controller.action.process", with: :track_request
  handle "deferred.task.process", with: :track_task

  def self.track_request(name:, controller:)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
    Metrics.timing(name, duration, tags: { controller: controller.class.name, action: controller.action_name })
  end

  def self.track_task(name:)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
    Metrics.timing(name, duration)
  end
end
```

#### Wildcards

Use wildcards to match multiple spans. Only request common parameters like `name:` or use default arguments:

```ruby
class CableMonitor < Rage::Telemetry::Handler
  handle "cable.*", with: :monitor_cable

  def self.monitor_cable(name:, stream: nil)
    Rage.logger.debug "Cable operation", span: name, stream: stream
    yield
  end
end
```

### Error Tracking

When operations fail, `yield` returns a `SpanResult` object:

```ruby
class ErrorHandler < Rage::Telemetry::Handler
  handle "controller.action.process", with: :track_controller_errors
  handle "deferred.task.process", with: :track_task_errors

  def self.track_controller_errors(name:, controller:)
    result = yield

    if result.error?
      ErrorTracker.capture(
        result.error,
        context: { span: name, controller: controller.class.name, action: controller.action_name }
      )
    end
  end

  def self.track_task_errors(name:)
    result = yield
    ErrorTracker.capture(result.error, context: { span: name }) if result.error?
  end
end
```

The error still propagates normally - this just gives you visibility into failures.

### Practical Examples

#### Metrics Collection

```ruby
class MetricsHandler < Rage::Telemetry::Handler
  handle "controller.action.process", with: :record_request_metrics

  def self.record_request_metrics(name:, controller:)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    result = yield
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start

    tags = {
      controller: controller.class.name,
      action: controller.action_name,
      status: result.error? ? "error" : "success"
    }

    StatsD.distribution("http.request.duration", duration, tags: tags)
    StatsD.increment("http.request.count", tags: tags)
  end
end
```

#### Database Query Tracking

```ruby
class QueryCountHandler < Rage::Telemetry::Handler
  handle "controller.action.process", with: :count_queries

  def self.count_queries(name:, controller:)
    query_count = 0

    callback = ->(*) { query_count += 1 }
    ActiveSupport::Notifications.subscribed(callback, "sql.active_record") do
      yield
    end

    if query_count > 10
      Rage.logger.warn "High query count",
        controller: controller.class.name,
        action: controller.action_name,
        query_count: query_count
    end
  end
end
```

#### Request Tracing

```ruby
class TracingHandler < Rage::Telemetry::Handler
  handle "controller.action.process", with: :trace_request

  def self.trace_request(name:, controller:)
    trace_id = SecureRandom.uuid

    Rage.logger.with_context(trace_id: trace_id) do
      Rage.logger.info "Request started", controller: controller.class.name, action: controller.action_name
      start = Process.clock_gettime(Process::CLOCK_MONOTONIC)

      result = yield

      duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
      Rage.logger.info "Request completed",
        duration_ms: (duration * 1000).round,
        success: !result.error?
    end
  end
end
```

## References

- [Logging documentation](https://rage-rb.dev/docs/logging.md)
- [Telemetry documentation](https://rage-rb.dev/docs/telemetry.md)
- [Rage::Logger API](https://api.rage-rb.dev/Rage/Logger)
- [Rage::Telemetry::Handler API](https://api.rage-rb.dev/Rage/Telemetry/Handler)
- [Rage::Telemetry::Spans](https://api.rage-rb.dev/Rage/Telemetry/Spans) — all available span names and the parameters passed to handlers
