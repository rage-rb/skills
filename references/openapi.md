# Rage::OpenAPI (Auto-Generated API Documentation)

Generate OpenAPI specs directly from controller comments. Keep tags close to actions so documentation and behavior evolve together.

## Setup

Mount the OpenAPI application to serve the spec UI:

```ruby
# config/routes.rb
Rage.routes.draw do
  mount Rage::OpenAPI.application, at: "/publicapi"
end
```

Alternative — mount in `config.ru`:

```ruby
map "/publicapi" do
  run Rage::OpenAPI.application
end
```

After starting with `rage s`, visit `http://localhost:3000/publicapi` to see the generated specification.

## Documentation Model

`Rage::OpenAPI` uses YARD-style tags in controller comments.

```ruby
class Api::V1::UsersController < ApplicationController
  # Returns a single user.
  # @description Includes profile and
  #   account metadata.
  # @param include_inactive? {Boolean} Include deactivated users
  # @response 200 UserSerializer
  # @response 404 #/components/schemas/Error
  def show
  end
end
```

Tag scope:
- Action-level tags apply to one action.
- Class-level tags apply to all actions in that controller and child controllers.

## Tag Reference

Use these tags most often:

- Summary line comment
- `@description`
- `@param`
- `@request`
- `@response`
- `@auth`
- `@private`
- `@deprecated`
- `@title` and `@version` (use once, usually in `ApplicationController`)
- `@internal` (for team notes excluded from public spec)

Global metadata example:

```ruby
class ApplicationController < RageController::API
  # @title User Management API
  # @version 1.0.0
end
```

Deprecation and private visibility can be action-level:

```ruby
class Api::V1::UsersController < ApplicationController
  def index
  end

  # @deprecated
  def destroy
  end
end
```

Or class-level:

```ruby
class Api::V1::UsersController < ApplicationController
  # @deprecated

  def destroy
  end
end
```

## Response Schemas

Document response payloads with `@response`.

Preferred schema sources:
1. [Alba](https://github.com/okuramasafumi/alba) serializers or Active Record models
2. Shared refs in `config/openapi_components.yml`
3. Inline schemas for very small payloads

### Inline Schema

```ruby
# @response { id: Integer, full_name: String, email: String }
```

### Shared References

Define reusable components:

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
          format: email
    Error:
      type: object
      properties:
        message:
          type: string
        code:
          type: string
```

Reference by JSON Pointer:

```ruby
# @response 200 #/components/schemas/User
# @response 404 #/components/schemas/Error
```

### Alba / Model Auto-Generation

```ruby
class UserSerializer
  include Alba::Resource

  root_key :user
  attributes :id, :name, :email
end

class Api::V1::UsersController < ApplicationController
  # @response UserSerializer
  def show
  end
end
```

Generated schema:

```yaml
schema:
  type: object
  properties:
    user:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        email:
          type: string
```

Collections:

```ruby
# @response Array<UserSerializer>
```

Namespace resolution is automatic. From `Api::V2::UsersController`, `UserSerializer` resolves to `Api::V2::UserSerializer` when available.

Multiple status codes:

```ruby
# @response 200 UserSerializer
# @response 404 { error: String }
```

Global responses at controller level:

```ruby
class Api::V1::BaseController < ApplicationController
  # @response 404 { status: "NOT_FOUND" }
  # @response 500 { status: "ERROR", message: String }
end
```

## Request Schemas

`@request` accepts inline schemas, shared refs, and model-backed schemas.

```ruby
class Api::V1::UsersController < ApplicationController
  # @request { name: String, email: String, password: String }
  # @response 201 UserSerializer
  def create
  end
end
```

Model-backed request schemas exclude internal fields such as `id`, `created_at`, `updated_at`, and `type`.

## Query Parameters

Use `@param` for query params.

```ruby
# @param account_id Account owning these records
# @param created_at? Filter by creation date
# @param page {Integer} Page number
# @param per_page? {Integer} Page size
# @param is_active {Boolean} Active status filter
```

Rules:
- Use `?` suffix for optional params.
- Use `{Type}` for explicit typing.

## Authentication

Document auth once in a base controller and let callback usage drive endpoint security.

```ruby
class ApplicationController < RageController::API
  before_action :authenticate_by_token
  # @auth authenticate_by_token

  private

  def authenticate_by_token
    # Authentication logic
  end
end
```

`Rage::OpenAPI` tracks callback usage and respects `skip_before_action`.

Default scheme: HTTP bearer:

```yaml
type: http
scheme: bearer
```

Custom scheme:

```ruby
# @auth authenticate_by_token
#   type: apiKey
#   in: header
#   name: X-API-Key
```

Multiple schemes:

```ruby
# @auth authenticate_by_user_token
# @auth authenticate_by_service_token
```

Custom scheme name:

```ruby
# @auth authenticate_by_token UserAuth
```

## Visibility and Namespace Filtering

Hide endpoints from public docs:

```ruby
class Api::V1::UsersController < ApplicationController
  def index
  end

  # @private
  def create
  end
end
```

Limit spec by namespace:

```ruby
map "/publicapi" do
  run Rage::OpenAPI.application(namespace: "Api::V2")
end
```

Create multiple specs for different audiences:

```ruby
map "/publicapi" do
  run Rage::OpenAPI.application(namespace: "Api::Public")
end

map "/internalapi" do
  use Rack::Auth::Basic do |user, password|
    user == "admin" && password == ENV["ADMIN_PASSWORD"]
  end

  run Rage::OpenAPI.application(namespace: "Api::Internal")
end
```

## Custom Tag Organization

Customize how endpoints are grouped using a tag resolver:

```ruby
# config/application.rb
Rage.configure do
  config.openapi.tag_resolver = proc do |controller, action, default_tag|
    tags = [default_tag]

    if controller.name.include?("Admin")
      tags << "Admin"
    elsif controller.name.include?("Public")
      tags << "Public API"
    end

    tags
  end
end
```

The resolver receives:
- `controller` — the controller class
- `action` — the action name (symbol)
- `default_tag` — the originally generated tag

Returns a string or array of strings.

## Maintenance Checklist

- Update OpenAPI tags when action behavior, params, or serializers change.
- Prefer shared refs/serializers over large inline schemas.
- Keep auth callbacks and `@auth` tags aligned.
- Use `@private` or namespace filtering before exposing internal endpoints.
- Revisit tag grouping if docs become hard to navigate.

Official docs: https://rage-rb.dev/docs/openapi.md
