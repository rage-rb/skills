# Rage Rendering and Custom Renderers

Rage is API-first, but controllers can render HTML and other formats. Set the correct `content-type` header for non-JSON responses.

## One-Off HTML Rendering

Manual rendering works well for isolated pages or small integrations:

```ruby
def index
  template = ERB.new(Rage.root.join("app/views/index.html.erb").read)
  render plain: template.result
  headers["content-type"] = "text/html"
end
```

You can also set the content type once at the controller level:

```ruby
after_action { headers["content-type"] = "text/html" }

def index
  render plain: MyPhlexComponent.new.call
end
```

## Custom Renderers

Use `config.renderer` when the app renders templates or components in multiple places.

Define renderers in Rage configuration:

```ruby
# config/application.rb
Rage.configure do
  config.renderer(:phlex) do |component, **props|
    headers["content-type"] = "text/html"
    component.new(**props).call
  end
end
```

The renderer receives:

- The first argument passed to `render`
- Any additional keyword arguments

If the block does not perform its own response handling, its return value becomes the response body.

### Controller Context

The renderer block runs in controller context, so it can use controller state and helpers such as:

- `params`
- `headers`
- `cookies`
- `session`
- other controller methods

This also means the block can handle the response itself, for example by calling `head` or delegating to another render path like `render sse:`.

### Using Custom Renderers

Once configured, render with the renderer name:

```ruby
class GreetingsController < RageController::API
  def show
    render phlex: GreetingComponent, name: params[:name]
  end
end
```

This calls the `:phlex` renderer with `GreetingComponent` as the first argument and `name:` as a keyword argument.

### Phlex Example

```ruby
# config/application.rb
Rage.configure do
  config.renderer(:phlex) do |component, **props|
    headers["content-type"] = "text/html"
    component.new(**props).call
  end
end
```

```ruby
render phlex: GreetingComponent, name: params[:name]
```

### ERB with Caching and Reloading

```ruby
Rage.configure do
  templates = {}

  config.after_reload do
    templates.clear
  end

  config.renderer(:erb) do |path|
    headers["content-type"] = "text/html"
    templates[path] ||= ERB.new(
      Rage.root.join("app", "views", "#{path}.html.erb").read
    )

    templates[path].result(binding)
  end
end
```

The cache avoids reparsing templates and is cleared automatically during development reloads.

## Inertia.js

Use the official [Inertia.js adapter for Rage](https://github.com/rage-rb/inertia-rage) to render Inertia responses:

```ruby
class PostsController < RageController::Inertia
  def index
    render inertia: "Posts/Index", props: { posts: current_user.posts }
  end
end
```

## Form-Oriented Apps

Rage resource routing is API-first. `resources` and `resource` do not expose Rails-style form actions like `new` and `edit` unless you enable them explicitly.

Use `form_actions` for template-oriented or full-stack apps that render forms:

```ruby
Rage.configure do
  config.router.form_actions = true
end
```

With this enabled, `resources` and `resource` include the extra form-oriented actions that Rails apps typically expect.

## Practical Guidance

- Use manual HTML rendering for one-off pages or small integrations.
- Use `config.renderer` for reusable template systems like ERB, Phlex, or another view library.
- Use `inertia-rage` for Inertia.js applications.
- Enable `config.router.form_actions = true` when building HTML CRUD flows with `new` and `edit` pages.
- Do not assume Rails route helpers exist; Rage still does not generate `*_path` and `*_url` helpers.
