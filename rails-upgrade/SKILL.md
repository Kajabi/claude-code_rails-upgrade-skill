---
name: rails-upgrade
description: Analyzes Rails applications and generates comprehensive upgrade reports with breaking changes, deprecations, and step-by-step migration guides for Rails 2.3 through 8.1. Use when upgrading Rails applications, planning multi-hop upgrades, or querying version-specific changes. Based on FastRuby.io methodology and "The Complete Guide to Upgrade Rails" ebook.
---

# Rails Upgrade Assistant Skill

## Skill Identity
- **Name:** Rails Upgrade Assistant
- **Purpose:** Intelligent Rails application upgrades from 2.3 through 8.1
- **Skill Type:** Modular with external workflows and examples
- **Upgrade Strategy:** Sequential only (no version skipping)
- **Methodology:** Based on FastRuby.io upgrade best practices and "The Complete Guide to Upgrade Rails" ebook
- **Attribution:** Content based on "The Complete Guide to Upgrade Rails" by FastRuby.io (OmbuLabs)

---

## Dependencies

- **bootboot Bundler plugin** ([github.com/Shopify/bootboot](https://github.com/Shopify/bootboot)) — Provides the dual-boot mechanism this skill assumes. Bootboot keeps `Gemfile.lock` and `Gemfile_next.lock` in sync from a single `Gemfile` and lets the app boot with the alternate dependency set via `DEPENDENCIES_NEXT=1`. Step 2 of the workflow installs and configures it inline (no external skill required).
- **rails-load-defaults skill** ([github.com/ombulabs/claude-code_rails-load-defaults-skill](https://github.com/ombulabs/claude-code_rails-load-defaults-skill)) — Handles incremental `load_defaults` updates with tiered risk assessment (Tier 1: low-risk, Tier 2: needs codebase grep, Tier 3: requires human review). Used as the final step after the Rails version upgrade is complete.

---

## Core Methodology (FastRuby.io Approach)

This skill follows the proven FastRuby.io upgrade methodology:

1. **Incremental Upgrades** - Always upgrade one minor/major version at a time
2. **Assessment First** - Understand scope before making changes
3. **Dual-Boot Testing** - Test both versions during transition using the `bootboot` Bundler plugin (one `Gemfile`, `Gemfile.lock` + `Gemfile_next.lock`, `DEPENDENCIES_NEXT=1` to boot the alternate set)
4. **Test Coverage** - Ensure adequate test coverage before upgrading (aim for 80%+)
5. **Gem Compatibility** - Check gem compatibility at each step using RailsBump
6. **Deprecation Warnings** - Address deprecations before upgrading
7. **Backwards Compatible Changes** - Deploy small changes to production before version bump

**Key Resources:**
- Step 2 installs and configures `bootboot` inline — see "CRITICAL: Dual-Boot Setup with bootboot" below
- See `references/deprecation-warnings.md` for managing deprecations
- See `references/staying-current.md` for maintaining upgrades over time

---

## CRITICAL: Dual-Boot Setup with bootboot

This skill assumes [bootboot](https://github.com/Shopify/bootboot), a Bundler plugin, as the dual-boot mechanism. Bootboot keeps a single `Gemfile` whose conditional blocks resolve to different versions depending on an environment variable, and maintains both `Gemfile.lock` and `Gemfile_next.lock` in lockstep on every `bundle install`.

### Install (one time, run inside the Rails app)

1. Add the plugin loader to the top of `Gemfile`:

   ```ruby
   # Only enable bootboot in dev / test (it does not play well with some deploy
   # platforms — e.g. Heroku). Set ENABLE_BOOTBOOT=1 to opt in elsewhere.
   plugin "bootboot", "~> 0.2.2" if !ENV["RAILS_ENV"] || ENV["RAILS_ENV"] == "test" || ENV["ENABLE_BOOTBOOT"]
   Plugin.send(:load_plugin, "bootboot") if Plugin.installed?("bootboot")

   if ENV["DEPENDENCIES_NEXT"] == "1"
     enable_dual_booting if Plugin.installed?("bootboot")
   end
   ```

2. Express the upgrade in a conditional block lower in the `Gemfile`:

   ```ruby
   if ENV["DEPENDENCIES_NEXT"] == "1"
     gem "rails", "~> 7.2.0"
     # Any gems that must move forward together with Rails
   else
     gem "rails", "~> 7.1.0"
     # Current-version pins (often the same gem at an older line)
   end
   ```

3. Install the plugin and seed the second lockfile. **Do not run `bundle bootboot`** after hand-writing the block in step 1: that command appends its own copy of the loader / `enable_dual_booting` block to the Gemfile unconditionally, so you would end up with two.

   ```sh
   bundle install                       # installs the plugin, resolves Gemfile.lock
   cp Gemfile.lock Gemfile_next.lock    # seed; bootboot only syncs a lockfile that already exists
   DEPENDENCIES_NEXT=1 bundle install   # re-resolves Gemfile_next.lock against the next-side pins and installs those gems
   ```

   From here on every plain `bundle install` / `bundle update` keeps both lockfiles in sync, but only installs the current side's gems. Re-run `DEPENDENCIES_NEXT=1 bundle install` whenever the next-side pins change. Commit `Gemfile`, `Gemfile.lock`, and `Gemfile_next.lock`.

### Run the app on either side

```sh
# Current Rails (Gemfile.lock)
bundle exec rspec
bin/rails server

# Next Rails (Gemfile_next.lock)
DEPENDENCIES_NEXT=1 bundle exec rspec
DEPENDENCIES_NEXT=1 bin/rails server
```

### Code-level dual-boot guard

When a fix has to live on both sides, gate it on the same env var Bundler uses. **Do not** use `respond_to?` or other feature detection — `ENV["DEPENDENCIES_NEXT"]` matches what bootboot itself reads, so the boot-time gate and the code-time gate stay aligned.

```ruby
if ENV["DEPENDENCIES_NEXT"]
  # Code that requires the next Rails version
else
  # Code that runs against the current Rails version
end
```

If the project wraps the check in a helper (e.g. `AppConfig.dependencies_next?`), use that helper instead of inlining the env-var read. The cleanup workflow knows how to collapse both shapes.

### Caveats

- **Always set `DEPENDENCIES_NEXT=1` exactly.** Bootboot picks the lockfile on truthiness (`if ENV['DEPENDENCIES_NEXT']`), while the Gemfile block above compares to `"1"`. Any other value (`true`, `yes`) makes bootboot select `Gemfile_next.lock` while the Gemfile resolves the *current* pins into it. CI env blocks should use the string `"1"`; the code-level guard below stays a plain truthiness check because bootboot itself exports `1` while syncing.
- `bootboot` resolves and locks dependencies for both sets, but it does not switch your local Ruby. If the next set bumps Ruby, install the new Ruby separately (via `asdf` / `rbenv`) and switch before running `DEPENDENCIES_NEXT=1` commands.
- `bootboot` does **not** ship a `bundle_report compatibility` equivalent. Step 4.5 (Gem Compatibility) uses railsbump.org as the primary check. `bundle_report compatibility` from `next_rails` is available as an offline alternative if the user already has it installed as a CLI; see `workflows/gem-compatibility-workflow.md`.

---

## Trigger Patterns

Claude should activate this skill when user says:

**Upgrade Requests:**
- "Upgrade my Rails app to [version]"
- "Help me upgrade from Rails [x] to [y]"
- "What breaking changes are in Rails [version]?"
- "Plan my upgrade from [x] to [y]"
- "What Rails version am I using?"
- "Analyze my Rails app for upgrade"
- "Find breaking changes in my code"
- "Check my app for Rails [version] compatibility"

**Specific Report Requests:**
- "Show me the app:update changes"
- "Preview configuration changes for Rails [version]"
- "Generate the upgrade report"
- "What will change if I upgrade?"

**Upgrade Cleanup Requests (delegate to the `upgrade-cleanup` plugin):**
- "Finish the upgrade"
- "Clean up after my Rails upgrade"
- "Remove the dual-boot setup"
- "Drop the DEPENDENCIES_NEXT branches"
- "We're done upgrading to Rails [version]"

---

## CRITICAL: Sequential Upgrade Strategy

### ⚠️ Version Skipping is NOT Allowed

Rails upgrades MUST follow a sequential path. Examples:

**For Rails 5.x to 8.x:**
```
5.0.x → 5.1.x → 5.2.x → 6.0.x → 6.1.x → 7.0.x → 7.1.x → 7.2.x → 8.0.x → 8.1.x
```

**You CANNOT skip versions.** Examples:
- ❌ 5.2 → 6.1 (skips 6.0)
- ❌ 6.0 → 7.0 (skips 6.1)
- ❌ 7.0 → 8.0 (skips 7.1 and 7.2)
- ✅ 5.2 → 6.0 (correct)
- ✅ 7.0 → 7.1 (correct)
- ✅ 7.2 → 8.0 (correct)

If user requests a multi-hop upgrade (e.g., 5.2 → 8.1):
1. Explain the sequential requirement
2. Break it into individual hops
3. Generate separate reports for each hop
4. Recommend completing each hop fully before moving to next

---

## Supported Upgrade Paths

### Legacy Rails (2.3 - 4.2)

| From | To | Difficulty | Key Changes | Ruby Required |
|------|-----|-----------|-------------|---------------|
| 2.3.x | 3.0.x | Very Hard | XSS protection, routes syntax | 1.8.7 - 1.9.3 |
| 3.0.x | 3.1.x | Medium | Asset pipeline, jQuery | 1.8.7 - 1.9.3 |
| 3.1.x | 3.2.x | Easy | Ruby 1.9.3 support | 1.8.7 - 2.0 |
| 3.2.x | 4.0.x | Hard | Strong Parameters, Turbolinks | 1.9.3+ |
| 4.0.x | 4.1.x | Medium | Spring, secrets.yml | 1.9.3+ |
| 4.1.x | 4.2.x | Medium | ActiveJob, Web Console | 1.9.3+ |
| 4.2.x | 5.0.x | Hard | ActionCable, API mode, ApplicationRecord | 2.2.2+ |

### Modern Rails (5.0 - 8.1)

| From | To | Difficulty | Key Changes | Ruby Required |
|------|-----|-----------|-------------|---------------|
| 5.0.x | 5.1.x | Easy | Encrypted secrets, yarn default | 2.2.2+ |
| 5.1.x | 5.2.x | Medium | Active Storage, credentials | 2.2.2+ |
| 5.2.x | 6.0.x | Hard | Zeitwerk, Action Mailbox/Text | 2.5.0+ |
| 6.0.x | 6.1.x | Medium | Horizontal sharding, strict loading | 2.5.0+ |
| 6.1.x | 7.0.x | Hard | Hotwire/Turbo, Import Maps | 2.7.0+ |
| 7.0.x | 7.1.x | Medium | Composite keys, async queries | 2.7.0+ |
| 7.1.x | 7.2.x | Medium | Transaction-aware jobs, DevContainers | 3.1.0+ |
| 7.2.x | 8.0.x | Very Hard | Propshaft, Solid gems, Kamal | 3.2.0+ |
| 8.0.x | 8.1.x | Easy | Bundler-audit, max_connections | 3.2.0+ |

---

## Available Resources

### Core Documentation
- `SKILL.md` - This file (entry point)

### Version-Specific Guides (Load as needed)

**Legacy Rails:**
- `version-guides/upgrade-3.2-to-4.0.md` - Rails 3.2 → 4.0 (Strong Parameters)
- `version-guides/upgrade-4.0-to-4.1.md` - Rails 4.0 → 4.1 (Spring, secrets.yml, enums)
- `version-guides/upgrade-4.1-to-4.2.md` - Rails 4.1 → 4.2 (ActiveJob, Web Console)
- `version-guides/upgrade-4.2-to-5.0.md` - Rails 4.2 → 5.0 (ApplicationRecord)

**Modern Rails:**
- `version-guides/upgrade-5.0-to-5.1.md` - Rails 5.0 → 5.1 (Encrypted secrets)
- `version-guides/upgrade-5.1-to-5.2.md` - Rails 5.1 → 5.2 (Active Storage, Credentials)
- `version-guides/upgrade-5.2-to-6.0.md` - Rails 5.2 → 6.0 (Zeitwerk)
- `version-guides/upgrade-6.0-to-6.1.md` - Rails 6.0 → 6.1 (Horizontal sharding)
- `version-guides/upgrade-6.1-to-7.0.md` - Rails 6.1 → 7.0 (Hotwire/Turbo)
- `version-guides/upgrade-7.0-to-7.1.md` - Rails 7.0 → 7.1 (Composite keys)
- `version-guides/upgrade-7.1-to-7.2.md` - Rails 7.1 → 7.2 (Transaction jobs)
- `version-guides/upgrade-7.2-to-8.0.md` - Rails 7.2 → 8.0 (Propshaft)
- `version-guides/upgrade-8.0-to-8.1.md` - Rails 8.0 → 8.1 (bundler-audit)

### Workflow Guides (Load when generating deliverables)
- `workflows/test-suite-verification-workflow.md` - **MANDATORY FIRST STEP** - How to run and verify test suite
- `workflows/no-test-suite-smoke-workflow.md` - **Load from Step 1 when no runnable RSpec/Minitest suite exists** - Rails boot, routes, migration-status, and build smoke baseline with partial-confidence reporting
- `workflows/direct-detection-workflow.md` - How to run breaking change detection directly
- `workflows/upgrade-report-workflow.md` - How to generate upgrade reports
- `workflows/gem-compatibility-workflow.md` - **Load in Step 4.5** - Per-lockfile gem compatibility check against the target Rails version. Documents the primary (railsbump.org API), the optional offline alternative (`bundle_report compatibility` from `next_rails`, available when the user has the CLI installed separately), and how to reconcile their output.
- `workflows/boot-smoke-test-workflow.md` - **Load in Step 4.6** - Boot Rails under the next dependency set (`DEPENDENCIES_NEXT=1`) to catch gem-level runtime incompat that the resolver can't see (gems calling removed Rails internals or `require`-ing removed files).
- `workflows/ci-sync-workflow.md` - **MANDATORY before opening the upgrade PR** - How to verify CI config matches the upgraded Gemfile
- `workflows/app-update-preview-workflow.md` - How to generate app:update previews
- **`upgrade-cleanup` companion plugin** - User-triggered. Removes dual-boot scaffolding and drops `if ENV["DEPENDENCIES_NEXT"]` branches. Deprecation triage stays with this skill for the next hop.

### Examples (Load when user needs clarification)
- `examples/simple-upgrade.md` - Single-hop upgrade example
- `examples/multi-hop-upgrade.md` - Multi-hop upgrade example

### External Dependencies
- **bootboot Bundler plugin** - Dual-boot mechanism, installed and configured inline in Step 2 (https://github.com/Shopify/bootboot)
- **rails-load-defaults skill** - Incremental load_defaults alignment (Step 7, final step) (https://github.com/ombulabs/claude-code_rails-load-defaults-skill)

### Reference Materials
- `references/deprecation-warnings.md` - Finding and fixing deprecations
- `references/staying-current.md` - Keeping up with Rails releases
- `references/breaking-changes-by-version.md` - Quick lookup
- `references/multi-hop-strategy.md` - Multi-version planning
- `references/testing-checklist.md` - Comprehensive testing
- `references/gem-compatibility.md` - Gem update order and the "no compatible version" playbook (fork / vendor / replace). Load only when Step 4.5's compatibility check produced blockers.
- `references/js-compressor-sprockets-mismatch.md` - Keeping terser / closure-compiler working when the target Rails pins Sprockets to the 2.x line. Load only when JS_COMPRESSOR_GEM_MISMATCH fires.

### Detection Pattern Resources
- `detection-scripts/patterns/rails-*.yml` - Version-specific patterns for direct detection

### Report Templates
- `templates/upgrade-report-template.md` - Main upgrade report structure
- `templates/app-update-preview-template.md` - Configuration preview

---

## High-Level Workflow

When user requests an upgrade, follow this workflow:

### Step 0: Verify Latest Patch Version (MANDATORY PRE-STEP)
```
⚠️  THIS STEP IS REQUIRED BEFORE ANY OTHER WORK

1. Read Gemfile.lock to find exact current Rails version (e.g., 3.2.19)
2. Compare against latest patch for that series:
   - EOL series (≤ 7.1): use static table in references/multi-hop-strategy.md
   - Active series (≥ 7.2): query RubyGems API (see references/multi-hop-strategy.md for commands)
3. If current version < latest patch:
   - INFORM user: "Your app is on Rails X.Y.Z but the latest patch is X.Y.W"
   - Guide through Gemfile update and bundle update rails
   - Run test suite after patch upgrade
   - Deploy patch upgrade before proceeding
   - Do NOT proceed to next minor/major until on latest patch
4. If current version == latest patch:
   - Proceed to Step 1
```

**Why patch first:** Patch releases contain security fixes, bug fixes, and additional deprecation warnings. Starting the version hop on the latest patch is safer (the security fixes are already in production) and easier to debug (the new deprecation warnings surface issues that would otherwise show up mid-upgrade).

### Step 1: Run Test Suite (MANDATORY FIRST STEP)
```
⚠️  THIS STEP IS REQUIRED BEFORE ANY OTHER WORK

1. Read: workflows/test-suite-verification-workflow.md
2. Detect test framework (RSpec, Minitest, or both)
3. Run test suite with: bundle exec rspec OR bundle exec rails test
4. Capture results: total tests, passing, failing, pending
5. If no runnable test suite exists:
   - Load: workflows/no-test-suite-smoke-workflow.md
   - Run the safe read-only smoke baseline: Rails boot, test-env boot when possible, routes load, migration status, and asset/build command if present
   - Record baseline confidence as partial
   - Continue only if boot/routes checks pass and the user accepts the risk of proceeding without real tests
6. If ANY tests fail:
   - STOP the upgrade process
   - Report failing tests to user
   - Offer to help fix failing tests
   - Do NOT proceed until all tests pass
7. If all tests pass:
   - Record baseline metrics (test count, coverage if available)
   - Proceed to Step 2
```

### Step 2: Set Up Dual-Boot with bootboot (EARLY SETUP)
```
Follow the "CRITICAL: Dual-Boot Setup with bootboot" section above.
The setup is short enough to do inline — no external skill required.

Before editing the Gemfile:
- Check whether bootboot is already wired up (grep for `plugin "bootboot"`
  in Gemfile and the presence of Gemfile_next.lock). If both are present,
  skip the install steps and reuse the existing scaffolding.
- Check whether next_rails was previously used (presence of Gemfile.next /
  Gemfile.next.lock, a local `next?` method in the Gemfile, or
  NextRails.next? references). If yes, ask the user whether to migrate
  from next_rails to bootboot or leave the existing setup. Do not
  silently delete the existing dual-boot scaffolding.

Then:
- Add the bootboot plugin loader and the `enable_dual_booting` block
  to Gemfile (snippet in the section above).
- Express the upgrade as an `if ENV["DEPENDENCIES_NEXT"] == "1"` block
  pinning the target Rails version.
- Run `bundle install`, then `cp Gemfile.lock Gemfile_next.lock`, then
  `DEPENDENCIES_NEXT=1 bundle install`. Do NOT run `bundle bootboot` — it
  appends a second copy of the block you just wrote.
- Confirm both Gemfile.lock and Gemfile_next.lock now exist and pin the
  expected Rails versions.
- Smoke-check the next side with `DEPENDENCIES_NEXT=1 bin/rails runner "puts Rails.version"`.
```

### Step 3: Validate Upgrade Path
```
1. Check if upgrade is single-hop or multi-hop
2. If multi-hop, explain sequential requirement
3. Plan individual hops
```

### Step 4: Run Breaking Changes Detection (DIRECT)
```
Claude runs detection directly using tools - NO script generation needed

1. Read: workflows/direct-detection-workflow.md
2. Read: detection-scripts/patterns/rails-{VERSION}-patterns.yml
3. For each pattern in the patterns file:
   - Use Grep tool to search for the pattern
   - Collect file paths and line numbers
   - Store findings with context
4. Read: version-guides/upgrade-{FROM}-to-{TO}.md for context
5. Compile all findings into structured data
```

### Step 4.5: Check Gem Compatibility Against Target Rails
```
Determines which gems must be bumped before the Rails version change can resolve.

1. Read: workflows/gem-compatibility-workflow.md and follow it. The
   workflow documents the primary check (railsbump.org API), the
   optional offline alternative (next_rails bundle_report, available
   when the CLI is installed separately), and the bucket mapping for
   both.
2. Pass the resulting three buckets — required bumps, blockers,
   already compatible — into Step 5's report so the gem-update
   section reflects real per-lockfile data.
3. If any blockers exist, load references/gem-compatibility.md for
   the fork/replace/vendor playbook and the gem update order. Skip
   otherwise.
```

### Step 4.6: Boot Smoke Test on the Next Dependency Set
```
Catches the gem-internal incompatibilities that Step 4 (codebase grep) and
Step 4.5 (resolver-level compat check) cannot see.

A gem can declare loose Rails constraints — no upper bound on activerecord /
activesupport — and both railsbump and `bundle install` will call it
"compatible." But at runtime, the gem may:

  - call a Rails internal that was removed at the target version
    (e.g. database_cleaner-active_record 2.1.x calling
    AR::ConnectionAdapters#schema_migration, removed in Rails 7.2)
  - require a file that was removed at the target version
    (e.g. jbuilder 2.11.x doing `require "active_support/proxy_object"`,
    removed in Rails 8.0)

These surface only when something boots Rails. Catching them here, before
the report is written, lets them land in fix-before-bump where they belong
instead of mid-implementation.

1. Read: workflows/boot-smoke-test-workflow.md
2. Run a Rails-loading command under the next dependency set:
     DEPENDENCIES_NEXT=1 bundle exec rspec --dry-run
   (or `DEPENDENCIES_NEXT=1 bin/rails runner "puts Rails.version"`, or
   `DEPENDENCIES_NEXT=1 bundle exec rspec` if the suite is fast enough —
   anything that triggers `Bundler.require(*Rails.groups)` and the
   framework boot under Gemfile_next.lock.)
3. If boot fails, capture the LoadError / NoMethodError trace, identify
   the offending gem (grep the bundle paths for the missing constant or
   file), check rubygems for a newer version with target-Rails compat,
   and add the bump to the fix-before-bump bucket for Step 5.
4. Re-run the boot smoke test until it succeeds. Then proceed to Step 5.
```

### Step 5: Load Report Resources & Generate Reports
```
1. Read: templates/upgrade-report-template.md
2. Read: templates/app-update-preview-template.md
3. Read: workflows/upgrade-report-workflow.md
4. Read: workflows/app-update-preview-workflow.md
```

**Deliverable #1: Comprehensive Upgrade Report**
- **Input:** Direct detection findings + version guide data
- **Output:** A report covering findings grouped into the two buckets defined in `workflows/direct-detection-workflow.md` — **fix-before-bump** (`kind: breaking` and `kind: deprecation`) and **fix-when-ready** (`kind: migration` and `kind: optional`) — with OLD vs NEW code examples taken from the user's actual files, custom-code warnings flagged with ⚠️, a step-by-step migration plan, a testing checklist, and a rollback plan.

**Deliverable #2: app:update Preview**
- **Input:** Actual config files + findings
- **Output:** A preview showing exact configuration file changes (OLD vs NEW), a list of new files that will be created, and a per-file impact assessment (HIGH / MEDIUM / LOW).

### Step 6: Present Reports & Implement Changes
```
1. Present Comprehensive Upgrade Report first
2. Present app:update Preview Report second
3. Apply fix-before-bump changes (`kind: breaking` and `kind: deprecation`). Most fixes are direct rewrites — the new API typically works on both sides of the dual-boot pair (e.g., `update_attributes` → `update`). Use an `if ENV["DEPENDENCIES_NEXT"]` guard only when the fix requires target-version-only APIs that don't exist in the current Rails
4. Update the Gemfile's next-side conditional to the target Rails version (and any required gem bumps), then run `bundle install` so both Gemfile.lock and Gemfile_next.lock update
5. Run test suite against both versions (`bundle exec rspec` and `DEPENDENCIES_NEXT=1 bundle exec rspec`)
6. **Check CI config matches the upgraded Gemfile** — load `workflows/ci-sync-workflow.md`, fix any mismatches before proceeding
7. Deploy and verify
```

**Do not fix `load_defaults`-triggered runtime deprecation warnings about *future* Rails versions during this hop.** This caveat covers post-bump runtime warnings emitted by Rails X+1 about behavior scheduled to change in X+2 — typically surfaced once `load_defaults X.Y` flips on in Step 7. Those belong to the *next* upgrade cycle and are addressed before the next version bump.

This is **not** a contradiction of fix-before-bump. The `kind: deprecation` patterns from Step 4's detection are warnings emitted by the *current* Rails version about APIs that go away at the *target* version — they stay in fix-before-bump and should be addressed in this hop.

Triaging tomorrow's deprecation warnings now expands the scope of the current hop and risks shipping a half-finished change.

### Step 7: Align load_defaults
```
⚠️  THIS STEP HAPPENS AFTER THE UPGRADE IS COMPLETE

1. DELEGATE to the rails-load-defaults skill
2. That skill walks through each config change one at a time, grouped by risk tier
3. Tests are re-run between each change
4. Consolidates into config/application.rb when done
```

### Step 8: Mention Cleanup (USER-TRIGGERED)
```
⚠️  DO NOT AUTO-RUN. Mention it; let the user decide.

1. Tell the user the cleanup option exists
2. Delegate to the upgrade-cleanup plugin only when the user explicitly asks
   ("finish the upgrade", "clean up dual-boot", "drop the DEPENDENCIES_NEXT branches")
3. The cleanup plugin removes `if ENV["DEPENDENCIES_NEXT"]` branches and
   retires the bootboot scaffolding (plugin line, `enable_dual_booting`
   block, Gemfile_next.lock, conditional gem groups). Deprecation triage
   stays with this skill for the next hop, not with cleanup.
```

**Sample wording the agent can crib from when prompting the user:**

> Rails X.Y is in. When you're ready to remove dual-boot scaffolding (drop `if ENV["DEPENDENCIES_NEXT"]` branches, retire the bootboot plugin and `Gemfile_next.lock`), ask me to clean up. If you're heading straight to the next hop, keeping dual-boot in place is also fine.

---

## Pre-Upgrade Checklist (FastRuby.io Best Practices)

Before starting ANY upgrade:

### 1. Test Coverage Assessment (AUTOMATED - Step 1 of Workflow)
- [x] Run test suite - all tests passing? **← Claude runs this automatically**
- [x] Check test coverage (aim for >70%) **← Claude captures this if SimpleCov is configured**
- [ ] Review critical paths have coverage

**Note:** This step is now automated. Claude will run the test suite and BLOCK the upgrade if any tests fail.

### 2. Dependency Audit
- [ ] Run `bundle outdated`
- [ ] Check gem compatibility with target Rails version
- [ ] Identify gems that need upgrading first

### 3. Database Backup
- [ ] Backup production database
- [ ] Backup development/staging databases
- [ ] Verify backup restore process works

### 4. Git Branch Strategy
- [ ] Create upgrade branch from main/master
- [ ] Set up CI for upgrade branch
- [ ] Plan merge strategy

### 5. Deprecation Warnings
- [ ] Run app with Rails deprecations turned on (configured in config/environment files)
- [ ] Address existing deprecation warnings
- [ ] Enable verbose deprecations in test environment

---

## Common Request Patterns

### Pattern 1: Full Upgrade Request
**User says:** "Upgrade my Rails app to 8.1"

**Action - Step 0 (MANDATORY: Verify Latest Patch):**
1. Read `Gemfile.lock` for exact Rails version
2. Compare against latest patch for that series (see `references/multi-hop-strategy.md`)
3. If not on latest patch → Guide user through patch upgrade first
4. If on latest patch → Proceed to Step 1

**Action - Step 1 (MANDATORY: Verify Tests Pass):**
1. Load: `workflows/test-suite-verification-workflow.md`
2. Detect test framework (RSpec or Minitest)
3. Run test suite: `bundle exec rspec` or `bundle exec rails test`
4. If tests FAIL → STOP and help fix tests first
5. If tests PASS → Record baseline and proceed

**Action - Step 2 (Set Up Dual-Boot):**
1. Follow the "CRITICAL: Dual-Boot Setup with bootboot" section above for inline setup
2. Wire bootboot into the Gemfile, run `bundle install`, seed `Gemfile_next.lock` (`cp Gemfile.lock Gemfile_next.lock && DEPENDENCIES_NEXT=1 bundle install`), and extend CI to run the next-dependency-set side (`DEPENDENCIES_NEXT=1`)

**Action - Step 3 (Validate Upgrade Path):**
1. Validate upgrade path (single-hop vs multi-hop)

**Action - Step 4 (Run Detection Directly):**
1. Load: `workflows/direct-detection-workflow.md`
2. Load: `detection-scripts/patterns/rails-{VERSION}-patterns.yml`
3. Use Grep/Glob/Read tools to search for each pattern
4. Collect findings with file:line references

**Action - Step 4.5 (Check Gem Compatibility):**
1. Load: `workflows/gem-compatibility-workflow.md` and follow it
2. Run primary check (railsbump API); fall back to `bundle_report compatibility` only if the user has `next_rails` installed as a standalone CLI and the workflow's conditions trigger
3. Pass the resulting buckets (required bumps, blockers, already compatible) into Step 5's report
4. If blockers exist, load `references/gem-compatibility.md` for the fork/replace/vendor playbook

**Action - Step 5 (Generate Reports):**
1. Load: `workflows/upgrade-report-workflow.md`
2. Load: `workflows/app-update-preview-workflow.md`
3. Generate Comprehensive Upgrade Report (using direct findings)
4. Generate app:update Preview (using actual config files)
5. Present both reports to user

**Action - Step 6 (Implement & Upgrade):**
1. Apply fix-before-bump changes (`kind: breaking` and `kind: deprecation`). Most fixes are direct rewrites — the new API typically works on both sides of the dual-boot pair (e.g., `update_attributes` → `update`). Use an `if ENV["DEPENDENCIES_NEXT"]` guard only when the fix requires target-version-only APIs that don't exist in the current Rails
2. Update the Gemfile's next-side conditional to the target Rails version and run `bundle install` so both Gemfile.lock and Gemfile_next.lock update
3. Run tests against both versions (`bundle exec rspec` and `DEPENDENCIES_NEXT=1 bundle exec rspec`)
4. **Check CI config matches the upgraded Gemfile** (`workflows/ci-sync-workflow.md`) — fix any mismatches before declaring Step 6 complete
5. Deploy and verify

**Action - Step 7 (Align load_defaults - FINAL):**
1. DELEGATE to the `rails-load-defaults` skill
2. Walk through each config incrementally after the upgrade is complete

### Pattern 2: Multi-Hop Request
**User says:** "Help me upgrade from Rails 5.2 to 8.1"

**Action - Step 0 (MANDATORY: Verify Latest Patch):**
1. Check exact current version from `Gemfile.lock`
2. If not on latest patch of current series → Upgrade to latest patch first
3. For multi-hop: This check applies at the START and again after each hop

**Action - Step 1 (MANDATORY: Verify Tests Pass):**
1. Run test suite BEFORE planning any upgrade work
2. If tests fail → STOP and fix first
3. If tests pass → Proceed with planning

**Action - Step 2 (Set Up Dual-Boot):**
1. Follow the inline bootboot setup (see "CRITICAL: Dual-Boot Setup with bootboot") if not already set up
2. Dual-boot stays active throughout the multi-hop process — between hops, update the next-side conditional in the Gemfile to the new target and run `bundle install` again

**Action - Step 3 (Plan & Execute):**
1. Explain sequential requirement
2. Calculate hops: 5.2 → 6.0 → 6.1 → 7.0 → 7.1 → 7.2 → 8.0 → 8.1
3. Reference: `references/multi-hop-strategy.md`
4. Follow Pattern 1 Steps 4-6 for FIRST hop (5.2 → 6.0)
5. After first hop complete, repeat for next hops
6. **IMPORTANT:** After each hop, align load_defaults to the new version before starting the next hop

### Pattern 3: Breaking Changes Analysis Only
**User says:** "What breaking changes affect my app for Rails 8.0?"

This pattern is analysis-only — it intentionally skips Step 2 (Dual-Boot setup) and Step 3 (Validate Upgrade Path) because the user is not yet committing to an upgrade.

**Action - Step 0 (MANDATORY: Verify Latest Patch):**
1. Check if on latest patch — warn if not, recommend patching first

**Action - Step 1 (MANDATORY: Verify Tests Pass):**
1. Run test suite first
2. If tests fail → Warn user and recommend fixing first
3. If tests pass → Proceed with analysis

**Action - Step 4 (Run Detection):**
1. Load: `workflows/direct-detection-workflow.md`
2. Run detection directly using tools
3. Present findings summary
4. Offer to generate full upgrade report

---

## Quality Checklist

Before delivering, verify:

**For Direct Detection:**
- [ ] All patterns from version-specific YAML file checked
- [ ] Grep/Glob tools used correctly for each pattern
- [ ] File:line references collected for all findings
- [ ] Context captured for each finding

**For Comprehensive Upgrade Report:**
- [ ] All {PLACEHOLDERS} replaced with actual values
- [ ] Used ACTUAL findings from direct detection (not generic examples)
- [ ] Findings grouped into the two buckets — fix-before-bump (`kind: breaking` and `kind: deprecation`) and fix-when-ready (`kind: migration` and `kind: optional`) — with real file:line references
- [ ] Custom code warnings based on actual detected issues
- [ ] Code examples use user's actual code from affected files
- [ ] Next steps clearly outlined

**For CI Config Check (Step 6, before opening the PR):**
- [ ] Every CI file in the repo enumerated (GitHub Actions, CircleCI, Jenkins, GitLab, etc.)
- [ ] Ruby version, Rails matrix, and service versions diffed against the upgraded Gemfile
- [ ] CI sync report produced with per-file verdict
- [ ] All DRIFT entries fixed; overall verdict is OK

**For app:update Preview:**
- [ ] All {PLACEHOLDERS} replaced with actual values
- [ ] File list matches user's actual config files
- [ ] Diffs based on real current config vs target version
- [ ] Next steps clearly outlined

---

## Key Principles

1. **ALWAYS Verify Latest Patch First** (MANDATORY - ensure app is on latest patch of current series before any version hop)
2. **ALWAYS Run Test Suite** (MANDATORY - no exceptions, no upgrade work until tests pass)
3. **Block on Failing Tests** (if tests fail, STOP and help fix them before any upgrade work)
4. **Set Up Dual-Boot Early** (dual-boot is Step 2, right after tests pass - run both versions during the entire transition)
5. **Run Detection Directly** (use Grep/Glob/Read tools - no script generation needed)
6. **Always Use Actual Findings** (no generic examples in reports)
7. **Always Flag Custom Code** (with ⚠️ warnings based on detected issues)
8. **Always Use Templates** (for consistency)
9. **Always Check Quality** (before delivery)
10. **Load Workflows as Needed** (don't hold everything in memory)
11. **Sequential Process is Critical** (patch check → tests → dual-boot → validate path → detection → reports → implement → load_defaults)
12. **Follow FastRuby.io Methodology** (incremental upgrades, assessment first)
13. **Always Use `ENV["DEPENDENCIES_NEXT"]` for Dual-Boot Code** (NEVER use `respond_to?` for version branching. Bootboot reads the same env var to pick a lockfile, so the boot-time gate and the code-time gate stay aligned. See "CRITICAL: Dual-Boot Setup with bootboot" for patterns.)
14. **Check CI Config Before Opening the PR** (run `workflows/ci-sync-workflow.md` to make sure every CI file matches the upgraded Gemfile — stale CI is the most common cause of red builds on upgrade PRs)
15. **Align load_defaults After the Version Bump** (load_defaults update happens AFTER the Rails version upgrade is complete)
16. **Mention, Don't Auto-Run, Cleanup** (after the upgrade ships, mention the `upgrade-cleanup` plugin. Delegate to it only when the user explicitly asks: "finish the upgrade", "clean up dual-boot", "drop the DEPENDENCIES_NEXT branches". Cleanup removes `if ENV["DEPENDENCIES_NEXT"]` branches and retires bootboot scaffolding. Deprecation triage stays with this skill for the next hop.)

---

## Success Criteria

A successful upgrade assistance session:

✅ **Verified latest patch version** (Step 0 - MANDATORY)
✅ **Upgraded to latest patch if needed** (before any minor/major hop)
✅ **Ran test suite** (Step 1 - MANDATORY)
✅ **Verified all tests pass** (blocked if tests failed)
✅ **Recorded baseline metrics** (test count, coverage)
✅ **Set up dual-boot** (Step 2 - early, before upgrading)
✅ **Validated upgrade path** (Step 3 - single-hop vs multi-hop, hops planned)
✅ **Ran detection directly** (using Grep/Glob/Read tools - no script)
✅ **Generated Comprehensive Upgrade Report** using actual findings
✅ **Generated app:update Preview** using actual config files
✅ Used user's actual code from findings (not generic examples)
✅ Flagged all custom code with ⚠️ warnings based on detected issues
✅ **Implemented changes and upgraded Rails version**
✅ **Verified CI config matches the upgraded Gemfile** (Ruby, Rails matrix, services — all mismatches fixed before opening the PR)
✅ **Aligned load_defaults** (after upgrade is complete)
✅ **Mentioned cleanup after the upgrade shipped** (pointed to the `upgrade-cleanup` plugin without auto-running it; delegated only when the user explicitly asked)
✅ Provided clear next steps
✅ Offered to help implement changes

---

See [CHANGELOG.md](CHANGELOG.md) for version history and current version.
