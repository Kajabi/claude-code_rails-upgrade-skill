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

Widening a private gem's Rails ceiling unblocks the resolver; it doesn't prove the gem works. Before relying on the widened release:

1. Check which Rails the gem's CI runs (often only its own lockfile's, behind every consumer).
2. Add a matrix of consumer versions: one `gemfiles/rails_X_Y.gemfile` per version that sets the Rails requirement and `eval_gemfile`s the main Gemfile, selected with `BUNDLE_GEMFILE`. Seed each lockfile from the main one (`cp`, then `bundle lock --update rails <test gems>`).
3. Read what the re-resolve moved. Fix transitive major jumps (e.g. a pool or HTTP gem going keyword-only) in the gem before consumers lift their pin.

---

## Major bumps of a gem your code subclasses

When the only compatible version is a new major and the app subclasses its classes:

1. **Diff the API surface.** Install both versions to a temp dir; diff `public_instance_methods` and `private_instance_methods` (minus `Object`'s) on the base classes and value objects the app uses. Removed methods are call sites to fix. Added private methods can be silently overridden by a subclass method of the same name; grep the app for each.
2. **Reproduce each break in a short script against both versions** to see which raise and which change behavior silently.
3. **Resolve the enclosing class for every hit** (walk the AST); name-only scans over-count.
4. **Check what the gem now does with caller values.** Grep the new version for `== nil`, `==`, `===`, `to_s`, `present?` on inputs. `value == nil` calls the value's `#==`: an `AssociationRelation` or `CollectionProxy` loads every row, and an app `#==` that assumes the same class can raise. Also check nil handling of required inputs and the shape of error output that reaches users or APIs.
5. **Ship both-version changes as slices ahead of the bump**; keep one-version changes in the bump PR.
6. **Add a spec for each silent failure mode.** A prepend patch for gem behavior needs a version check that warns on any gem version change and a spec that fails without it.
