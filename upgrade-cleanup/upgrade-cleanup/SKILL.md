---
name: upgrade-cleanup
description: Clean up after (or abandon) a Rails upgrade. Drop `if ENV["DEPENDENCIES_NEXT"]` branches and retire bootboot dual-boot scaffolding (Gemfile_next.lock, the bootboot plugin loader, the `enable_dual_booting` block, and conditional Gemfile gem blocks), keeping either the next or the current version. Trigger when the user says they are done with the upgrade, want to clean up dual-boot, want to drop the DEPENDENCIES_NEXT branches, want to finish the upgrade, want to abandon or revert the upgrade attempt, want to roll back to the current Rails version, or want to pause this upgrade hop. Based on FastRuby.io's "Finishing an Upgrade" methodology, extended with an abandon/pause path and adapted for the bootboot Bundler plugin.
---

# Upgrade Cleanup Skill

Companion to the `rails-upgrade` plugin. Runs the cleanup pass that removes dual-boot scaffolding and aligns the codebase to the new version baseline.

When activated, follow the workflow in `workflows/upgrade-cleanup-workflow.md` end-to-end. **Before any destructive step, confirm direction with the user** (Phase 0 Step 1): are they keeping the **next** version (finishing the upgrade) or keeping the **current** version (abandoning or pausing this hop)? Every subsequent step branches on that answer. To detect the next version, read the `Gemfile` (look for the `if ENV["DEPENDENCIES_NEXT"]` / `else` block; the truthy branch holds the upgraded-to version) or `Gemfile_next.lock`. Do NOT rely on `Gemfile.lock` alone, since during dual-boot it still pins the current version.

## When to Run

Run when the user has explicitly decided to **end the dual-boot phase** in one of two directions:

- **Keep next**: upgrade is done (final hop or stopping point), drop the `else` / current branches.
- **Keep current**: abandoning or pausing this hop, drop the `if ENV["DEPENDENCIES_NEXT"]` / next branches and `Gemfile_next.lock`.

Either way the previous parallel branch must no longer be needed (no rollback window). Deployment to production is not a hard prerequisite.

## Ownership and Delegations

This skill **owns** the cleanup. Phase 1 below is the step list to follow.

- **Dual-boot scaffolding removal**: performed here in Phase 1.
- **`load_defaults` alignment**: out of scope. The `rails-upgrade` skill handles this via its `rails-load-defaults` step before cleanup runs.

## Critical Rules

- **Do NOT leave `if ENV["DEPENDENCIES_NEXT"]` branches (or app-specific wrappers like `AppConfig.dependencies_next?`) in the tree.** That is the failure mode this skill exists to prevent.
- **Do NOT start removing branches before confirming direction.** Keeping the wrong side throws away the work the user wants to keep. If the user has not stated next vs current, ask.

## Workflow

See `workflows/upgrade-cleanup-workflow.md` for the full process: a pre-flight check, dual-boot scaffolding removal, old-version code retirement, CI and Ruby pin alignment, final verification, and the cleanup PR.

## Reference

- [Finishing an Upgrade, FastRuby.io](https://www.fastruby.io/blog/finishing-an-upgrade.html) (next_rails-flavored, the principles translate directly to bootboot)
- [bootboot](https://github.com/Shopify/bootboot), the Bundler plugin this skill assumes
- `workflows/upgrade-cleanup-workflow.md`, full workflow
