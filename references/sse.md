# Rage SSE (Server-Sent Events)

Use `render sse:` for one-way server-to-client streaming over HTTP.

## Patterns

Rage supports three SSE patterns:

- **Enumerator streams** - finite multi-message responses
- **One-off updates** - one SSE message, then close
- **Unbounded streams** - long-lived connections that receive broadcasts

## Enumerator Streams

Render an enumerator to stream multiple SSE messages over time:

```ruby
class MessagesController < RageController::API
  def index
    stream = Enumerator.new do |y|
      "Hello, world!".each_char do |char|
        sleep 1
        y << char
      end
    end

    render sse: stream
  end
end
```

Rage closes the connection automatically when the enumerator finishes.

### External Sources

Enumerators work well with Redis, message queues, and polling loops:

```ruby
class MessagesController < RageController::API
  def index
    redis = Redis.new

    stream = Enumerator.new do |y|
      loop do
        _, message = redis.blpop("messages", timeout: 5)
        break if message == "close"
        y << message
      end
    ensure
      redis.close
    end

    render sse: stream
  end
end
```

Yielding `nil` is safe. Rage ignores it.

### Object Streaming

Hashes, arrays, and other Ruby objects are serialized to JSON automatically:

```ruby
stream = Enumerator.new do |y|
  loop do
    _, message = redis.blpop("events")
    break if message == "close"
    y << { event: "message", data: message, timestamp: Time.now.to_i }
  end
end

render sse: stream
```

## SSE Fields

Use `Rage::SSE.message` for SSE fields like `id`, `event`, and `retry`:

```ruby
file = File.new(params[:file], "r")

stream = file.each_line.with_index.lazy.map do |line, i|
  Rage::SSE.message(line, id: i, event: "line", retry: 100)
end

render sse: stream
```

Example frame:

```text
id: 0
event: line
retry: 100
data: First line of the file
```

`Rage::SSE.message` also works with object payloads:

```ruby
stream = Enumerator.new do |y|
  loop do
    _, message = redis.blpop("notifications")
    break if message == "close"
    y << Rage::SSE.message({ notification: message }, event: "notification")
  end
end

render sse: stream
```

## Connection Lifecycle

- Rage sends periodic `: ping` comments to keep idle SSE connections alive.
- Enumerator streams get up to 15 seconds to finish during server restart.
- Unbounded streams are interrupted immediately on restart.

## One-Off Updates

Pass any value directly to `render sse:` to send one message and close:

```ruby
user = User.find(params[:id])
render sse: user
```

## Unbounded Streams

Use `Rage::SSE.stream` for long-lived connections that receive broadcasts later:

```ruby
render sse: Rage::SSE.stream("notifications-#{params[:user_id]}")
```

### Broadcasting

Broadcast from controllers, models, or deferred tasks:

```ruby
Rage::SSE.broadcast("notifications-#{user.id}", user.notifications.last)
```

Include SSE fields when needed:

```ruby
notification = user.notifications.last

Rage::SSE.broadcast(
  "notifications-#{user.id}",
  Rage::SSE.message(notification, id: notification.id, event: "notification")
)
```

### Closing Streams

Close all active connections for a stream:

```ruby
Rage::SSE.close_stream("notifications-#{user.id}")
```

### Composite Keys

Arrays make stream names easier to organize:

```ruby
# Controller
render sse: Rage::SSE.stream([:notifications, params[:user_id]])

# Broadcasting
Rage::SSE.broadcast([:notifications, user.id], user.notifications.last)

# Closing
Rage::SSE.close_stream([:notifications, user.id])
```

## Multi-Process and Multi-Server Setup

### Single Server, Multiple Processes

No configuration needed - Rage uses IPC (inter-process communication) automatically.

### Multiple Servers

Use the Redis pubsub adapter when broadcasts must cross processes or servers, or when external systems like Sidekiq need to publish into Rage SSE streams.

Configure Redis Adapter in `config/pubsub.yml`:

```yaml
# config/pubsub.yml
production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL", "redis://localhost:6379") %>
  channel_prefix: myapp_production
```

With this configured, `Rage::SSE.broadcast` and `Rage::SSE.close_stream` synchronize across all app instances.

## Buffering

Messages can be lost if broadcasting starts before the stream object exists:

```ruby
FetchNotifications.perform_async
render sse: Rage::SSE.stream("notifications")
```

Create the stream first when early broadcasts are possible. Rage buffers messages for known streams until the response starts:

```ruby
stream = Rage::SSE.stream("notifications")
FetchNotifications.perform_async
render sse: stream
```

## Low-Level Proc Access

Use a proc only when you need raw access to the SSE connection:

```ruby
render sse: ->(connection) do
  connection.write("data: Hello, world!\n\n")
ensure
  connection.close
end
```

With a proc, you are responsible for:

- Formatting valid SSE frames
- Managing the connection lifecycle
- Closing the connection

## References

- [Server-Sent Events documentation](https://rage-rb.dev/docs/sse.md)
