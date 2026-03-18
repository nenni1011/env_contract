# env_contract

> Schema validation for ENV variables — fail at boot, not in production.

[![Build Status](https://github.com/yourusername/env_contract/actions/workflows/ci.yml/badge.svg)](https://github.com/yourusername/env_contract/actions)
[![Gem Version](https://badge.fury.io/rb/env_contract.svg)](https://badge.fury.io/rb/env_contract)
[![Coverage](https://codecov.io/gh/yourusername/env_contract/branch/main/graph/badge.svg)](https://codecov.io/gh/yourusername/env_contract)
[![Ruby Version](https://img.shields.io/badge/ruby-%3E%3D%203.0-red)](https://www.ruby-lang.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## The Problem

Every Ruby and Rails application reads from `ENV` — but nothing enforces that the right
keys exist, with the right types, before your app starts serving requests.

**Before `env_contract` — silent failures, discovered in production:**

```ruby
# ❌ Missing key — raises NoMethodError 3 stack frames deep in production
Stripe.api_key = ENV["STRIPE_SECRET_KEY"]

# ❌ "false" is truthy in Ruby — feature activates on every environment
if ENV["ENABLE_NEW_CHECKOUT"]
  render_new_checkout
end

# ❌ nil.to_i returns 0 — database pool silently initializes at 0
pool_size = ENV["DB_POOL_SIZE"].to_i
ActiveRecord::Base.establish_connection(pool: pool_size)
```

**After `env_contract` — loud failure at boot, never in production:**

```ruby
# ✅ App refuses to boot with a clear, actionable error listing ALL problems:
EnvContract.define do
  required :STRIPE_SECRET_KEY, :string,  sensitive: true
  required :ENABLE_NEW_CHECKOUT, :boolean
  required :DB_POOL_SIZE,      :integer, min: 1, max: 100
end

# On boot with missing/invalid config:
# EnvContract::ConfigurationError:
#   3 environment variable contract violation(s):
# [MISSING]  STRIPE_SECRET_KEY
#              → Add STRIPE_SECRET_KEY=<value> to your environment
# [INVALID]  ENABLE_NEW_CHECKOUT
#              → Cannot coerce "nope" to boolean
#              → Expected: true/false, yes/no, 1/0, on/off
# [INVALID]  DB_POOL_SIZE
#              → Value 0 is below minimum 1
```

---

## Why Not Just Use `ENV.fetch`?

| Approach | What it does | What it misses |
|---|---|---|
| `ENV["KEY"]` | Returns `nil` silently | Presence, type, constraints |
| `ENV.fetch("KEY")` | Raises on missing | Type coercion, constraints, all-at-once errors |
| `dotenv` | Loads `.env` files | Validation entirely |
| `dotenv_validator` | Validates presence | Type coercion, typed accessor, zero-deps |
| `envied` | Types + required | Abandoned, has dependencies |
| **`env_contract`** | **All of the above** | **Nothing** |

---

## Installation

Add to your `Gemfile`:

```ruby
gem "env_contract"
```

Then:

```bash
bundle install
```

Or install directly:

```bash
gem install env_contract
```

**Requirements:** Ruby >= 3.0 | Zero runtime dependencies

---

## Quick Start

```ruby
# config/initializers/env_contract.rb (Rails)
# OR anywhere before your app boots (plain Ruby)

EnvContract.define do
  required :DATABASE_URL, :string
  required :REDIS_URL,     :string
  optional :MAX_THREADS,   :integer, default: 5
end

# In Rails — validation runs automatically at boot via Railtie.
# In plain Ruby — call validate! explicitly:
EnvContract.validate!

# Access typed, validated config anywhere:
config = EnvContract.config
config.database_url  # => "postgres://..." (String)
config.max_threads   # => 5               (Integer, not "5")
```

That's it. If any required var is missing or any value fails its constraint,
the app **refuses to start** with a clear, complete error message.

---

## Full API Reference

### `EnvContract.define`

The entry point. Call once, before boot. Accepts a block with field declarations.

```ruby
EnvContract.define do
  # field declarations here
end
```

> **Rails:** Place in `config/initializers/env_contract.rb`. The Railtie calls
> `validate!` automatically during `before_initialize`.
>
> **Plain Ruby / Sinatra / Rake:** Call `EnvContract.validate!` explicitly after
> `define`.

---

### Required Fields

Declares a field that MUST be present and non-empty in `ENV`.
Raises `ConfigurationError` at boot if missing.

```ruby
EnvContract.define do
  required :DATABASE_URL, :string
  required :PORT,         :integer
  required :RATE_LIMIT,   :float
  required :DEBUG_MODE,   :boolean
  required :APP_ENV,      :symbol
  required :WEBHOOK_URL,  :uri
end
```

**Signature:**
```ruby
required(name, type = :string, **options)
```

| Parameter | Type | Description |
|---|---|---|
| `name` | Symbol / String | ENV key name. Automatically uppercased. |
| `type` | Symbol | One of `:string`, `:integer`, `:float`, `:boolean`, `:symbol`, `:uri` |
| `**options` | Hash | Constraints and metadata (see below) |

---

### Optional Fields with Defaults

Declares a field that is not required. If absent, the `default` is used.

```ruby
EnvContract.define do
  optional :LOG_LEVEL,    :string,  default: "info"
  optional :MAX_THREADS,  :integer, default: 5
  optional :VERBOSE,      :boolean, default: false
  optional :TIMEOUT,      :float,   default: 30.0
end
```

If no `default` is provided and the ENV var is absent, the config accessor
returns `nil` for that field.

---

### Supported Types

| Type | ENV Input | Ruby Output | Notes |
|---|---|---|---|
| `:string` | `"hello"` | `"hello"` | Strips surrounding whitespace |
| `:integer` | `"42"` | `42` | Strips whitespace; `nil` on non-numeric |
| `:float` | `"3.14"` | `3.14` | Strips whitespace; `nil` on non-numeric |
| `:boolean` | `"true"`, `"1"`, `"yes"`, `"on"` | `true` | Case-insensitive |
| `:boolean` | `"false"`, `"0"`, `"no"`, `"off"` | `false` | Case-insensitive |
| `:symbol` | `"production"` | `:production` | Calls `.to_sym` |
| `:uri` | `"https://..."` | `"https://..."` | Validates HTTP/HTTPS URI |

**Boolean variants accepted (case-insensitive):**

```
true  → "true", "True", "TRUE", "1", "yes", "YES", "on",  "ON"
false → "false","False","FALSE","0", "no",  "NO",  "off", "OFF"
```

Anything outside this list (e.g., `"nope"`, `"enabled"`) is an `[INVALID]` violation.

---

### Value Constraints

Constraints are validated after type coercion.

#### `min:` and `max:` — Numeric bounds

```ruby
required :PORT,         :integer, min: 1024, max: 65535
required :TIMEOUT,      :float,   min: 0.1,  max: 60.0
optional :MAX_THREADS,  :integer, default: 5, min: 1, max: 32
```

#### `included_in:` — Allowlist

```ruby
required :APP_ENV,   :string, included_in: %w[development staging production]
required :LOG_LEVEL, :string, included_in: %w[debug info warn error fatal],
                              default: "info"
```

#### `format:` — Regex pattern

```ruby
required :API_KEY,     :string, format: /\A[A-Za-z0-9_\-]{32,}\z/
required :ADMIN_EMAIL, :string, format: /\A[^@\s]+@[^@\s]+\z/
```

> ⚠️ **Security note:** Developer-supplied regexes are not checked for catastrophic
> backtracking (ReDoS). Test your regex patterns independently.

#### `sensitive: true` — Redact value from error output

```ruby
required :SECRET_KEY_BASE, :string, sensitive: true
required :STRIPE_API_KEY,  :string, sensitive: true

# Error output (value NEVER exposed):
# [INVALID] SECRET_KEY_BASE → [REDACTED] does not match required format
```

#### `desc:` — Self-documenting annotation

```ruby
required :DATABASE_URL, :string,
         desc: "PostgreSQL primary connection string"

# Error output:
# [MISSING] DATABASE_URL
#           → PostgreSQL primary connection string
#           → Add DATABASE_URL=<value> to your environment
```

#### `skip_in:` — Skip validation in specific environments

```ruby
required :STRIPE_SECRET_KEY, :string, skip_in: [:test]
required :SENTRY_DSN,        :string, skip_in: %i[test development]
```

Uses `RAILS_ENV` or `RACK_ENV` to detect the current environment.

---

### Group Namespacing

Organise related fields into logical groups for readability and namespaced accessors.

```ruby
EnvContract.define do
  group :database do
    required :DATABASE_URL,  :string,  desc: "PostgreSQL primary"
    optional :DB_POOL_SIZE,  :integer, default: 5, min: 1, max: 100
    optional :DB_TIMEOUT,    :integer, default: 5000
  end

  group :redis do
    required :REDIS_URL,     :string, desc: "Sidekiq + ActionCable"
    optional :REDIS_TIMEOUT, :integer, default: 1
  end

  group :stripe do
    required :STRIPE_SECRET_KEY,      :string, sensitive: true
    required :STRIPE_PUBLISHABLE_KEY, :string
    optional :STRIPE_WEBHOOK_SECRET,  :string, sensitive: true,
             skip_in: %i[development test]
  end
end

# Namespaced accessor:
EnvContract.config.database.db_pool_size  # => 5
EnvContract.config.redis.redis_url        # => "redis://..."
EnvContract.config.stripe.stripe_secret_key  # => [runtime value]
```

---

### `EnvContract.validate!`

Validates all declared fields against the current `ENV`.

```ruby
# Returns a Config object on success:
config = EnvContract.validate!

# Raises ConfigurationError on any violation:
# EnvContract::ConfigurationError:
#   2 environment variable contract violation(s):
#   ...
```

In Rails, this is called automatically by the Railtie. You should only call it
manually in plain Ruby, Rake tasks, or Sinatra apps.

---

### `EnvContract.config`

Returns the validated, type-coerced, frozen `Config` object.

```ruby
config = EnvContract.config

# Accessor methods (snake_cased from ENV key name):
config.database_url   # => "postgres://..."  (String)
config.max_threads    # => 5                 (Integer)
config.log_level      # => "info"            (String)
config.debug_mode     # => false             (Boolean)

# Predicate methods for boolean fields:
config.debug_mode?    # => false
config.verbose?       # => true

# Hash representation:
config.to_h
# => { "DATABASE_URL" => "postgres://...", "MAX_THREADS" => 5, ... }

# Raw key access (uppercase string):
config["DATABASE_URL"]  # => "postgres://..."
```

---

### A Full, Real-World Example

```ruby
# config/initializers/env_contract.rb

EnvContract.define do
  # ── Core Infrastructure ────────────────────────────────────
  group :app do
    required :APP_ENV,        :string,  included_in: %w[development staging production],
                                        desc: "Deployment environment"
    required :SECRET_KEY_BASE,:string,  sensitive: true, format: /\A.{64,}\z/,
                                        desc: "Rails secret key base (min 64 chars)"
    optional :PORT,           :integer, default: 3000, min: 1024, max: 65535
  end

  group :database do
    required :DATABASE_URL,   :string,  desc: "PostgreSQL primary connection URL"
    optional :DB_POOL_SIZE,   :integer, default: 5, min: 1, max: 100,
                                        desc: "ActiveRecord connection pool size"
  end

  group :redis do
    required :REDIS_URL,      :string,  desc: "Redis for Sidekiq + ActionCable"
  end

  # ── Third-Party Integrations ───────────────────────────────
  group :stripe do
    required :STRIPE_SECRET_KEY,      :string, sensitive: true,
                                               skip_in: %i[test]
    required :STRIPE_PUBLISHABLE_KEY, :string, skip_in: %i[test]
    optional :STRIPE_WEBHOOK_SECRET,  :string, sensitive: true,
                                               skip_in: %i[development test]
  end

  group :monitoring do
    optional :SENTRY_DSN,     :string,  skip_in: %i[development test],
                                        desc: "Sentry DSN for error tracking"
    optional :LOG_LEVEL,      :string,  default: "info",
                                        included_in: %w[debug info warn error fatal]
  end
end
```

---

## Rails Integration

### Automatic Boot Hook

`env_contract` ships with a Rails Railtie that calls `validate!` during
`before_initialize` — before database connections, cable setup, or any other
initializer runs.

**No extra setup is needed.** Place your `EnvContract.define` block in
`config/initializers/env_contract.rb` and the Railtie handles the rest.

```ruby
# config/initializers/env_contract.rb
EnvContract.define do
  required :DATABASE_URL, :string
  required :REDIS_URL,     :string
end
# ↑ That's all. Railtie calls validate! automatically.
```

### Boot Failure Output

When validation fails, `env_contract` exits the boot process with a formatted
message on `STDERR` before Rails reaches any connection setup:

```
╔══════════════════════════════════════════╗
║   env_contract — Boot Validation Failed  ║
╚══════════════════════════════════════════╝

2 environment variable contract violation(s):

  [MISSING]  DATABASE_URL
             → PostgreSQL primary connection URL
             → Add DATABASE_URL=<value> to your environment

  [INVALID]  LOG_LEVEL
             → Got "verbose", must be one of: debug, info, warn, error, fatal

Application cannot start. Fix the above violations.
```

### Boot Order

The Railtie runs at the earliest possible point in the Rails lifecycle:

```
config/application.rb loads
         ↓
env_contract Railtie fires (before_initialize)
         ↓
database.yml parsed / connections established
         ↓
other initializers run
         ↓
app boots ✅
```

This means a misconfigured `DATABASE_URL` is caught **before**
`ActiveRecord` tries to connect — not mid-request.

---

## Error Reference

### `EnvContract::ConfigurationError`

**When:** `validate!` is called and one or more fields are missing or invalid.

**Contains:** `violations` array (all errors, not just the first).

```ruby
rescue EnvContract::ConfigurationError => e
  e.violations
  # => [
  #   "[MISSING]  DATABASE_URL\n → ...",
  #   "[INVALID]  MAX_THREADS\n → ..."
  # ]
  e.message   # Full formatted string
end
```

---

### `EnvContract::NotDefinedError`

**When:** `config` or `validate!` is called before `define` has been called.

```
EnvContract::NotDefinedError:
  Call EnvContract.define { } before calling validate! or config.
```

**Fix:** Ensure your `EnvContract.define` block loads before `validate!` is called.

---

### `EnvContract::NotValidatedError`

**When:** `config` is called before `validate!` has been called.

```
EnvContract::NotValidatedError:
  Call EnvContract.validate! before accessing config.
```

---

### `EnvContract::FrozenContractError`

**When:** `define` is called again after `validate!` has already run.

```
EnvContract::FrozenContractError:
  Contract is frozen after validation. Do not redefine after boot.
```

**Fix:** Call `define` once, before `validate!`. In tests, call `EnvContract.reset!`
between examples.

---

### `EnvContract::DuplicateFieldError`

**When:** The same ENV key name is declared more than once inside `define`.

```
EnvContract::DuplicateFieldError:
  DATABASE_URL is declared twice in EnvContract.define.
```

---

### `EnvContract::InvalidDefaultError`

**When:** A `default:` value cannot be coerced to the field's declared type.

```
EnvContract::InvalidDefaultError:
  Default "five" cannot be coerced to :integer for MAX_THREADS.
  Hint: Use default: 5, not default: "five"
```

---

## CLI Usage

`env_contract` ships with a CLI for validating your environment outside of the
app boot cycle — useful in CI/CD pipelines, deployment scripts, and Docker
health checks.

### `check` — Validate current ENV against your contract

```bash
bundle exec env_contract check
```

**Output (all passing):**
```
  ✓  DATABASE_URL              "postgres://localhost/myapp" (string)
  ✓  REDIS_URL                 "redis://localhost:6379" (string)
  ✓  MAX_THREADS               5 (integer)
  ✓  LOG_LEVEL                 "info" [default: "info"]
  ✓  STRIPE_SECRET_KEY         [REDACTED] (string)

All 5 contract fields satisfied.
```

**Output (violations found):**
```
  ✓  DATABASE_URL              "postgres://localhost/myapp" (string)
  ✗  REDIS_URL                 MISSING (required)
  ✓  MAX_THREADS               5 (integer)
  ✗  STRIPE_SECRET_KEY         INVALID — cannot coerce to string

2 violation(s) found.
```

Exits with **status code `1`** on any violation — ready for CI/CD pipelines.

### CI/CD Integration

```yaml
# GitHub Actions
- name: Validate ENV contract
  run: bundle exec env_contract check
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
    REDIS_URL: ${{ secrets.REDIS_URL }}
    STRIPE_SECRET_KEY: ${{ secrets.STRIPE_SECRET_KEY }}
```

```dockerfile
# Dockerfile health check
HEALTHCHECK CMD bundle exec env_contract check || exit 1
```

---

## Testing

### `EnvContract.reset!`

Clears the contract between test examples so each spec starts clean.

```ruby
# spec/spec_helper.rb
RSpec.configure do |config|
  config.before(:each) { EnvContract.reset! }
end
```

---

### `EnvContract.with_env`

Temporarily overrides ENV keys within a block. Restores all original values
after the block exits — even if the block raises an error.

```ruby
EnvContract.with_env(MAX_THREADS: "2", DEBUG: "true") do
  # ENV["MAX_THREADS"] == "2" inside this block only
  # Original values are restored after
end
```

---

### Writing Specs for Your Contract

```ruby
# spec/env_contract_spec.rb
require "spec_helper"

RSpec.describe "Application ENV contract" do
  let(:valid_env) do
    {
      "DATABASE_URL"      => "postgres://localhost/testdb",
      "REDIS_URL"         => "redis://localhost:6379",
      "MAX_THREADS"       => "5",
      "STRIPE_SECRET_KEY" => "sk_test_" + "x" * 32
    }
  end

  before do
    EnvContract.define do
      required :DATABASE_URL,      :string
      required :REDIS_URL,         :string
      optional :MAX_THREADS,       :integer, default: 5, min: 1
      required :STRIPE_SECRET_KEY, :string,  sensitive: true, skip_in: [:test]
    end
  end

  context "with all valid ENV vars" do
    it "validates successfully and returns typed config" do
      config = EnvContract.validate!(valid_env)

      expect(config.database_url).to eq("postgres://localhost/testdb")
      expect(config.max_threads).to eq(5)
      expect(config.max_threads).to be_a(Integer)
    end
  end

  context "with missing required vars" do
    it "raises ConfigurationError listing all missing keys" do
      expect {
        EnvContract.validate!({})
      }.to raise_error(EnvContract::ConfigurationError) do |error|
        expect(error.violations.size).to eq(2)
        expect(error.message).to include("DATABASE_URL")
        expect(error.message).to include("REDIS_URL")
      end
    end
  end

  context "with invalid type" do
    it "raises ConfigurationError with coercion message" do
      env = valid_env.merge("MAX_THREADS" => "many")

      expect {
        EnvContract.validate!(env)
      }.to raise_error(EnvContract::ConfigurationError, /MAX_THREADS/)
    end
  end

  context "with boolean feature flag" do
    it "correctly parses truthy string variants" do
      %w[true 1 yes on TRUE YES].each do |val|
        EnvContract.reset!
        EnvContract.define { required :FLAG, :boolean }

        EnvContract.with_env(FLAG: val) do
          config = EnvContract.validate!
          expect(config.flag).to eq(true),
            "Expected #{val.inspect} to coerce to true"
          expect(config.flag?).to eq(true)
        end
      end
    end
  end

  context "with sensitive field" do
    it "never exposes the secret value in error messages" do
      EnvContract.reset!
      EnvContract.define do
        required :API_SECRET, :string, sensitive: true, format: /\A.{64,}\z/
      end

      expect {
        EnvContract.validate!("API_SECRET" => "tooshort")
      }.to raise_error(EnvContract::ConfigurationError) do |error|
        expect(error.message).not_to include("tooshort")
        expect(error.message).to include("[REDACTED]")
      end
    end
  end
end
```

---

## Comparison with Alternatives

| Feature | `env_contract` | `dotenv` | `dotenv_validator` | `envied` | `ENVV` | `anyway_config` |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Boot-time validation | ✅ | ❌ | ✅ | ✅ | ✅ | ❌ |
| All errors at once | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Type coercion | ✅ | ❌ | ⚠️ basic | ✅ | ✅ | ✅ |
| Typed config accessor | ✅ | ❌ | ❌ | ⚠️ | ❌ | ✅ |
| Value constraints | ✅ | ❌ | ⚠️ regex | ⚠️ | ✅ | ❌ |
| Rails Railtie auto-hook | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Zero runtime dependencies | ✅ | ✅ | ❌ | ❌ | ❌ | ⚠️ |
| Schema in Ruby code | ✅ | ❌ | ❌ | ⚠️ Envfile | ✅ | ✅ |
| Sensitive field masking | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Test helper (`with_env`) | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| CLI check command | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Group namespacing | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ |
| `skip_in: [:test]` | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Actively maintained | ✅ | ✅ | ✅ | ❌ | ⚠️ | ✅ |
| Boolean variants (1/yes/on) | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Empty string = missing | ✅ | ❌ | ❌ | ⚠️ | ✅ | ❌ |

> **`env_contract` + `dotenv` is the recommended pairing.** Use `dotenv` to
> _load_ `.env` files. Use `env_contract` to _validate_ them. They complement
> each other perfectly.

---

## Contributing

Contributions, bug reports, and feature requests are welcome.

### Setup

```bash
git clone https://github.com/yourusername/env_contract.git
cd env_contract
bundle install
```

### Run Tests

```bash
bundle exec rspec                    # full test suite
bundle exec rspec spec/unit/         # unit tests only
bundle exec rspec spec/integration/  # integration tests only
```

### Run Linter

```bash
bundle exec rubocop
bundle exec rubocop -a   # auto-correct safe offenses
```

### Coverage

Test coverage is enforced at **100%** via SimpleCov. The CI build will fail
if coverage drops below 100% on any PR.

### PR Guidelines

- One concern per PR — keep it focused
- All new features must include specs
- All new public methods must include YARD documentation
- Update `CHANGELOG.md` under `[Unreleased]` for every change
- Ensure `bundle exec rspec && bundle exec rubocop` both pass locally

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full contribution guide,
including commit message format and branch naming conventions.

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for a full history of releases and changes.

---

## Security

If you discover a security vulnerability in `env_contract`, please do **not**
open a public GitHub issue. Instead, follow the responsible disclosure process
described in [SECURITY.md](SECURITY.md).

---

## License

`env_contract` is released under the [MIT License](LICENSE.txt).

Copyright © 2026 Your Name
