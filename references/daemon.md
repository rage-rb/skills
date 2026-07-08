# Rage::Daemon

Use `Rage::Daemon` for long-lived background processes that should run alongside the app: queue listeners, streaming API consumers, persistent external connections, supervised commands, and recurring loops with adaptive timing.

## Lifecycle

Rage manages daemon lifecycle:

- Daemons start when the server boots.
- If `perform` exits or raises, Rage restarts the daemon with exponential backoff.
- If a worker process dies, a `:node` daemon restarts in another worker.
- On shutdown, Rage stops daemons gracefully and calls `cleanup`.

Each restart uses a new daemon instance. Put connection setup in `initialize` when the connection should belong to one daemon run, and release it in `cleanup`.

## Defining Daemons

```ruby
class RedisListener < Rage::Daemon
  def initialize
    @redis = Redis.new
  end

  def perform
    @redis.subscribe("notifications") do |on|
      on.message do |_channel, message|
        Rage::Cable.broadcast("notifications", message)
      end
    end
  end

  def cleanup
    @redis.close
  end
end
```

`perform` should contain the blocking operation or main loop. `cleanup` runs when `perform` returns or raises, and when the server shuts down.

## Registering Daemons

Register daemon classes in configuration:

```ruby
# config/application.rb
Rage.configure do
  config.daemons << RedisListener
  config.daemons << MetricsExporter
end
```

## Scope

By default, daemons use `:node` scope: one instance per server, coordinated across workers. Use `scope :worker` when each worker should run its own daemon instance:

```ruby
class LocalCacheWarmer < Rage::Daemon
  scope :worker

  def perform
    loop do
      LocalCache.refresh
      sleep 60
    end
  end
end
```

Valid scopes are `:node` and `:worker`.

## Stopping Without Restart

Returning anything except `Stop` from `perform` triggers a restart. Return `Stop` when the daemon has intentionally completed and should not restart:

```ruby
class BackfillDaemon < Rage::Daemon
  def perform
    while (record = Backfill.next_pending)
      process(record)
    end

    Stop
  end
end
```

## Supervising External Processes

Daemons can supervise a blocking command. If the command exits, `perform` returns and Rage restarts the daemon with backoff.

```ruby
class RabbitMQListener < Rage::Daemon
  def perform
    system("bundle exec rake rabbitmq:listen")
  end
end
```

## Adaptive Recurring Work

Use a daemon instead of a recurring deferred task when the interval needs to adapt to workload or external state:

```ruby
class MetricsExporter < Rage::Daemon
  def perform
    loop do
      metrics = MetricsCollector.claim
      export_metrics(metrics) unless metrics.empty?

      sleep(metrics.size > 1000 ? 0.1 : 1)
    end
  end

  private

  def export_metrics(metrics)
    # ...
  end
end
```

## Choosing Daemons vs Deferred Tasks

- Use `Rage::Deferred` for discrete background jobs, retries, delayed execution, and WAL-backed task persistence.
- Use `Rage::Deferred` recurring schedules for simple fixed-interval jobs.
- Use `Rage::Daemon` for continuous listeners, persistent connections, supervised blocking commands, and recurring loops that need adaptive sleep intervals.

## References

- [Rage::Daemon API](https://api.rage-rb.dev/Rage/Daemon.html)
