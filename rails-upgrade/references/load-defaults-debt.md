# Accumulated `load_defaults` Debt

An app can sit on Rails X while `config.load_defaults` still reads X-2 or X-3, because earlier hops shipped the gem bump without finishing the defaults alignment. Check for this at Step 0: it changes how much pre-work the hop carries.

## Measure it

1. Read `config.load_defaults` from `config/application.rb`.
2. List `config/initializers/new_framework_defaults_*.rb`. Each file is one un-flipped version.
3. Per file, count active vs commented flags on the base branch. Don't trust a ticket's counts; they go stale as flags land one at a time.
4. Report the total as `<n> files / <m> flags`.

Before counting a flag as debt, check that it does something in this app:

- Diff the commented flags against `load_defaults` in the installed Rails (`railties-X/lib/rails/application/configuration.rb`). Flags a later version removed are no-ops, as are flags already set elsewhere and flags for framework sections the app doesn't load.
- Verify a flipped flag's **effective** value, e.g. `bin/rails runner 'p ActiveRecord::Base.automatic_scope_inversing'`. `ActiveRecord::Base`-level settings are copied from config only in `on_load(:active_record)`. If an earlier initializer or gem loads `ActiveRecord::Base`, a flag set in `new_framework_defaults_*.rb` is silently ignored and only takes effect at the `load_defaults` bump. Set Base-level flags in `config/application.rb`.

## Sequencing

Flags are independently shippable and revertable and don't depend on the gem version, so accumulated debt does not block the next version bump. Finish any file that is already part-flipped before the hop; untouched files can trail it.

## Flags that need more than "flip and run specs"

Give each its own change; a green suite doesn't prove them safe:

- `cookies_serializer`: needs the two-phase `:hybrid` migration in production, or live sessions are invalidated on deploy.
- `key_generator_hash_digest_class` / `hash_digest_class`: changes derived keys, so signed cookies and cache entries invalidate at deploy. Bare `ActiveSupport::KeyGenerator.new(secret)` calls used for data at rest follow the global default too, and rotation doesn't cover stored ciphertext. Pin the current digest at those call sites first, with a spec that encrypts, switches the global, then decrypts.
- `raise_on_open_redirects`: turns latent redirect bugs into 500s.
- `active_record.partial_inserts`: changes the INSERT shape for every model.
- `active_storage.variant_processor`: swaps the image-processing backend.
- `automatic_scope_inversing`: lets Rails use a **scoped** association as an inverse. See below.

## `automatic_scope_inversing`

Boot with the flag off and on, dump every scoped association's inferred inverse, and diff. The diff is the blast radius:

```ruby
Rails.application.eager_load!
ActiveRecord::Base.descendants.each do |k|
  k.reflect_on_all_associations.each do |r|
    next if r.scope.nil?
    puts "#{k}##{r.name}|#{r.macro}|#{r.inverse_of&.name.inspect}"
  end
end
```

Newly inferred inverses are usually structurally correct. The risks are:

- **Stale cached collections.** Assigning the parent on an unsaved child back-populates the inverse, which loads and caches the parent's scoped collection at that moment. Later reads see the cache, not the database. Clone/duplicate flows and counts break silently.
- **Sibling ambiguity.** When several parent associations share one child and foreign key, back-population targets only one of them.

`inverse_of: false` on the parent doesn't reliably prevent this, because the child's `belongs_to` can infer the scoped parent independently; verify any pin with the flag on. Treat the flag as its own change with a per-association verdict. If correct-but-newly-active inverses break code, prefer leaving the global flag off over scattering `inverse_of: false` and collection resets through the app.
