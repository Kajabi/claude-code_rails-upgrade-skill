# Proving a Mechanical Refactor Is Behavior-Neutral

Many fix-before-bump changes are mechanical rewrites of an API whose runtime behavior should not change: a declaration syntax that is removed at the target version, a method renamed onto another object, an option renamed. Specs are necessary but not sufficient for these. A spec suite exercises the paths someone thought to test, not every value of every declaration. This reference describes a stronger check. Use it whenever a refactor touches many call sites, or touches money, auth or data at rest.

## When to use it

- The change is a rewrite of the *declaration* of something whose runtime artifacts Rails generates: enums, associations, attribute types, callbacks, scopes, delegations.
- The change spans many files and is split into several PRs.
- A mistake would be silent: wrong data written, a method missing only on a rare path. A mistake that raises at boot doesn't need this.

## 1. Before/after artifact dump

Write a small script that loads each affected class and dumps everything Rails generated from the declaration. Run it on the PR's merge-base and on its head, then diff the two outputs. Identical output means identical behavior for everything the dump covers.

For a declaration like `enum`, dump:

- the value map (`defined_enums`, keys and stored values, in order)
- the generated instance methods (predicates and bang methods, including prefixed and suffixed names)
- the generated scopes (positive and negative)
- the attribute type, subtype and default
- for **every** value, a round trip:
  - assign it through the setter
  - read it back
  - record the raw stored value and what gets serialized to the database
  - record the SQL that `where(attr => value)` generates

Resolve the affected classes from the changed files. Handle files that aren't autoloaded (`lib/`, separately loaded engines) by requiring them. Handle modules and concerns by dumping every class that includes them. Make the script robust to records that raise when built bare (callbacks that assume associations): rescue per value and record the failure instead of aborting. A dump that errors on both sides produces two identical *empty* results. Fail loudly when a dump file is missing, rather than reporting "identical".

## 2. Mutation test the check itself

Before trusting "identical", plant one deliberate mistake (drop an option, rename a key) and confirm the diff reports it. This proves the dump covers the thing that could go wrong.

## 3. Exercise it through the running app

Run a local server on a branch that combines all the refactor PRs, with any "stop allowing the old form" gate enabled, so a missed site raises. Seed demo data and exercise a representative set of pages that read and write the affected declarations. Check the stored values in the local database after each write.

## 4. Verify stored values in production after release

After each release reaches 100% of production, query the tables it touches. For each rows-written-since-release window, group by the stored value and check it against the allowed set. Focus on declarations whose key differs from the stored value (e.g. key `in_progress` stored as `"in-progress"`). That is the only mistake that writes wrong data without raising. Read from a replica, and bound each scan to rows written after the release.

Ship in slices ordered by risk. Put anything with unusual load paths (engines, `lib/`, values built at load time from a file) in its own release, so a problem is easy to attribute.

