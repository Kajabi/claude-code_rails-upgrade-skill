# Proving a Mechanical Refactor Is Behavior-Neutral

Use for fix-before-bump rewrites that should not change behavior (removed declaration syntax, renamed methods or options) when they span many call sites or touch money, auth or data at rest. Skip it when a mistake would raise at boot.

1. **Dump before and after.** Write a script that loads each affected class and dumps everything Rails generates from the declaration. For `enum`: the value map in order, generated methods and scopes, attribute type and default, and for every value a setter → getter → stored value → `where(attr => value)` SQL round trip. Run it on the PR's merge-base and head; diff. Require non-autoloaded files and dump every class including affected concerns. Rescue per value and record failures; fail loudly if a dump file is missing.
2. **Mutation-test the dump.** Plant one deliberate mistake (drop an option, rename a key) and confirm the diff reports it.
3. **Exercise it locally.** On a branch combining all the refactor PRs, with any "stop allowing the old form" gate enabled, seed data and use pages that read and write the affected declarations. Check stored values after each write.
4. **Check stored values in production after each release.** Group rows written since the release by stored value and compare against the allowed set, prioritizing declarations whose key differs from the stored value (e.g. key `in_progress` stored as `"in-progress"`). Read from a replica, bounded to post-release rows.

Ship slices ordered by risk; give unusual load paths (engines, `lib/`, values built at load time) their own release.
