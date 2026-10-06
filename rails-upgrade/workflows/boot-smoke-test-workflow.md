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

Boot-fix-reboot finds one failure per boot. On a large app, collect them all instead:

**Resolve into the right lockfile.** Under bootboot, `DEPENDENCIES_NEXT=1 bundle lock` writes to `Gemfile.lock`. Use `--lockfile=Gemfile_next.lock` (or `bundle install`) and confirm `git diff --stat Gemfile.lock` is empty.

**Get past a known resolve blocker.** Give the next side alone the newer version (`if ENV["DEPENDENCIES_NEXT"] == "1"` pin) and re-resolve until the bundle resolves. Then diff the two lockfiles: every moved gem is must-move-with-bump or already ticketed.

**Load every constant and collect failures.** `eager_load!` stops at the first bad file. Walk Zeitwerk's expected constants instead:

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

Run the same scan on the current side and report only the difference.

**Expect app-level failures, not only gem ones:**
- Version tripwires from the previous hop (`raise ... unless Rails.version < "8.0"`). Replace them with a guard that works on both versions.
- Deprecation subscribers that raise in dev/test. The target Rails warns about the version after it; allow-list those warnings (they belong to the next hop).
- Framework validations the target adds at boot (check the version guide). When the error names only the first offender, temporarily patch the validating method to log and continue, so one boot lists them all.

**Boot all three environments.** Production eager-loads during boot. Boot development, test and production on both sides; only the differences count.

**Check how CI sets the variable.** A CI default of `DEPENDENCIES_NEXT=0` is truthy to bootboot and to `if ENV["DEPENDENCIES_NEXT"]` guards. Make sure guards compare to `"1"`, or default CI silently runs the next lockfile.

**Split the work.** Fixes that behave the same on both versions go in a pre-work PR; the `Gemfile_next` PR stays `Gemfile` + `Gemfile_next.lock`.

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
