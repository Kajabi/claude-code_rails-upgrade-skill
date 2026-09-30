# Gem Compatibility Reference

**When to read this:** Step 4.5's compatibility check returned at least one blocker (a gem with no released version that supports the target Rails). This file is the playbook for resolving those.

For the compatibility check itself, see `SKILL.md` Step 4.5 (primary: railsbump.org API; optional offline alternative: `next_rails` `bundle_report compatibility` when the CLI is installed separately; full procedure in `workflows/gem-compatibility-workflow.md`). Don't load this file unless you're already past the check and have blockers to work through.

This file used to carry static `gem × Rails-version` lookup tables. They drifted out of date and ignored the user's actual lockfile, so they were removed in favor of the live checks.

---

## Update order

When applying the bumps the compatibility check produced:

1. **Ruby** — meet the target Rails version's minimum.
2. **Rails** — update to the target version.
3. **Auth / authorization gems** — `devise`, `pundit`, `cancancan`, `doorkeeper`. Boot-blocking when wrong.
4. **Testing gems** — `rspec-rails`, `factory_bot_rails`, `capybara`, `shoulda-matchers`. Required to validate the upgrade.
5. **Everything else** — incrementally, one gem at a time (`bundle update <gem> --conservative`).

The compatibility check tells you the minimum target for each gem. Use it to decide which gems can stay on their current version and which need an explicit bump.

---

## Playbook: gem has no compatible version

When `bundle_report` lists a gem under "with no new compatible versions" or "with no new versions", or railsbump returns `incompatible` with no `earliest_compatible_version`, you have a blocker. Options, in order of preference:

1. **Look for a maintained fork** that adds Rails support — search GitHub for "fork of <gem>" with the target Rails in recent commits.
2. **Check the gem's open PRs / issues** for in-flight Rails support. If a PR is close to merging, ask the user whether to vendor the branch as a temporary `git:` source.
3. **Wait for an upstream release** if the gem is actively maintained and the next release is imminent.
4. **Switch to an alternative** when the gem is abandoned, OR when a Rails-removed API has a back-compat gem and you'd rather use it than refactor. Examples:
   - `paperclip` → `shrine` / Active Storage (paperclip is abandoned).
   - When moving Rails 3 → 4, `attr_accessible` was removed from core. Two paths: refactor controllers to strong parameters, OR add `protected_attributes` to keep `attr_accessible` working. The shim is a valid choice when the controller surface is large and the upgrade timeline is tight; refactoring later is still possible.
5. **Fork and patch** as a last resort. Document the fork in the Gemfile and open an upstream PR.

Surface the choice to the user — do not silently swap gems. Each option has a different long-term cost.

A common false-blocker pattern: gems that became unnecessary in a specific Rails release. They show up under "with no new versions" in `bundle_report`, but the fix is to remove them, not to find a compatible version.

- **Merged into Rails core**: `strong_parameters` was extracted from Rails 3.x as a backport, and is built into Rails ≥ 4.0 (`ActionController::Parameters`). Just remove the gem when targeting Rails 4 or newer.
- **Replaced by core / a successor gem**: `turbo-sprockets-rails3` is replaced by `sprockets-rails` in Rails 4. `paperclip` is unmaintained and replaced by Active Storage / Shrine.
- **Extracted as a back-compat shim**: `protected_attributes` is the *opposite* shape — it was extracted from Rails 4.0 specifically as a transitional shim for the old mass-assignment API and was never re-merged. It still exists as a separate gem, but if your code is moving to `strong_parameters` you can remove it once the controllers are converted.

If `bundle_report` flags one of these, check the target version guide before assuming you need a fork.

---

## Gems your organization owns

A private or in-house gem that caps Rails (`rails < 8.0`) is a blocker only you can clear. Widening the gemspec ceiling unblocks the resolver, but it doesn't show that the gem works on the target. Check the gem's own CI before relying on the widened release:

- **Which Rails does its CI actually run?** A gem's CI often tests only the Rails pinned in its development `Gemfile.lock`, which can be several minors behind every consumer. A green run then says nothing about the versions that matter.
- **Add a matrix of consumer versions.** Keep the gem's own `Gemfile` as is (it may also build a service image). Read the Rails requirement from an env var with the old pin as the default, add one `gemfiles/rails_X_Y.gemfile` per version that sets the var and calls `eval_gemfile File.expand_path("../Gemfile", __dir__)`, and select them in CI with `BUNDLE_GEMFILE`. In GitHub Actions, make the Rails version a real matrix axis and use `include` only to attach each gemfile. When a key is defined only in `include`, later entries overwrite earlier ones and the matrix collapses to one job.
- **Seed each matrix lockfile from the main one** (`cp Gemfile.lock gemfiles/rails_X_Y.gemfile.lock`, then `bundle lock --update rails <test gems>`). A fresh resolve can fail on `git:` dependencies whose branch no longer exists upstream; the locked revision still resolves.
- **Expect the test tooling to move.** Old `rspec-rails` / `database_cleaner-active_record` / `shoulda-matchers` versions often don't load on newer Rails, so the matrix lockfiles carry newer ones.
- **Read what the re-resolve pulled in.** Unpinned transitive gems can jump a major (e.g. `connection_pool` 2.x → 3.x, which accepts only keyword arguments) and expose latent bugs that consumers pinning the older major don't hit yet. Fix those in the gem before a consumer lifts the pin.

Found on kajabi-products' 7.2 → 8.0 hop: `kj_notify` CI ran only Rails 7.0.8.7 while the host ran 7.2. Its 7.2 matrix lockfile resolved `connection_pool` 3.0.2 and 14 specs failed on a positional-hash `ConnectionPool.new` call.

---

## Major bumps of a gem your code subclasses

When the only compatible version of a gem is a new major, and the app subclasses its classes (`ActiveInteraction::Base`, service-object bases, form objects), the changelog's "Upgrading" notes are necessary but not enough. Before planning slices:

- **Diff the API surface, not just the changelog.** Load each version in isolation (`gem install <gem> -v X --install-dir <tmp> --ignore-dependencies`, then `GEM_PATH=<tmp>:$(gem env gempath) ruby -e 'gem "<gem>", ARGV[0]; require ...'`). Diff `public_instance_methods`, `private_instance_methods`, and the methods on any value objects the app touches, minus `Object`'s. Removed methods are call sites to fix. **Added private methods are collision risks:** a subclass method or input with the same name now overrides framework behavior. A subclass that defines the same name replaces the framework step silently, with no error.
- **Reproduce each break in a ten-line script against both versions.** It confirms which breaks raise and which silently change behavior, and gives the PR body concrete evidence.
- **Resolve the enclosing class before counting a hit.** Name-based AST scans over-count: a `def filter` on a nested `View` or finder class inside an interaction file isn't an override. Walk the Prism tree and record the class stack for every hit. The same applies to spec rewrites: `subject.<name>` may refer to a nested object, not the class being renamed.
- **Split by what works on both versions.** Renames, `.to_h` conversions and reserved-name changes usually do, so they ship as independent slices ahead of the bump. Changes that need the new API on one side and the old on the other (a method that moved objects, a renamed callback) ride with the bump.
- **Add a guard spec to the bump PR** for any silent failure mode, e.g. "no descendant overrides the framework's private step", so it can't come back after the upgrade.

Found on kajabi-products' `active_interaction` 4.1 → 5.5 (needed for Rails 8.0). 5.0 adds a private `Base#filter` that runs input validation.
- Two interactions defined their own `filter`. On 5.5 one skipped type checks entirely, and the other raised `ArgumentError` on every call.
- Three declared `string :filter`, now a reserved name, which raised at class load.
- `Inputs` stopped being a Hash, so 45 `**inputs` splats raised `TypeError`.
- None of this was in the gem's upgrade notes. The first scan counted 7 overrides; only 2 were on interactions.
