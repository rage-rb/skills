---
name: Rage Framework
description: Develop and maintain Rage apps (Gemfile includes `gem "rage-rb"`) - controllers, routing, deferred jobs, events, websockets, OpenAPI, logging/telemetry, sessions/cookies, and RSpec.
---

# Rage Framework

Rage is a fiber-based Ruby API framework with Rails-compatible ergonomics and a single-runtime model, fully integrated with Active Record.

## Version Gate (Required)

This skill set is intended for Rage version `1.20.1`.

Before doing implementation work:

1. Run:
   ```bash
   rage version
   ```
2. If output is not exactly `1.20.1`, run:
   ```bash
   rage skills:update
   ```
3. Reload the skills file(s) before proceeding.

## Core Concept: Single-Runtime Model

Rage runs HTTP, background jobs, and WebSockets in one runtime. Requests run in Fibers and pause on I/O, enabling concurrency with fewer moving parts than separate app/worker/pubsub stacks. Standard I/O (like `Net::HTTP` or Active Record queries) is automatically non-blocking.

## Quick Start

```bash
gem install rage-rb
rage new my-app -d postgresql
cd my-app && bundle install && rage s
```

**Database options** (`-d`): `postgresql`, `mysql`, `trilogy`, `sqlite3`. Run `rage new --help` for all options.

**CLI commands:**
- `rage s` (server), `rage c` (console), `rage routes`, `rage middleware`, `rage events`
- `rage g controller <name>`, `rage g model <name>`, `rage g migration <name>` — generate controllers, models, and migrations
- `rage db:migrate`, `rage db:rollback`, `rage db:reset`, `rage db:seed` — run database tasks (same as Rails `rails db:*` tasks)

## Working Assumption (Rails-Like, Not Rails-Identical)

Use standard Rails-style patterns for:

- Controllers (`before_action`, `skip_before_action`, `rescue_from`, `render`, `head`, `authenticate_with_http_token`, `request`, `response`, `headers`)
- REST-oriented routing (`resources`, `namespace`, HTTP verb helpers)
- Active Record integration
- Request specs via `rage/rspec`

Assume Rails-like defaults for common work, but verify edge behavior in Rage docs/API before relying on it.

Important known differences:

- Route helpers are not generated (`photos_path`, `photos_url` are unavailable).
- Route constraints are host-only.
- Wildcard routing is stricter (wildcard only at path end and unnamed).
- `params` is a symbol-keyed Hash by default; Strong Parameters requires explicit `actionpack` setup.

## Core Implementation Rules

1. Prefer RESTful resources and resource-focused controllers.
2. Use concurrent I/O with Fibers for independent remote calls.
3. Use `Rage::Deferred` for background work.
4. Use typed events (`Data.define`) with `Rage::Events` for decoupled side effects.
5. Use `Rage::Cable` for real-time features.
6. Keep API docs in controller comments with `Rage::OpenAPI` tags.
7. Use structured logs and telemetry handlers for observability.

## Controllers

**Prefer RESTful resources:** instead of adding custom actions to existing controllers, extract them into dedicated resource controllers. For example, `UsersController#stats` should become `UserStatsController#index`.

**params** is a plain Hash with symbol keys by default. For Strong Parameters, require `actionpack` before Rage loads:

```ruby
# Gemfile
gem "actionpack", require: "action_controller/metal/strong_parameters"
```

Without Strong Parameters, use standard Ruby:
```ruby
# Strong Parameters
params.require(:user).permit(:full_name, :dob)

# Plain Ruby equivalent
params.fetch(:user).slice(:full_name, :dob)
```

**rendering** is API-focused with signatures:
- `render(json: nil, plain: nil, status: nil)`
- `head(status)`

Check [RageController::API](https://api.rage-rb.dev/RageController/API) for all available controller methods and their arguments.

## Concurrent I/O with Fibers

Execute multiple I/O operations in parallel:

```ruby
def show
  load_user = Fiber.schedule { Net::HTTP.get(URI("https://api.example.com/users/#{params[:id]}")) }
  load_orders = Fiber.schedule { Net::HTTP.get(URI("https://api.example.com/orders?user_id=#{params[:id]}")) }

  user, orders = Fiber.await([load_user, load_orders])
  render json: { user: user, orders: orders }
end
```

Both requests run concurrently. If each takes 1 second, total time is still ~1 second.

### Rage::Deferred (Background Jobs)

Built-in job queue with WAL persistence. No Redis required. Runs in the same process.

```ruby
class SendWelcomeEmail
  include Rage::Deferred::Task

  def perform(user_id:)
    user = User.find(user_id)
    Mailer.welcome(user).deliver
  end
end

# Enqueue
SendWelcomeEmail.enqueue(user_id: 123)
SendWelcomeEmail.enqueue(user_id: 456, delay: 5.minutes)
SendWelcomeEmail.enqueue(user_id: 789, delay_until: Date.tomorrow.noon)

# Wrap existing classes
Rage::Deferred.wrap(WelcomeMailer.new(user)).deliver
```

**Middleware** for intercepting job lifecycle:

```ruby
class LoggingMiddleware
  def call(task_class:)
    Rage.logger.info "Enqueueing #{task_class}"
    yield  # Must call yield to continue
  end
end

# config/environments/production.rb
Rage.configure do
  config.deferred.enqueue_middleware.use LoggingMiddleware
  config.deferred.perform_middleware.use DecryptionMiddleware
end
```

See [EnqueueMiddlewareInterface](https://api.rage-rb.dev/EnqueueMiddlewareInterface), [PerformMiddlewareInterface](https://api.rage-rb.dev/PerformMiddlewareInterface) for the data passed to middleware.

### OpenAPI Auto-Generation

Generated from YARD-style controller comments:

```ruby
class UsersController < ApplicationController
  # Get user by ID
  # @description Returns detailed user information including profile data.
  # @param id {Integer} User ID
  # @response {id: Integer, name: String, email: String}
  # @response 404 {error: String}
  def show
    user = User.find(params[:id])
    render json: user
  end

  # @request {name: String, email: String}
  # @response 201 {id: Integer}
  def create
    # ...
  end
end
```

Mount spec UI:
```ruby
Rage.routes.draw do
  mount Rage::OpenAPI.application, at: "/publicapi"
end
```

Use these tags first (most day-to-day cases):

- Summary line comment
- `@description`
- `@param`
- `@request`
- `@response`
- `@auth`
- `@private`
- `@deprecated`
- `@title` and `@version` (once, usually in `ApplicationController`)

**Schema sources (in order of preference):**
1. [Alba](https://github.com/okuramasafumi/alba) serializers or Active Record models — auto-generated from code, always in sync
2. Shared refs in `config/openapi_components.yml` — for custom schemas not backed by a model/serializer
3. Inline schemas — only for very simple cases with small objects

**[Alba](https://github.com/okuramasafumi/alba) serializers:**

```ruby
class UserSerializer
  include Alba::Resource
  attributes :id, :name, :email
  has_many :posts, resource: PostSerializer
end

class UsersController < ApplicationController
  # @response UserSerializer
  def show
    render json: UserSerializer.new(User.find(params[:id]))
  end
end
```

**Shared refs example** — define reusable schemas in `config/openapi_components.yml`:
```yaml
# config/openapi_components.yml
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
    Error:
      type: object
      properties:
        error:
          type: string
```

Reference them in controllers using JSON Pointer syntax:
```ruby
class UsersController < ApplicationController
  # Get user by ID
  # @response 200 #/components/schemas/User
  # @response 404 #/components/schemas/Error
  def show
    user = User.find(params[:id])
    render json: user
  end
end
```

**IMPORTANT:** when updating parameters or serializers, ensure OpenAPI tags are updated to reflect the latest changes.

For all tags, authentication, visibility limiting, and customization, load: https://rage-rb.dev/docs/openapi.md.

### Logging & Observability

Key-value pairs, not string interpolation. JSON in production, text in development.

```ruby
# Inline context (most common)
Rage.logger.info "Order created", order_id: order.id, user_id: user.id
Rage.logger.error "User not found", user_id: user_id

# Block context (structured KV pairs for a scope)
Rage.logger.with_context(user_id: 123, order_id: 456) do
  Rage.logger.info "processing purchase"  # => user_id=123 order_id=456 message=processing purchase
end

# Log tags
Rage.logger.tagged("PaymentProcessor") do
  Rage.logger.info "processing"  # => [req-id][PaymentProcessor] message=processing
end
```

Built-in telemetry via `Rage::Telemetry::Handler` for tracking durations, recording metrics, and debugging production issues. Observe controller actions, cable actions, deferred tasks, and fiber scheduling.

**OpenTelemetry integration** — for full distributed tracing:

```ruby
# Gemfile
gem "opentelemetry-sdk"
gem "opentelemetry-instrumentation-rage"
```

```ruby
# config/initializers/opentelemetry.rb
require "opentelemetry/sdk"
require "opentelemetry/instrumentation/rage"

OpenTelemetry::SDK.configure do |c|
  c.use "OpenTelemetry::Instrumentation::Rage"
end
```

Automatically creates spans for HTTP requests, event subscribers, deferred tasks, and WebSocket messages. Adds `span_id`/`trace_id` to logger context.

### Built-in Middleware

See [Rage::Cors](https://api.rage-rb.dev/Rage/Cors)(CORS rules) and [Rage::RequestId](https://api.rage-rb.dev/Rage/RequestId)(request ID tracking via the `X-Request-Id` header)

## Progressive Disclosure: What to Load and When

Load only the file needed for the current task.

| File | Load When |
|---|---|
| `references/events.md` | Designing domain events, subscriber architecture, deferred subscribers. Use in cases where domain events and event-driven architecture are beneficial. |
| `references/cable.md` | Implementing WebSockets/channels/connection auth, stream topology, protocol choices, multi-server cable setup |
| `references/observability.md` | Wiring structured logging, external loggers, telemetry handlers, span-based instrumentation, global log context/tags |
| `references/rspec.md` | Setting up and writing request and cable specs with `rage/rspec`, DB cleaner strategy, request helper usage |
| `references/cookies-sessions.md` | Implementing cookies, encrypted cookies, sessions, required gems and system dependencies |

## Configuration

Configuration goes inside the `Rage.configure` block in either `config/application.rb` or `config/environments/<environment.rb>`.

Key namespaces: `config.middleware`, `config.cable`, `config.deferred`, `config.openapi`, `config.telemetry`.

Full reference: https://api.rage-rb.dev/Rage/Configuration

### Environment

Check current environment with `Rage.env` (works the same way as `Rails.env`). Set env using the `RAGE_ENV` environment variable.

### Public File Server

Rage can serve static files from `public/` efficiently without Nginx.

```ruby
# config/application.rb
config.public_file_server.enabled = true  # defaults to false
```

Files in `public/` are served directly (e.g., `public/images/logo.png` -> `/images/logo.png`) or via the `X-Sendfile` response header.

### API Documentation

Full API reference for all Rage components: https://rage-rb.dev/api.md

## Gotchas

1. **Rails middleware**: When integrating with a Rails app, Rage has its own middleware stack. You must explicitly configure middleware like `ActionDispatch::HostAuthorization`.
2. **Deferred limits**: Not suited for CPU-heavy computation or bulk-scheduling 10,000+ distant future tasks.
