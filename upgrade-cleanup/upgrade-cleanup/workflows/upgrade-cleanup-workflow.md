# Upgrade Cleanup Workflow

Run when the user is done with the upgrade campaign and wants to remove dual-boot scaffolding. Aligns the codebase to the new version baseline.

Based on FastRuby.io's [Finishing an Upgrade](https://www.fastruby.io/blog/finishing-an-upgrade.html) methodology and adapted for the [bootboot](https://github.com/Shopify/bootboot) Bundler plugin. Rule of thumb: **never start the next hop with `if ENV["DEPENDENCIES_NEXT"]` branches still in the tree.** They accumulate, lose context, and make the next dual-boot impossible to reason about.

---

## When to Run This

Run when the user has explicitly decided to **end the dual-boot phase** in one of two directions:

- **Keep next** — upgrade is done (final hop or stopping point). Drop the `else` / current branches.
- **Keep current** — abandoning or pausing this hop. Drop the `if ENV["DEPENDENCIES_NEXT"]` / next branches and `Gemfile_next.lock`.

The other side (whichever is being dropped) must no longer be needed (no rollback window, no parallel branch). Deployment to production is not required. If the user wants to clean up before deploying, that is their decision.

---

## Prerequisites

- Test suite passes on the side the user is keeping
- Working tree is clean (or on a dedicated cleanup branch)
- User confirms direction (next vs current) and, if keeping next, target version (e.g. "we just upgraded to 7.1")

---

## Phase 0: Pre-flight

Tearing down dual-boot scaffolding is destructive and direction-dependent. Settle direction first, then validate the side that is being kept.

### Step 1: Confirm direction

Before any destructive step, ask the user which side they want to keep. Do not infer it.

> *"Are we cleaning up because the upgrade succeeded and you want to keep the **next** version, or because you're abandoning/pausing this hop and want to keep the **current** version?"*

Record the answer as **keep next** or **keep current**. Every later step in this workflow branches on that label — Phase 1 onward spells out what each path does. If the user is unsure, stop and clarify. Don't guess from repo signals, both sides may look healthy.

### Step 2: Detect how the app runs

Inspect the repo to infer the run environment, then **ask the user** if anything is ambiguous. Do not guess.

| Signal | Implication |
|---|---|
| `Dockerfile` + `docker-compose.yml` (or `compose.yaml`) with a Rails service | App likely runs in Docker. Smoke checks should run via `docker compose run --rm <service> <cmd>`. |
| `bin/dev` or `Procfile.dev` | Local dev with foreman/overmind. Bundler runs locally. |
| `.tool-versions` / `.ruby-version` matches local Ruby, no Docker rails service | Local. Run commands directly. |
| Multiple options present (e.g. Dockerfile for prod, local for dev) | ASK the user which path to validate against. |

If unsure after inspection, ask: *"Should I run the pre-flight checks via Docker (`docker compose run --rm <svc> ...`) or locally (`bundle ...` directly)?"* Do not pick one silently.

### Step 3: Smoke-check the side being kept

Only validate the side that survives cleanup. The side being dropped is about to be deleted, so its health doesn't matter.

The commands below are shown for a local Ruby setup. If Step 2 settled on Docker, prefix each command with the appropriate container runner (e.g. `docker compose run --rm web ...`). If Step 2 was inconclusive and the user said "just try", run local first and fall back to Docker only if local can't resolve gems or boot the app.

**If keeping next:**

1. Next side bundles:
   ```sh
   DEPENDENCIES_NEXT=1 bundle check \
     || DEPENDENCIES_NEXT=1 bundle install
   ```
2. App boots on the next side (catches initializer / autoload / gem-API regressions that bundle alone misses). Skip only if a database is genuinely unreachable:
   ```sh
   DEPENDENCIES_NEXT=1 bin/rails runner "puts Rails.version"
   ```
   Output should match the upgraded-to version.

**If keeping current:**

1. Current side bundles:
   ```sh
   bundle check || bundle install
   ```
2. App boots on the current side:
   ```sh
   bin/rails runner "puts Rails.version"
   ```

### Stop conditions

- **Kept-side bundle fails:** the side the user wants to keep is broken. Cleanup is premature. Stop and tell the user; resolve the environment or boot regression before tearing down the parallel branch.
- **Kept-side rails runner fails but bundle succeeds:** boot regression on the version being kept. Tell the user; don't tear down the rollback path until it's fixed.

If the user explicitly says *"skip the pre-flight, I know it works"*, record their override and continue. Their call.

---

## Phase 1: Remove `DEPENDENCIES_NEXT` Branches and Bootboot Scaffolding

Each step indicates the action for **keep next** vs **keep current**, per Phase 0 Step 1. The two paths are symmetric: whichever branch is kept becomes unconditional code.

1. Find all `ENV["DEPENDENCIES_NEXT"]` references in application code (Ruby and Gemfile):
   ```sh
   grep -rnE 'ENV\["DEPENDENCIES_NEXT"\]|ENV\[.DEPENDENCIES_NEXT.\]' . \
     --include="*.rb" --include="Gemfile" --include="Rakefile" -l
   ```
   Also search for any project-specific wrapper helpers that read the same env var (kajabi-products style: `AppConfig.dependencies_next?`, sometimes a private predicate method in the Gemfile):
   ```sh
   grep -rnE 'dependencies_next\??|DependenciesNext' . --include="*.rb" -l
   ```
   The wrapper is the one Phase 2 has to retire alongside the branches it gated.
2. Collapse the conditionals based on direction:
   - **Keep next:** keep the `if ENV["DEPENDENCIES_NEXT"]` true branch, drop the `else`. Ternaries collapse to their truthy value (e.g. `config.load_defaults ENV["DEPENDENCIES_NEXT"] ? 8.0 : 7.0` → `config.load_defaults 8.0`).
   - **Keep current:** drop the `if ENV["DEPENDENCIES_NEXT"]` block entirely (keep what was in the `else`). Ternaries collapse to their falsy value (e.g. `config.load_defaults ENV["DEPENDENCIES_NEXT"] ? 8.0 : 7.0` → `config.load_defaults 7.0`).
3. If the project defined a wrapper predicate (e.g. `AppConfig.dependencies_next?`) and no other code still calls it, remove the definition. If it has callers outside the upgrade scope, leave the definition and only collapse the upgrade-related call sites.
4. Collapse `if ENV["DEPENDENCIES_NEXT"]` / `else` conditionals in the `Gemfile` (remove the wrapper, keep one side's gems unconditional):
   - **Keep next:** keep the next-version gems (the truthy branch).
   - **Keep current:** keep the current-version gems (the `else` branch).
5. Remove the bootboot plugin loader and the `enable_dual_booting` activation from the `Gemfile`. The block to delete typically looks like:
   ```ruby
   plugin "bootboot", "~> 0.2.2" if !ENV["RAILS_ENV"] || ENV["RAILS_ENV"] == "test" || ENV["ENABLE_BOOTBOOT"]
   Plugin.send(:load_plugin, "bootboot") if Plugin.installed?("bootboot")

   if ENV["DEPENDENCIES_NEXT"] == "1"
     enable_dual_booting if Plugin.installed?("bootboot")
   end
   ```
   If the user wants to keep bootboot installed for the next hop, leave the plugin loader and only drop the `enable_dual_booting` block plus the version-conditional gem groups. Tell them.
6. Reconcile lockfiles based on direction. Either path leaves `Gemfile_next.lock` gone from the tree:
   - **Keep next:** replace `Gemfile.lock` with `Gemfile_next.lock` (`mv Gemfile_next.lock Gemfile.lock`, which also removes `Gemfile_next.lock` from its old path). This preserves the exact gem versions tested during the upgrade. Do not run `bundle update` or delete the lockfile to re-resolve from scratch, that risks drift on versions you already validated.
   - **Keep current:** delete `Gemfile_next.lock`. Leave `Gemfile.lock` alone, it already pins the current version.
7. Run `bundle install` (NOT `bundle update`). `bundle install` is incremental: it removes references to gems no longer in the `Gemfile` (such as any dual-boot-only gems) without re-resolving the rest. Rails and friends stay pinned. If you removed the bootboot plugin line, also clear the cached plugin so subsequent bundles don't try to load it:
   ```sh
   bundle plugin uninstall bootboot 2>/dev/null || true
   ```
8. Run the project's test suite (detect the runner from the `Gemfile`: `rspec-rails` means `bundle exec rspec`, otherwise `bundle exec rake test` or `bin/rails test`).
9. Update CI to drop the dual-boot job/matrix entry (the one that ran `DEPENDENCIES_NEXT=1`).

**Sanity check (keep next only):** after the lockfile swap, run `git diff Gemfile.lock` to confirm the new Rails version is actually pinned. If `Gemfile_next.lock` was never regenerated during the upgrade, it can be byte-identical to the old `Gemfile.lock`. In that case the lock is stale. Flag it to the user and run `bundle install` to resolve.

---

## Phase 2: Retire Stale Version-Conditional Code

Beyond the `DEPENDENCIES_NEXT` branches, hunt for version-conditional code that no longer applies. Direction matters: when keeping next, the *current*-version scaffolding is dead; when keeping current, the *next*-version scaffolding is dead.

1. **Temporary monkey-patches and backports.** Search `config/initializers/` for files named like `rails_X_Y_backport.rb`, `monkey_patches/`, or comments tied to a specific version.
   - **Keep next:** delete patches that targeted the current (old) Rails version, they're now unconditional.
   - **Keep current:** delete patches that were added for the next version (no longer reachable). Confirm with user before deleting.
2. **Gem version pins.** Run `bundle outdated` and inspect pins.
   - **Keep next:** loosen pins held back for current-version compatibility, the constraint is gone.
   - **Keep current:** revert any pins that were bumped in anticipation of the next version, if they break the current version.
3. **Conditional `Gemfile` groups.** Drop groups keyed off the version being dropped (direction-symmetric: keep next → drop current-version groups; keep current → drop next-version groups).
4. **`docker-compose.yml` / `compose.yaml` sister services.** Dual-boot setups occasionally add a parallel service (e.g. `web-next`, `worker-next`) whose entrypoint sets `DEPENDENCIES_NEXT=1` and reuses the primary service via YAML anchors. These break once dual-boot is gone. Grep both compose files:
   ```sh
   grep -nE "DEPENDENCIES_NEXT|-next:" docker-compose.yml compose.yaml 2>/dev/null
   ```
   - **Keep next:** drop the `*-next` service definitions; if the *primary* service was the one carrying `DEPENDENCIES_NEXT=1`, unset that env var instead of deleting the service.
   - **Keep current:** drop the `*-next` service definitions outright.
5. **Documentation drift.** Sweep `README.md`, `CONTRIBUTING.md`, `bin/setup`, setup scripts, `.tool-versions`, and `Dockerfile` for stale Ruby/Rails references and any leftover mentions of `DEPENDENCIES_NEXT` / `Gemfile_next.lock`.
   - **Keep next:** update to the new baseline.
   - **Keep current:** revert any docs that were updated to the next version prematurely.

---

## Phase 3: CI and Ruby Pin Alignment

Tighten the build/test surface around the kept version:

1. **CI matrix.** Drop the dropped-side Rails version from any test matrix; you're not testing against it anymore. (Keep next → drop the old version. Keep current → drop the new version.) For bootboot setups this usually means removing the matrix entry whose env block sets `DEPENDENCIES_NEXT=1`.
2. **`Dockerfile` / `Gemfile` Ruby pin.**
   - **Keep next:** if the upgrade required a Ruby bump, confirm the Dockerfile, `.tool-versions`, `.ruby-version`, and CI all agree on the new Ruby. Distinguish drift introduced by this upgrade (fix in cleanup) from pre-existing drift that predates the upgrade (flag it for the user, but leave it out of scope, fixing unrelated infra in a cleanup PR muddies the diff).
   - **Keep current:** if a Ruby bump was staged for the next version, revert it back to the current Ruby. Same scope rule, only undo what this upgrade attempt introduced.

---

## Phase 4: Final Verification

Before declaring cleanup done:

- [ ] Test suite passes on the kept version (no dual-boot, single lockfile)
- [ ] CI is green on the cleanup branch
- [ ] `grep -rnE 'ENV\["DEPENDENCIES_NEXT"\]|dependencies_next\?' . --include="*.rb" --include="Gemfile"` returns nothing (apart from the bootboot plugin definition if you intentionally kept it for the next hop)
- [ ] `Gemfile_next.lock` is gone
- [ ] No leftover `plugin "bootboot"` / `enable_dual_booting` in `Gemfile` (unless explicitly kept for the next hop)
- [ ] Documentation reflects the kept Ruby/Rails versions (keep next → new baseline; keep current → unchanged from before the upgrade attempt)

If the local environment cannot run the test suite (no DB, sandboxed shell), CI on the cleanup branch is the validating environment. Commit and push the cleanup PR, then track Phase 4 as in-progress until CI is green. Do not block the commit/PR step on a local test run that cannot happen.

---

## Phase 5: Commit and Open the PR

A dedicated cleanup PR is the recommended default. The diff reads as "remove scaffolding," nothing else, which makes review fast. If the user prefers to fold it into another branch, that is their call.

Suggested commit messages (pick by direction):

- **Keep next:**
  - `Remove dual-boot setup after Rails X.Y upgrade`
  - `Drop DEPENDENCIES_NEXT branches and bootboot scaffolding`
- **Keep current:** (`X.Y` is the abandoned target version, `A.B` is the version we stay on)
  - `Abandon Rails X.Y upgrade attempt; stay on Rails A.B`
  - `Drop DEPENDENCIES_NEXT branches and bootboot scaffolding`

Keep them as separate commits so reviewers can see each cleanup pass in isolation.

---

## What This Workflow Does NOT Do

- It does not roll back the upgrade. There is no rollback path here.
- It does not start the next version hop. After cleanup, the user invokes the rails-upgrade skill again for the next version.
- It does not triage deprecation warnings. Those belong to the rails-upgrade skill's next-hop workflow.

---

## Notes for Claude

- **Direction first, always.** Never start collapsing branches before Phase 0 Step 1 has settled next vs current. Both sides may look healthy in the repo; only the user knows which one is the goal.
- Treat every destructive step (`rm`, lockfile replacement, plugin removal, CI edits) as needing user confirmation. The workflow is reversible only via git, so move slowly.
- If `Gemfile_next.lock` does not exist, dual-boot was never set up or is already cleaned. Skip Phase 1 and tell the user.
- If `ENV["DEPENDENCIES_NEXT"]` references appear inside vendored gems or `vendor/bundle/`, ignore them. Only application code matters.
- When keeping next, detect the target version from the `Gemfile`'s `if ENV["DEPENDENCIES_NEXT"]` block or `Gemfile_next.lock`, not `Gemfile.lock`, which still holds the current version during dual-boot.
- If the user wants to keep bootboot installed for the next hop, skip the plugin-line removal in Phase 1 Step 5 but still delete `Gemfile_next.lock` and collapse the `if ENV["DEPENDENCIES_NEXT"]` conditionals.
- **If the repo was previously on next_rails and migrated to bootboot mid-upgrade**, residue may include `Gemfile.next`, `Gemfile.next.lock`, a Gemfile-local `next?` method, and stray `NextRails.next?` / `NextRails.current?` references. Run the same grep with the older patterns once (`grep -rE "NextRails\.(next|current)\?" . --include="*.rb" -l`) and sweep them as part of the same cleanup pass. The next_rails gem also shipped a `DeprecationTracker` library; if `grep -rnE "deprecation_tracker|DeprecationTracker|DEPRECATION_TRACKER" spec/ test/` returns hits, tear those down too (drop the `require`s, the `DeprecationTracker.track_rspec(...)` call, and any `spec/support/deprecation_warning.shitlist.json`). Removing the next_rails gem without sweeping the tracker first breaks test boot with `LoadError: cannot load such file -- deprecation_tracker`.
