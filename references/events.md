# Rage::Events (Typed Domain Events)

Object-oriented in-process events using `Data.define`, not string-based. Supports hierarchical subscriptions via Ruby inheritance and async execution via `deferred: true`.

## When to Use Events

**Good fit:**
- Behavior spans multiple concerns (e.g., order creation triggers email, analytics, inventory update)
- Side effects evolve over time and you want to add/remove reactions without modifying core logic
- You want clear domain boundaries and decoupled components

**Not ideal:**
- Single-purpose operations where a direct method call is clearer
- Core business logic that requires explicit control flow

## Defining Events

Events are immutable, typed Ruby objects. Any class can be used, but prefer using `Data.define` by default:

```ruby
# Simple event
OrderCreated = Data.define(:order_id, :user_id, :total_amount)

# Event with default values
UserRegistered = Data.define(:user_id, :source) do
  def initialize(user_id:, source: "web")
    super
  end
end
```

## Event Hierarchies

Use modules or inheritance to group related events. Subscribers can listen to parent types to catch all children:

```ruby
# Base module for all order events
module OrderEvent; end

# Define events as typed objects
class OrderCreated < Data.define(:order_id, :user_id)
  include OrderEvent
end

class OrderShipped < Data.define(:order_id, :tracking_number)
  include OrderEvent
end

class OrderCancelled < Data.define(:order_id, :reason)
  include OrderEvent
end
```

## Subscribers

Subscribers react to published events. Include `Rage::Events::Subscriber` and declare which events to handle:

```ruby
class SendOrderConfirmation
  include Rage::Events::Subscriber
  subscribe_to OrderCreated

  def call(event)
    user = User.find(event.user_id)
    Mailer.order_confirmation(user, event.order_id).deliver
  end
end
```

### Subscribing to Event Hierarchies

Subscribe to a parent module/class to catch all related events:

```ruby
class OrderAuditLog
  include Rage::Events::Subscriber
  subscribe_to OrderEvent  # Catches OrderCreated, OrderShipped, OrderCancelled, etc.

  def call(event)
    AuditLog.create!(
      event_type: event.class.name,
      data: event.to_h,
      occurred_at: Time.current
    )
  end
end
```

### Async Subscribers (Deferred)

Mark subscribers with `deferred: true` to execute asynchronously via `Rage::Deferred`:

```ruby
class UpdateAnalytics
  include Rage::Events::Subscriber
  subscribe_to OrderCreated, deferred: true

  def call(event)
    Analytics.track("order_created", order_id: event.order_id, user_id: event.user_id)
  end
end

class SyncToExternalCRM
  include Rage::Events::Subscriber
  subscribe_to UserRegistered, deferred: true

  def call(event)
    ExternalCRM.create_contact(user_id: event.user_id)
  end
end
```

**IMPORTANT:** Deferred events are serialized to WAL. If you change event fields (add/remove/rename), deserialization will break on restart. Deprecate old fields gradually or create new event classes.

### Error Handling in Subscribers

Use `rescue_from` for centralized exception handling:

```ruby
class SendOrderConfirmation
  include Rage::Events::Subscriber
  subscribe_to OrderCreated

  rescue_from ActiveRecord::RecordNotFound do |error|
    Rage.logger.error "User not found for order confirmation", error: error.message
  end

  rescue_from StandardError do |error|
    ErrorTracker.capture(error)
  end

  def call(event)
    user = User.find(event.user_id)
    Mailer.order_confirmation(user, event.order_id).deliver
  end
end
```

## Exception Isolation

Subscribers are separate components — exceptions raised in subscribers **do not propagate** back to `Rage::Events.publish`. The publish call always completes successfully. Use `rescue_from` within subscribers to handle errors (see Error Handling below).

## Publishing Events

Publish events without knowing who will react:

```ruby
# In a controller or service
def create
  order = Order.create!(order_params)

  Rage::Events.publish(OrderCreated.new(
    order_id: order.id,
    user_id: current_user.id,
    total_amount: order.total
  ))

  render json: order, status: :created
end
```

### Passing Context

Pass additional metadata unrelated to the event itself using `context`:

```ruby
Rage::Events.publish(
  OrderCreated.new(order_id: order.id, user_id: user.id),
  context: { request_id: request.id, ip_address: request.remote_ip }
)

# In subscriber
class OrderAuditLog
  include Rage::Events::Subscriber
  subscribe_to OrderCreated

  def call(event, context:)
    AuditLog.create!(
      event_type: event.class.name,
      data: event.to_h,
      request_id: context[:request_id],
      ip_address: context[:ip_address]
    )
  end
end
```

## CLI Visualization

Visualize event-subscriber relationships:

```bash
# Show all events and their subscribers
rage events

# Show subscribers for a specific event
rage events OrderCreated
```

## File Organization

Recommended structure:

```
app/
  events/
    order_event.rb
    order_created.rb
    order_shipped.rb
  subscribers/
    send_order_confirmation.rb
    update_analytics.rb
    order_audit_log.rb
```

## References

- [Events documentation](https://rage-rb.dev/docs/event-system.md)
- [Rage::Events API](https://api.rage-rb.dev/Rage/Events)
