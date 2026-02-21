# Rage Cookies and Sessions

## Cookies

Rage provides Rails-compatible cookie handling for reading and writing HTTP cookies.

### Reading Cookies

```ruby
class SessionsController < ApplicationController
  def show
    token = cookies[:auth_token]
    user_preferences = cookies[:preferences]
    render json: { token: token, preferences: user_preferences }
  end
end
```

### Writing Cookies

```ruby
class SessionsController < ApplicationController
  def create
    user = User.authenticate(params[:email], params[:password])

    # Simple cookie
    cookies[:user_id] = user.id

    # Cookie with options
    cookies[:auth_token] = {
      value: user.generate_token,
      expires: 1.week.from_now,
      httponly: true,
      secure: Rage.env.production?
    }

    render json: { status: "logged_in" }
  end
end
```

### Encrypted Cookies

Encrypted cookies are both tamper-proof and unreadable by clients:

```ruby
# Write encrypted cookie
cookies.encrypted[:sensitive_data] = { account_id: 123, role: "admin" }

# Read encrypted cookie
data = cookies.encrypted[:sensitive_data]
# => { account_id: 123, role: "admin" }
```

### Permanent Cookies

Set cookies that expire 20 years in the future:

```ruby
cookies.permanent[:remember_me] = "yes"
cookies.permanent.signed[:user_id] = current_user.id
cookies.permanent.encrypted[:preferences] = user_preferences
```

### Deleting Cookies

```ruby
cookies.delete(:auth_token)
cookies.delete(:user_id, domain: ".example.com")
```

### Cookie Options

| Option | Description |
|--------|-------------|
| `:value` | Cookie value |
| `:path` | Path scope (default: `/`) |
| `:domain` | Domain scope |
| `:expires` | Expiration time |
| `:secure` | HTTPS only |
| `:httponly` | Not accessible via JavaScript |
| `:same_site` | SameSite attribute (`:strict`, `:lax`, `:none`) |

## Sessions

Rage sessions provide server-side storage with a session ID stored in a cookie. Sessions require the `libsodium` system library for encryption.

### System Dependency

```bash
# Ubuntu/Debian
sudo apt install libsodium23

# macOS
brew install libsodium
```

### Secret Key Base

Sessions require a secret key for encryption. Configure via environment variable (recommended) or directly:

```bash
# Generate a secure key
ruby -r securerandom -e 'puts SecureRandom.hex(64)'

# Set via environment variable (recommended)
export SECRET_KEY_BASE="your-generated-key"
```

```ruby
# Or configure directly (e.g., in config/application.rb)
Rage.configure do
  config.secret_key_base = ENV["SECRET_KEY_BASE"]
end
```

Keep this value private and out of version control.

### Required Gems

```ruby
# Gemfile
gem "base64"
gem "domain_name"
gem "rbnacl"
```

### Basic Usage

```ruby
class ApplicationController < RageController::API
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
end

class SessionsController < ApplicationController
  def create
    user = User.authenticate(params[:email], params[:password])
    if user
      session[:user_id] = user.id
      session[:logged_in_at] = Time.now.to_i
      render json: { status: "success" }
    else
      render json: { error: "Invalid credentials" }, status: :unauthorized
    end
  end

  def destroy
    session.clear
    render json: { status: "logged_out" }
  end
end
```

### Session Configuration

```ruby
# config/application.rb
Rage.configure do
  # Session cookie name
  config.session.key = "_myapp_session"
end
```

### Accessing Session Data

```ruby
# Read
user_id = session[:user_id]

# Write
session[:cart_id] = cart.id

# Check existence
session.key?(:user_id)

# Delete single key
session.delete(:temporary_data)

# Clear entire session
session.clear
```

## References

- [Rage::Cookies API](https://api.rage-rb.dev/Rage/Cookies)
- [Rage::Session API](https://api.rage-rb.dev/Rage/Session)
