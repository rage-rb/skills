# Rage::Cable (WebSockets)

Action Cable-compatible WebSocket implementation. Channels are subscription endpoints; streams are broadcast groups. Very efficient, runs in the same process as HTTP - no separate server needed.

What's **different** from Action Cable:
- `Rage::Cable` as the root namespace
- Explicit mounting of a Cable app is required in `config/routes.rb` or `config.ru`

## Setup

Mount the Cable application:

```ruby
# config/routes.rb
Rage.routes.draw do
  mount Rage::Cable.application, at: "/cable"
end
```

## Core Concepts

- **Connection**: Handles authentication when a client connects
- **Channel**: Manages subscriptions and message handling for a feature (e.g., ChatChannel)
- **Stream**: A named broadcast group that delivers messages to subscribers

Clients subscribe to channels; channels attach connections to streams based on business logic.

## Connection (Authentication)

The connection class handles WebSocket authentication. Located at `app/channels/rage_cable/connection.rb`:

```ruby
class RageCable::Connection < Rage::Cable::Connection
  identified_by :current_user

  def connect
    self.current_user = find_verified_user
    reject_unauthorized_connection unless current_user
  end

  private

  def find_verified_user
    # From query params
    if (token = params[:token])
      User.find_by(auth_token: token)
    # From cookies
    elsif (user_id = cookies.encrypted[:user_id])
      User.find_by(id: user_id)
    # From session
    elsif (user_id = session[:user_id])
      User.find_by(id: user_id)
    end
  end
end
```

**IMPORTANT:** connections are accepted unless `reject_unauthorized_connection` is called.

### Available in Connection

- `params` - Query parameters from WebSocket URL
- `cookies` - HTTP cookies (including encrypted)
- `session` - Session data
- `request` - The original HTTP request
- `reject_unauthorized_connection` - Deny the connection

## Channels

Channels handle subscriptions and messages for specific features:

```ruby
# app/channels/chat_channel.rb
class ChatChannel < Rage::Cable::Channel
  def subscribed
    room = Room.find(params[:room_id])

    # Reject if user can't access this room
    unless current_user.member_of?(room)
      reject
      return
    end

    # Attach to the stream for this room
    stream_for room
  end

  def unsubscribed
    # Cleanup when client unsubscribes
    Rage.logger.info "User left chat", user_id: current_user.id
  end

  def receive(data)
    # Handle incoming messages from client
    room = Room.find(params[:room_id])
    message = room.messages.create!(
      user: current_user,
      body: data["message"]
    )

    # Broadcast to all subscribers
    self.class.broadcast_to(room, {
      id: message.id,
      user: current_user.name,
      body: message.body,
      created_at: message.created_at
    })
  end
end
```

**Two variants of streams (in order of preference):**
1. Channel-specific streams:
  - Subscribe via `stream_for`, e.g. `stream_for(current_user)`
  - Broadcast via `Rage::Cable::Channel.broadcast_to`, e.g. `ChatChannel.broadcast_to(current_user, message)`
2. Global channel-independent streams:
  - Subscribe via `stream_from`, e.g. `stream_from("chat_room_#{params[:id]}")`
  - Broadcast via `Rage::Cable.broadcast`, e.g. `Rage::Cable.broadcast("chat_room_42", message)`

### RPC-Style Actions

Define public methods that clients can call directly:

```ruby
class ChatChannel < Rage::Cable::Channel
  def subscribed
    stream_from "chat_room_#{params[:room_id]}"
  end

  # Client calls: channel.perform("typing", {})
  def typing(data)
    Rage::Cable.broadcast("chat_room_#{params[:room_id]}", {
      type: "typing",
      user: current_user.name
    })
  end

  # Client calls: channel.perform("mark_read", { message_id: 123 })
  def mark_read(data)
    Message.find(data["message_id"]).mark_read_by(current_user)
  end

  # Client has unsubscribed from the channel
  def unsubscribed
    room = Room.find(params[:room_id])
    room.members.delete(current_user)
  end
end
```

### Channel Callbacks

```ruby
class ChatChannel < Rage::Cable::Channel
  before_subscribe :verify_membership
  after_subscribe :announce_presence

  def subscribed
    stream_from "chat_room_#{params[:room_id]}"
  end

  private

  def verify_membership
    reject unless current_user.member_of?(Room.find(params[:room_id]))
  end

  def announce_presence
    Rage::Cable.broadcast("chat_room_#{params[:room_id]}", {
      type: "user_joined",
      user: current_user.name
    })
  end
end
```

## Broadcasting

### From Anywhere in Your App

```ruby
# From controllers
class MessagesController < ApplicationController
  def create
    message = Message.create!(message_params)

    Rage::Cable.broadcast("chat_room_#{message.room_id}", {
      id: message.id,
      body: message.body,
      user: current_user.name
    })

    render json: message
  end
end

# From background jobs
class NotifyUserJob
  include Rage::Deferred::Task

  def perform(user_id:, notification:)
    Rage::Cable.broadcast("user_#{user_id}_notifications", notification)
  end
end

# From event subscribers
class BroadcastOrderUpdate
  include Rage::Events::Subscriber
  subscribe_to OrderShipped

  def call(event)
    order = Order.find(event.order_id)
    Rage::Cable.broadcast("user_#{order.user_id}_orders", {
      type: "order_shipped",
      order_id: order.id,
      tracking: event.tracking_number
    })
  end
end
```

### Transmit to Current Connection Only

Use `transmit` to send messages only to the current connection (not broadcast):

```ruby
class NotificationsChannel < Rage::Cable::Channel
  def subscribed
    # Send unread count only to this connection
    transmit({
      type: "unread_count",
      count: current_user.unread_notifications.count
    })
  end
end
```

## Client Integration

### ActionCable Protocol (Default)

Use the `@rails/actioncable` JavaScript library:

```javascript
import { createConsumer } from "@rails/actioncable"

// Connect
const cable = createConsumer("ws://localhost:3000/cable?token=user_auth_token")

// Subscribe to a channel
const chatChannel = cable.subscriptions.create(
  { channel: "ChatChannel", room_id: 123 },
  {
    connected() {
      console.log("Connected to chat")
    },

    disconnected() {
      console.log("Disconnected from chat")
    },

    received(data) {
      console.log("Received:", data)
      // Handle incoming messages
      if (data.type === "typing") {
        showTypingIndicator(data.user)
      } else {
        appendMessage(data)
      }
    }
  }
)

// Send message via receive handler
chatChannel.send({ message: "Hello!" })

// Call RPC-style action
chatChannel.perform("typing", {})
chatChannel.perform("mark_read", { message_id: 456 })

// Unsubscribe
chatChannel.unsubscribe()
```

### Raw WebSocket JSON Protocol

For simpler clients without Action Cable dependency. Enable in config:

```ruby
# config/application.rb
Rage.configure do
  config.cable.protocol = :raw_websocket_json
end
```

Clients use the standard browser WebSocket API. The URL path after `/cable/` determines the channel:

```javascript
// Subscribe to ChatChannel with room_id=123
const socket = new WebSocket("ws://localhost:3000/cable/chat?room_id=123&token=auth")

socket.onopen = () => {
  console.log("Connected")
}

socket.onmessage = (event) => {
  const data = JSON.parse(event.data)
  console.log("Received:", data)
}

socket.onclose = () => {
  console.log("Disconnected")
}

// Send message
socket.send(JSON.stringify({ message: "Hello!" }))

// Close connection
socket.close()
```

Channel name mapping: URL path is converted to channel class name
- `/cable/chat` → `ChatChannel`
- `/cable/notifications` → `NotificationsChannel`
- `/cable/order_updates` → `OrderUpdatesChannel`

Also possible to use full channel name:
- `/cable/ChatChannel`
- `/cable/NotificationsChannel`

## Multi-Process and Multi-Server Setup

### Single Server, Multiple Processes

No configuration needed - Rage uses IPC (inter-process communication) automatically.

### Multiple Servers

Configure Redis Adapter in `config/cable.yml`:

```yaml
# config/cable.yml
production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL", "redis://localhost:6379") %>
  channel_prefix: myapp_production
```

Redis Streams ensures messages aren't lost during brief network disruptions.

## Channel Organization

Recommended file structure:

```
app/
  channels/
    rage_cable/
      connection.rb          # Authentication
    chat_channel.rb          # Chat feature
    notifications_channel.rb # User notifications
    presence_channel.rb      # Online status
```

## Error Handling

```ruby
class ChatChannel < Rage::Cable::Channel
  rescue_from ActiveRecord::RecordNotFound do |error|
    Rage.logger.error "Room not found"
    reject
  end

  rescue_from UnauthorizedError do |error|
    Rage.logger.error "Access denied"
    reject
  end

  def subscribed
    room = Room.find(params[:room_id])
    raise UnauthorizedError unless current_user.can_access?(room)
    stream_from "chat_room_#{room.id}"
  end
end
```

## Testing WebSocket Connections

```ruby
# spec/channels/rage_cable/connection_spec.rb
require "spec_helper"

RSpec.describe RageCable::Connection, type: :channel do
  let(:user) { User.create!(name: "Alice") }

  it "authenticates user by param token" do
    # send connection request
    connect "/cable?auth_token=123", headers: { "X-Caller-Id" => "abcde" }

    # verify connection
    expect(connection).not_to be_rejected
    expect(connection.current_user).to eq(user)
  end

  it "authenticates user by session token" do
    # send connection request
    session[:user_id] = user.id
    connect "/cable", headers: { "X-Caller-Id" => "abcde" }

    expect(connection).not_to be_rejected
    expect(connection.current_user).to eq(user)
  end

  it "rejects connections with no token" do
    connect "/cable"
    expect(connection).to be_rejected
    expect(connection.current_user).to be_nil
  end
end
```

## Testing WebSocket Channels

```ruby
# spec/channels/chat_channel_spec.rb
require "spec_helper"

RSpec.describe ChatChannel, type: :channel do
  let(:user) { User.create!(name: "Alice") }
  let(:room) { Room.create!(name: "General") }

  before do
    room.members << user
  end

  it "subscribes to room stream" do
    # emulate successful connection
    stub_connection(current_user: user)

    # emulate successful subscription
    subscribe(room_id: room.id)

    # verify subscription
    expect(subscription).to be_confirmed

    # verify stream presence
    expect(subscription).to have_stream_from("chat_room_42")
  end

  it "broadcasts messages" do
    stub_connection(current_user: user)
    subscribe(room_id: room.id)

    # verify broadcast
    expect(Rage::Cable).to receive(:broadcast).with("chat_room_42", {
      type: "user_joined",
      user: current_user.name
    })

    # call channel action
    perform :announce_presence
  end
end
```

## References

- [WebSockets documentation](https://rage-rb.dev/docs/websockets.md)
- [Rage::Cable::Connection API](https://api.rage-rb.dev/Rage/Cable/Connection)
- [Rage::Cable::Channel API](https://api.rage-rb.dev/Rage/Cable/Channel)
