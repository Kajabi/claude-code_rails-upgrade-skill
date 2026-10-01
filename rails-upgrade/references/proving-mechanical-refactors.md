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

After each release reaches 100% of production, query the tables it touches. For each rows-written-since-release window, group by the stored value and check it against the allowed set. Focus on declarations whose key differs from the stored value (e.g. key `offer_coupon` stored as `offer`, or `community_based` stored as `community-based`). That is the only mistake that writes wrong data without raising. Read from a replica, and bound each scan to rows written after the release.

Ship in slices ordered by risk. Put anything with unusual load paths (engines, `lib/`, values built at load time from a file) in its own release, so a problem is easy to attribute.

---

## `automatic_scope_inversing` (Rails 7.0 default): audit before adopting

This flag makes Rails willing to use a **scoped** parent association as the
`inverse_of` for a child `belongs_to` (and vice-versa). It is often treated as a
free, low-risk default. It is not: it changes in-memory association behavior
across every scoped association at once.

### Diagnostic: flag-off vs flag-on inverse delta

Boot the app twice (flag off, then on) and dump every scoped association's
inferred inverse, then diff:

```ruby
Rails.application.eager_load!
ActiveRecord::Base.descendants.each do |k|
  next unless k.respond_to?(:reflect_on_all_associations)
  k.reflect_on_all_associations.each do |r|
    next if r.scope.nil?               # only scoped assocs are affected
    puts "#{k}##{r.name}|#{r.macro}|#{(r.inverse_of&.name).inspect}"
  end
end
```

The diff is the exact blast radius. Set the flag in `config/application.rb`, not
`new_framework_defaults_*.rb`, or `ActiveRecord::Base`-level flags are silently
ignored (see SKILL Step 7).

### Audit every changed inverse, not just the ones a failing spec exercises

For each changed association, resolve the child class and check whether the
inferred inverse's FK genuinely points back (structural correctness), and whether
**sibling** associations on the same parent point to the same child+FK. In a
large monolith almost all inferred inverses are structurally correct (the FK
matches) — the danger is not a wrong reciprocal. The danger is:

1. **Cache pollution.** Assigning the parent on a child (`child.parent = p`, or
   `Fabricate(:child, parent: p)`) now back-populates the inverse, which **loads
   the parent's scoped collection** at that moment and caches it. If the child
   isn't saved yet (or doesn't match the scope), the collection caches empty/stale.
   Later reads of `p.scoped_assoc` (`.present?`, `.size`, `.each`) see the stale
   cache, not the DB. This silently breaks clone/duplicate flows, counts, and any
   "assign then read the collection" code. It does **not** raise.
2. **Sibling ambiguity.** When several parent associations share one child+FK
   (e.g. `has_one :occurrence, -> {…}` + `has_many :occurrences, -> {…}`, or many
   `has_one :media_library_asset_location_*` all with `inverse_of: :location`),
   only one can be the child's real inverse; back-population targets one of them.

Pinning `inverse_of: false` on the obvious has_many does **not** reliably stop the
load — the child's `belongs_to` infers the scoped parent independently, and the
load can come through an indirect path (an `after_create` that assigns the parent,
a factory). Verify any pin with the flag-on repro, don't assume.

**Recommendation:** treat this flag as its own change with a full inverse-delta
audit and a per-association verdict, not part of a bulk defaults flip. If adoption
breaks correct-but-now-active inverses, prefer deferring the global flag (its
benefit is marginal) or enabling `inverse_of:` explicitly per intended association
with a dedicated soak, over scattering `inverse_of: false` / collection `reset`
calls through money paths.
