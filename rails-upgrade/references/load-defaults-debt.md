# Accumulated `load_defaults` Debt

Use when `config.load_defaults` lags the installed Rails by more than one version. Check at Step 0.

## Measure

1. Read `config.load_defaults` in `config/application.rb`.
2. List `config/initializers/new_framework_defaults_*.rb`; each file is one un-flipped version.
3. Count active vs commented flags per file on the base branch (not from a ticket).
4. Drop flags that are no-ops here: removed by a later Rails (diff against `load_defaults` in the installed `railties`), already set elsewhere, or for frameworks the app doesn't load.
5. Report `<n> files / <m> flags`.

When verifying a flipped flag, read its **effective** value (e.g. `bin/rails runner 'p ActiveRecord::Base.automatic_scope_inversing'`). `ActiveRecord::Base`-level flags set in `new_framework_defaults_*.rb` are ignored if anything loads `ActiveRecord::Base` earlier; set those in `config/application.rb`.

## Sequence

Debt does not block the version bump. Finish any part-flipped file before the hop; untouched files can trail it.

## Flags to ship one at a time

- `cookies_serializer`: two-phase `:hybrid` migration, or sessions break on deploy.
- `key_generator_hash_digest_class` / `hash_digest_class`: signed cookies and cache entries invalidate. First pin the current digest on any bare `ActiveSupport::KeyGenerator.new(secret)` used for data at rest, with a spec that encrypts, switches the global, then decrypts.
- `raise_on_open_redirects`: latent redirect bugs become 500s.
- `active_record.partial_inserts`: INSERT shape changes for every model.
- `active_storage.variant_processor`: image backend changes.
- `automatic_scope_inversing`: see below.

## `automatic_scope_inversing`

Dump every scoped association's inferred inverse with the flag off and on; the diff is the blast radius:

```ruby
Rails.application.eager_load!
ActiveRecord::Base.descendants.each do |k|
  k.reflect_on_all_associations.each do |r|
    next if r.scope.nil?
    puts "#{k}##{r.name}|#{r.macro}|#{r.inverse_of&.name.inspect}"
  end
end
```

For each changed association, check:
- **Stale cached collections:** assigning the parent on an unsaved child now loads and caches the parent's scoped collection; later reads see the cache.
- **Sibling ambiguity:** several parent associations sharing one child and foreign key; back-population picks one.

Verify any `inverse_of: false` pin with the flag on. If newly active inverses break code, leave the global flag off rather than scattering pins and resets.
