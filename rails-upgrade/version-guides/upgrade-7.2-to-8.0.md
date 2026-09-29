# Rails 7.2 → 8.0 Upgrade Guide

**Ruby Requirement:** 3.2.0+ (required)

---

## Overview

Rails 8.0 is a major release with architectural changes:
- **Propshaft** replaces Sprockets as default asset pipeline
- **Solid Cache/Queue/Cable** as database-backed defaults
- **Kamal** for deployment
- **Thruster** for production HTTP serving
- **No more PaaS mode** - designed for containerized deployment

---

## Breaking Changes

### 🔴 HIGH PRIORITY

#### 1. Sprockets → Propshaft

**What Changed:**
Propshaft is the new default asset pipeline. Sprockets is no longer included by default.

**Detection Pattern:**
```ruby
# Gemfile
gem 'sprockets-rails'
gem 'sassc-rails'

# Config
config.assets.compile = true
config.assets.digest = true
```

**Migration Options:**

**Option A: Keep Sprockets (Simplest)**
```ruby
# Gemfile - explicitly keep Sprockets
gem 'sprockets-rails'
gem 'sassc-rails'  # if using Sass
```

This is fine! Sprockets still works.

**Option B: Migrate to Propshaft (Recommended for new features)**
```ruby
# Gemfile
gem 'propshaft'

# Remove
# gem 'sprockets-rails'
# gem 'sassc-rails'
```

**Propshaft Differences:**
- No asset compilation transforms
- Direct serving from `app/assets/`
- Use `cssbundling-rails` for Sass
- Simpler configuration

```ruby
# Remove these Sprockets configs
# config.assets.compile
# config.assets.digest
# config.assets.debug
```

**Asset Helpers:**
```erb
<!-- Both work the same -->
<%= stylesheet_link_tag 'application' %>
<%= javascript_include_tag 'application' %>
```

---

#### 2. Multi-Database Configuration for Solid Gems

**What Changed:**
Rails 8.0 uses Solid Cache/Queue/Cable which may need separate database connections.

**Old database.yml:**
```yaml
production:
  adapter: postgresql
  database: myapp_production
  pool: 5
```

**New database.yml (if using Solid gems):**
```yaml
production:
  primary:
    adapter: postgresql
    database: myapp_production
    pool: 5
  cache:
    adapter: sqlite3
    database: storage/production_cache.sqlite3
  queue:
    adapter: sqlite3
    database: storage/production_queue.sqlite3
  cable:
    adapter: sqlite3
    database: storage/production_cable.sqlite3
```

**If NOT using Solid gems**, keep your existing structure!

---

#### 3. assume_ssl Configuration

**What Changed:**
Rails 8.0 introduces `config.assume_ssl` for apps behind SSL-terminating proxies.

**Detection Pattern:**
```ruby
# config/environments/production.rb
config.force_ssl = true
```

**Fix:**
```ruby
# config/environments/production.rb
config.force_ssl = true
config.assume_ssl = true  # Add this for load balancers
```

This prevents SSL redirect loops when behind a proxy.

---

#### 4. sqlite3_deprecated_warning Removed

**What Changed:**
The `sqlite3_deprecated_warning` configuration option is removed.

**Detection Pattern:**
```ruby
config.active_record.sqlite3_deprecated_warning = false
```

**Fix:**
Remove this line from your configuration files.

---

#### 5. Ruby 3.2+ Strictly Required

**What Changed:**
Rails 8.0 requires Ruby 3.2.0 or newer.

**Fix:**
```bash
rbenv install 3.3.0
rbenv local 3.3.0
```

---

#### 6. `query_constraints:` association option removed (composite foreign keys)

**What Changed:**
Rails 8.0 **removes** the `query_constraints:` option on associations (`belongs_to`/`has_many`/etc.). It was deprecated in Rails 7.2 and now raises `ActiveRecord::ConfigurationError` **at class load**, so the app fails to boot.

**Detection Pattern:**
```ruby
# app/models/*.rb
belongs_to :activity, query_constraints: [:activity_id, :school_id]
```

**Error (Rails 8.0):**
```
ActiveRecord::ConfigurationError:
  Setting `query_constraints:` option on `ActivityParticipant.belongs_to :activity`
  is not allowed. To get the same behavior, use the `foreign_key` option instead.
```

**Fix — pass the composite key as an `Array` to `foreign_key:`:**
```ruby
# BEFORE (7.2 deprecation → 8.0 raises)
belongs_to :activity, query_constraints: [:activity_id, :school_id]

# AFTER (works on Rails 7.2 AND 8.0)
belongs_to :activity, foreign_key: [:activity_id, :school_id]
```

This is **behavior-preserving and version-agnostic**: when `foreign_key:` is given an `Array`, ActiveRecord internally maps it back to `query_constraints` (in both 7.2 and 8.0), so no dual-boot (`ENV["DEPENDENCIES_NEXT"]`) branch is needed. On 7.2 it also silences the deprecation warning.
  
> ⚠️ Not in the official Rails Upgrade Guide or the 7.2/8.0 release notes — documented only in `activerecord` `CHANGELOG.md` (7.2) and the source. A **boot smoke test** (`bin/rails runner`) is the reliable way to catch it.

---

### 🟡 MEDIUM PRIORITY

#### 7. Solid Cache (Optional)

**What Changed:**
Rails 8.0 defaults to Solid Cache for caching (database-backed).

**Detection Pattern:**
```ruby
config.cache_store = :redis_cache_store
config.cache_store = :mem_cache_store
```

**Options:**

**Keep Redis/Memcached:**
```ruby
# No change needed - your existing setup still works
config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }
```

**Switch to Solid Cache:**
```ruby
# Gemfile
gem 'solid_cache'

# Install
rails solid_cache:install

# Config
config.cache_store = :solid_cache_store
```

---

#### 8. Solid Queue (Optional)

**What Changed:**
Rails 8.0 defaults to Solid Queue for background jobs (database-backed).

**Detection Pattern:**
```ruby
config.active_job.queue_adapter = :sidekiq
config.active_job.queue_adapter = :async
```

**Options:**

**Keep Sidekiq:**
```ruby
# No change needed
config.active_job.queue_adapter = :sidekiq
```

**Switch to Solid Queue:**
```ruby
# Gemfile
gem 'solid_queue'

# Install
rails solid_queue:install

# Config
config.active_job.queue_adapter = :solid_queue
```

---

#### 9. Solid Cable (Optional)

**What Changed:**
Rails 8.0 defaults to Solid Cable for WebSockets (database-backed).

**Detection Pattern:**
```yaml
# config/cable.yml
production:
  adapter: redis
```

**Options:**

**Keep Redis:**
```yaml
# No change needed
production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") %>
```

**Switch to Solid Cable:**
```yaml
production:
  adapter: solid_cable
  polling_interval: 0.1.seconds
```

---

#### 10. Docker/Thruster for Production

**What Changed:**
Rails 8.0 apps include Dockerfile and Thruster gem.

**Fix:**
If using Docker, add:
```ruby
# Gemfile
gem 'thruster'
```

Thruster provides:
- HTTP/2 support
- Asset compression
- Static file serving

---

#### 11. Kamal Deployment

**What Changed:**
Rails 8.0 includes Kamal configuration for deployment.

**New Files:**
- `config/deploy.yml`
- `.kamal/` directory

**If not using Kamal**, you can ignore or delete these files.

---

## Gem Resolution Notes (from a real 7.2 → 8.0 upgrade)

These came from resolving a large app's bundle against Rails 8.0.5.1 (see "Resolver dry-run and pre-load pass" in `workflows/gem-compatibility-workflow.md`). They are specific to this hop.

- **`uri >= 0.13.1` is a new activesupport 8.0 dependency.** An app pinning `uri ~> 0.10` can't resolve, and moving the pin removes `URI.escape` / `unescape` / `encode` / `decode` and changes `URI::DEFAULT_PARSER` to the RFC 3986 parser. Patterns `URI_ESCAPE_REMOVED` and `URI_DEFAULT_PARSER_REGEXP` cover app code. Gem fallout seen in practice:
  - `dartsass-ruby` 3.0.x fails to load (`URI::Parser.new(hash)`). Fix: `dartsass-sprockets >= 3.1`, which switches to `sassc-embedded`, plus any fork that hard-requires `dartsass-ruby`.
  - `paypalhttp` 1.0.1 form-encodes with `URI.escape`. `paypal-checkout-sdk` pins `paypalhttp ~> 1.0.1`, and the Checkout SDK line is deprecated in favor of `paypal-server-sdk`, so patch the encoder until that migration happens.
- **`rails-i18n` 8.x requires `railties >= 8.0`**, so it moves in the bump PR itself, not ahead of it.
- **Keep incidental majors out of the bump.** The resolver moves `minitest` 5 → 6 and `rdoc` → 8.x when Rails is unlocked, but Rails 8.0 requires neither (railties 8 needs `irb ~> 1.13`, and `irb` accepts `rdoc >= 4.0.0`). Pin both at their current majors in the bump PR. `rdoc` 8.x also declares `GPL-2.0-only` alongside `Ruby`, which license scanners flag.
- **Rack stays on 2.2.** Rails 8.0 supports Rack 2.2, and in a bundle with `sprockets` 3.x, `omniauth` 2.0, `rack-session` 1.x, `rackup` 1.x or `rack-protection` 2.x, Rack 3 won't resolve. Rack 3 is a separate project, not part of this hop.
- **Pre-loadable on 7.2** (they resolve on 7.2 to their 8.0 versions): `sprockets-rails` 3.5.x, `tilt` 2.9.x, `i18n` 1.15.x, `reline` 0.7.x, `io-console` 0.9.x, and a patch-level batch (`rack` 2.2.x, `loofah`, `rails-html-sanitizer`, `mail`, `net-imap`, `net-protocol`, `crass`, `cgi`, `pp`, `zeitwerk`). `i18n` 1.15.0 makes config storage (including `I18n.locale`) Fiber-aware, so watch locale behavior after it ships.
- **Check path and private gems by resolving, not with railsbump.** In-repo engines' gemspecs, and privately hosted gems with `rails < 8.0.0` caps, only show up as resolution failures.

## Solid Gems Decision Guide

| Current Setup | Recommendation |
|--------------|----------------|
| Redis for cache/jobs/cable | Keep Redis - simpler to maintain |
| Sidekiq with complex workflows | Keep Sidekiq |
| Simple background jobs | Consider Solid Queue |
| Need real-time WebSockets | Keep Redis for Cable |
| Want simpler infrastructure | Use Solid gems |
| Heroku/PaaS deployment | Solid gems or Redis add-on |
| Self-hosted/Docker | Either works well |

---

## Migration Steps

### Phase 1: Preparation
```bash
git checkout -b rails-80-upgrade

# Verify Ruby version
ruby -v  # Should be 3.1+
```

### Phase 2: Gemfile Updates
```ruby
# Gemfile
gem 'rails', '~> 8.0.0'

# Choose asset pipeline:
gem 'propshaft'  # New default
# OR
gem 'sprockets-rails'  # Keep existing

# Optional Solid gems (only if migrating):
# gem 'solid_cache'
# gem 'solid_queue'
# gem 'solid_cable'
```

```bash
bundle update rails
```

### Phase 3: Asset Pipeline Decision

**If keeping Sprockets:**
```ruby
# Gemfile
gem 'sprockets-rails'
```
No other changes needed!

**If migrating to Propshaft:**
1. Remove Sprockets-specific configs
2. Update asset structure if needed
3. Use cssbundling-rails for Sass

### Phase 4: Configuration
```bash
rails app:update
```

Update `config/application.rb`:
```ruby
config.load_defaults 8.0
```

Add to production.rb:
```ruby
config.assume_ssl = true
```

### Phase 5: Testing
- Verify assets load correctly
- Test caching if using Solid Cache
- Test background jobs
- Test WebSockets if applicable

---

## Propshaft Migration Checklist

- [ ] Remove `sprockets-rails` from Gemfile
- [ ] Add `propshaft` to Gemfile
- [ ] Remove `config.assets.*` from environment files
- [ ] Verify assets serve correctly
- [ ] If using Sass, add `cssbundling-rails`
- [ ] Update asset precompilation in deployment

---

## Common Issues

### Issue: Assets Not Loading

**Error:** 404 for CSS/JS files

**Cause:** Asset pipeline misconfigured

**Fix for Propshaft:**
```ruby
# No config needed - just place files in app/assets/
```

**Fix for Sprockets:**
```ruby
# Ensure sprockets-rails is in Gemfile
gem 'sprockets-rails'
```

### Issue: SSL Redirect Loop

**Error:** ERR_TOO_MANY_REDIRECTS

**Cause:** Missing assume_ssl behind proxy

**Fix:**
```ruby
config.assume_ssl = true
```

### Issue: Solid Queue Jobs Not Processing

**Error:** Jobs stuck in pending

**Cause:** Solid Queue supervisor not running

**Fix:**
```bash
bin/jobs  # Start job processor
```

---

## Gem Compatibility

| Gem | Minimum Version | Notes |
|-----|-----------------|-------|
| devise | 4.9.0 | Works well |
| sidekiq | 7.0.0 | Keep if using |
| rspec-rails | 6.0.0 | Update recommended |
| capybara | 3.39.0 | Works well |
| webmock | 3.19.0 | Works well |

---

## Files Changed by app:update

| File | Change |
|------|--------|
| Gemfile | Rails version, Propshaft |
| config/application.rb | load_defaults 8.0 |
| config/environments/production.rb | assume_ssl, cache config |
| config/database.yml | Multi-database structure |
| Dockerfile | New file |
| config/deploy.yml | New file (Kamal) |
| bin/jobs | New file (Solid Queue) |

---

## Resources

- [Rails 8.0 Release Notes](https://guides.rubyonrails.org/8_0_release_notes.html)
- [Propshaft Documentation](https://github.com/rails/propshaft)
- [Solid Cache](https://github.com/rails/solid_cache)
- [Solid Queue](https://github.com/rails/solid_queue)
- [Kamal Documentation](https://kamal-deploy.org)
