# Testing Rage Applications with RSpec

Rage provides integrated RSpec support for API endpoint testing with Rails-style request spec helpers.

## Setup

### 1. Add Dependencies

```ruby
# Gemfile
group :test do
  gem "rspec"
  gem "database_cleaner-active_record"
end
```

```bash
bundle install
```

### 2. Initialize RSpec

```bash
rspec --init
```

### 3. Configure spec_helper.rb

```ruby
# spec/spec_helper.rb
require "rage/rspec" # => IMPORTANT
require "database_cleaner/active_record"

RSpec.configure do |config|
  # Database cleaning - MUST use truncation, not transaction
  config.before(:suite) do
    DatabaseCleaner.strategy = :truncation
    DatabaseCleaner.clean_with(:truncation)
  end

  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end

  # Standard RSpec config
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  config.shared_context_metadata_behavior = :apply_to_host_groups
end
```

## IMPORTANT: Transaction Strategy Not Supported

Rage's Fiber architecture uses separate database connections per fiber. This means:

- **Don't use** `DatabaseCleaner.strategy = :transaction`
- **Always use** `DatabaseCleaner.strategy = :truncation`

If you use transactions, test data created in `before` blocks won't be visible to your controllers.

## Writing Request Specs

### Basic Request Spec

```ruby
# spec/requests/users_spec.rb
require "spec_helper"

RSpec.describe "Users API", type: :request do
  describe "GET /users/:id" do
    it "returns user data" do
      user = User.create!(name: "Alice", email: "alice@example.com")

      get "/users/#{user.id}"

      expect(response).to have_http_status(:ok)
      expect(response.parsed_body).to include(
        "id" => user.id,
        "name" => "Alice",
        "email" => "alice@example.com"
      )
    end

    it "returns 404 for non-existent user" do
      get "/users/999999"

      expect(response).to have_http_status(:not_found)
    end
  end
end
```

### POST Requests with JSON

```ruby
RSpec.describe "Users API", type: :request do
  describe "POST /users" do
    it "creates a new user" do
      post "/users",
        params: { name: "Bob", email: "bob@example.com" },
        as: :json

      expect(response).to have_http_status(:created)
      expect(response.parsed_body).to include("id" => be_present)
      expect(User.find_by(email: "bob@example.com")).to be_present
    end

    it "returns validation errors" do
      post "/users",
        params: { name: "", email: "invalid" },
        as: :json

      expect(response).to have_http_status(:unprocessable_entity)
      expect(response.parsed_body["errors"]).to be_present
    end
  end
end
```

### Custom Headers

```ruby
RSpec.describe "Protected API", type: :request do
  let(:user) { User.create!(name: "Alice", auth_token: "secret123") }

  it "authenticates with token" do
    get "/profile",
      headers: { "Authorization" => "Bearer #{user.auth_token}" }

    expect(response).to have_http_status(:ok)
  end

  it "rejects missing token" do
    get "/profile"

    expect(response).to have_http_status(:unauthorized)
  end
end
```

### Testing with Query Parameters

```ruby
RSpec.describe "Search API", type: :request do
  before do
    User.create!(name: "Alice", role: "admin")
    User.create!(name: "Bob", role: "user")
    User.create!(name: "Charlie", role: "user")
  end

  it "filters by role" do
    get "/users", params: { role: "admin" }

    expect(response).to have_http_status(:ok)
    expect(response.parsed_body.size).to eq(1)
    expect(response.parsed_body.first["name"]).to eq("Alice")
  end

  it "supports pagination" do
    get "/users", params: { page: 1, per_page: 2 }

    expect(response.parsed_body.size).to eq(2)
  end
end
```

## Available HTTP Methods

```ruby
get(path, params: {}, headers: {})
post(path, params: {}, headers: {}, as: nil)
put(path, params: {}, headers: {}, as: nil)
patch(path, params: {}, headers: {}, as: nil)
delete(path, params: {}, headers: {}, as: nil)
head(path, params: {}, headers: {})
options(path, params: {}, headers: {})
```

**Options:**
- `params:` - Query parameters (GET) or body (POST/PUT/PATCH)
- `headers:` - HTTP headers hash
- `as: :json` - Send params as JSON body with appropriate Content-Type

## Response Helpers

```ruby
# Response object
response              # Full response object
response.status       # HTTP status code (integer)
response.body         # Raw response body (string)
response.parsed_body  # Parsed JSON (hash/array)
response.headers      # Response headers

# Status matchers
expect(response).to have_http_status(:ok)           # 200
expect(response).to have_http_status(:created)      # 201
expect(response).to have_http_status(:no_content)   # 204
expect(response).to have_http_status(:bad_request)  # 400
expect(response).to have_http_status(:unauthorized) # 401
expect(response).to have_http_status(:forbidden)    # 403
expect(response).to have_http_status(:not_found)    # 404
expect(response).to have_http_status(:unprocessable_entity) # 422
expect(response).to have_http_status(:internal_server_error) # 500

# Or use numeric codes
expect(response).to have_http_status(201)
```

## Testing Subdomain Constraints

```ruby
RSpec.describe "Multi-tenant API", type: :request do
  let(:tenant) { Tenant.create!(subdomain: "acme") }

  it "routes to correct tenant" do
    get "/dashboard",
      headers: { "Host" => "acme.example.com" }

    expect(response).to have_http_status(:ok)
  end
end
```

## Testing File Uploads

```ruby
RSpec.describe "Uploads API", type: :request do
  it "accepts file upload" do
    file = Rack::Test::UploadedFile.new(
      "spec/fixtures/sample.pdf",
      "application/pdf"
    )

    post "/documents", params: { file: file }

    expect(response).to have_http_status(:created)
  end
end
```

## Running Tests

```bash
# Run all specs
rspec

# Run specific file
rspec spec/requests/users_spec.rb

# Run specific example
rspec spec/requests/users_spec.rb:15

# With documentation format
rspec --format documentation
```

## References

- [RSpec integration docs](https://rage-rb.dev/docs/rspec.md)
- [DatabaseCleaner](https://github.com/DatabaseCleaner/database_cleaner)
