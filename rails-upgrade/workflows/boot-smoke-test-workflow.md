# Boot Smoke Test Workflow

**When to run:** Step 4.6 of the upgrade workflow, after gem-compat (Step 4.5) and before report generation (Step 5).

**Why this step exists:** Step 4 (codebase grep) only sees the user's own code. Step 4.5 (railsbump API / optional `bundle_report compatibility`) only sees declared dependency constraints. Neither can detect a gem that resolves cleanly under the target Rails version but then crashes at boot because it calls a removed method or requires a removed file.

A booted Rails process is the only signal that catches that class of failure.

## Real examples this catches

| Hop | Gem | Failure mode | What the resolver / static patterns see |
|---|---|---|---|
| 7.1 → 7.2 | `database_cleaner-active_record 2.1.x` | `NoMethodError: undefined method 'schema_migration' for #<...PostgreSQLAdapter>` at first cleaner call | Resolver: ✓ compatible (no upper bound on activerecord). Patterns: ✗ no app-code reference. |
| 7.2 → 8.0 | `jbuilder 2.11.x` | `LoadError: cannot load such file -- active_support/proxy_object` at `Bundler.require` | Resolver: ✓ compatible (no upper bound on activesupport). Patterns: ✗ no app-code reference. |

In both cases the gem ships in default Rails-generated apps and the user did nothing wrong — the gem itself needed a newer minor version with target-Rails support.

## Procedure

### 1. Pick a boot trigger

Anything that loads `config/application.rb` under the next dependency set is sufficient. Cheapest options first:

```bash
# Cheapest: just boot the framework
DEPENDENCIES_NEXT=1 bundle exec rails runner "puts Rails.version"

# Slightly heavier: load the test environment without running specs
DEPENDENCIES_NEXT=1 bundle exec rspec --dry-run

# Heaviest but most thorough: full suite under target Rails
DEPENDENCIES_NEXT=1 bundle exec rspec
# or
DEPENDENCIES_NEXT=1 bundle exec rails test
```

Use `rails runner` first. If it boots cleanly, escalate to the full test suite — that catches gems whose problematic code only loads under a specific environment (e.g. test-only gems, eager-load-only paths).

### 2. Diagnose a failure

Boot failures usually show up as one of:

- `LoadError: cannot load such file -- <path>` — a gem `require`s a file Rails removed.
- `NoMethodError: undefined method '<x>' for <Rails internal>` — a gem calls a Rails API that was removed or renamed.
- `ArgumentError` / `TypeError` from a Rails class load — a gem passes args in a now-unsupported shape.

To find the offending gem:

```bash
# Search all bundled gem paths for the missing file / constant
find $(bundle show --paths | tr '\n' ' ') -name "*.rb" 2>/dev/null \
  | xargs grep -l '<missing-file-or-method>' 2>/dev/null
```

The output points at the gem version that needs to bump.

### 3. Resolve

For each offending gem:

1. Check rubygems for a newer minor or patch with target-Rails support:
   ```bash
   curl -s https://rubygems.org/api/v1/versions/<gem>.json \
     | ruby -rjson -e 'puts JSON.parse(STDIN.read).first(8).map{|x|x["number"]}'
   ```
2. Read the gem's CHANGELOG for the target-Rails-compat release.
3. Add a fix-before-bump entry to the upgrade report's gem-update list, citing:
   - The exact failure (`LoadError` / `NoMethodError` / etc.)
   - The minimum compatible version
   - Why a static check missed it (no upper bound declared)
4. Bump the floor in the Gemfile (`gem "<gem>", "~> <new-floor>"`, placing it inside the `if ENV["DEPENDENCIES_NEXT"]` branch if only the next side needs the bump) and re-run `bundle install` — bootboot will refresh both `Gemfile.lock` and `Gemfile_next.lock` in one pass.

### 4. Re-run boot

Repeat steps 1–3 until boot succeeds under `DEPENDENCIES_NEXT=1`. Then proceed to Step 5.

## Large apps: find every failure in one pass

On a large app, boot-fix-reboot finds one failure per boot. That's slow, and it hides how much work is left. These techniques come from the kajabi-products 7.2 → 8.0 smoke run, where a 12k-file app went from "does not resolve" to a complete blocker list in one session.

**Resolve into the right lockfile.** With bootboot 0.2.2, `DEPENDENCIES_NEXT=1 bundle lock` writes the next-side resolution into **`Gemfile.lock`**: bootboot's lockfile swap doesn't apply to `bundle lock`. Use `DEPENDENCIES_NEXT=1 bundle lock --lockfile=Gemfile_next.lock` or `DEPENDENCIES_NEXT=1 bundle install`. Confirm `git diff --stat Gemfile.lock` is empty afterwards.

**Get past a known resolve blocker to find the rest.** When the resolver stops on a blocker that already has a ticket, give the next side alone the newer version, then re-resolve. Use an `if ENV["DEPENDENCIES_NEXT"] == "1"` pin, e.g. `active_interaction ~> 5.5` while the current side stays on 4.1. Keep going until the bundle resolves. Afterwards, diff the two lockfiles' `specs:` sections: every gem that moved is either must-move-with-bump or already has a ticket.

**Load every constant and collect failures instead of stopping at the first.** `Rails.application.eager_load!` raises on the first bad file. Instead, walk Zeitwerk's expected constants and record each failure:

```ruby
# DEPENDENCIES_NEXT=1 RAILS_ENV=test bin/rails runner scan.rb
errors = Hash.new { |h, k| h[k] = [] }
Rails.autoloaders.each do |loader|
  loader.all_expected_cpaths.each do |path, cpath|
    next unless path.end_with?(".rb")
    Object.const_get(cpath)
  rescue Exception => e
    frame = e.backtrace&.find { |l| l.start_with?(Rails.root.to_s) }
    errors["#{e.class}: #{e.message.lines.first.strip[0, 160]}"] << "#{path} @ #{frame}"
  end
end
errors.sort_by { |_, v| -v.size }.each { |k, v| puts "[#{v.size}] #{k}", v.first(4).map { "   #{_1}" } }
```

Then **run the same scan on the current side**. Anything that also fails there, such as a spec file under a component-preview path, is a baseline artifact rather than a hop blocker. Report only the difference.

**Expect app-level failures, not only gem ones.** At 7.2 → 8.0 the first three boot failures came from the app's own code:
- **Version tripwires** left by the previous hop, e.g. `raise "Please remove this!" unless Rails.version < "8.0"`. Replace them with a guard that works on both versions, such as `if Rails.gem_version < Gem::Version.new("8.0")`.
- **Deprecation subscribers that raise in dev/test.** The target Rails emits deprecations about the version after it (8.0 warns about 8.1's `to_time` behavior). Add these to the allow-list rather than flipping the behavior. They belong to the next hop (see "Do not fix load_defaults-triggered runtime deprecation warnings about future Rails versions" in SKILL.md).
- **Rails 8.0 raises on invalid `resources … only:/except:` actions**, which 7.2 ignored. The error only names the first offender. To list every offender, temporarily prepend a module to `ActionDispatch::Routing::Mapper::Resources::Resource#invalid_only_except_options` in an uncommitted initializer. Have it log the `config/routes` caller and return `[]`. Prove the fix by dumping `routes.map { [verb, path.spec, defaults, name] }.sort` before and after on the current side; the dumps must be identical.

**Boot all three environments.** Development, test and production load differently: production eager-loads during boot. Run the production boot on both sides against a local DB (`DATABASE_URL`, `SECRET_KEY_BASE=dummy`, plus whatever `RAILS_SERVE_STATIC_FILES` / env vars the app needs). Only differences between the two sides count.

**Check how CI sets the variable.** Some CI configs export `DEPENDENCIES_NEXT=<param>` with a default of `0`. The string `"0"` is truthy to bootboot and to plain `if ENV["DEPENDENCIES_NEXT"]` code guards. Before the next side diverges from the current one, confirm that the Gemfile's `enable_dual_booting` guard compares to `"1"` and that no app code checks the variable for mere presence. Otherwise default CI silently runs against `Gemfile_next.lock`.

**Split the work into two PRs.** Put the fixes that behave the same on both versions (routes, tripwires, allow-list entries) in a pre-work PR off the base branch. Keep the `Gemfile_next` PR to `Gemfile` + `Gemfile_next.lock`, stacked on any resolve-blocker PR it needs.

## Output

A short report block to merge into Step 5's Comprehensive Upgrade Report:

```
Boot smoke test (Step 4.6):

  - Triggered: DEPENDENCIES_NEXT=1 bundle exec rspec --dry-run
  - Result: PASS / FAIL with N gem bumps required

If FAIL → bumps required (added to fix-before-bump):
  - <gem> <old> → <new>: <one-line failure reason>
  - ...
```

If the smoke test passes on the first run, record that explicitly — it is a positive signal that the resolver-level compat check covered everything for this hop.

## Notes

- The smoke test does not replace the post-bump test suite run in Step 6. It is a *boot* check, not a feature check. Step 6 still runs the full suite against both versions.
- Skip this step only if `Gemfile_next.lock` does not yet exist (very early in dual-boot setup — the lockfile seeding step in Step 2 hasn't run). In all other cases, run it.
