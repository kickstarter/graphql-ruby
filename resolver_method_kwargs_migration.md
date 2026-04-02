# `field` resolution kwargs across `1_8_14`, `1_8_15`, `1_10_14`

This note describes exactly how `method:`, `hash_key:`, and `resolver_method:` behave in each branch.

---

## TL;DR

- `1_8_14`: no `resolver_method:` keyword exists.
- `1_8_15`: `resolver_method:` is accepted as a **forward-compatible alias** for old `method:` behavior.
- `1_10_14`: `resolver_method:` and `method:`/`hash_key:` are separated conceptually:
  - `resolver_method:` targets method lookup on the GraphQL receiver (type/resolver object)
  - `method:` / `hash_key:` target fallback lookup on the underlying application object/hash.

---

## 1) Branch `1_8_14`

### Accepted kwargs
- `method:`
- `hash_key:`
- (`resolver_method:` is **not accepted**)

### Validation
- `method:` + `hash_key:` => raises `ArgumentError`

### Internal wiring
`GraphQL::Schema::Field#initialize` computes one shared lookup name:

```ruby
method_name = method || hash_key || underscore(field_name)
@method_sym = method_name.to_sym
@method_str = method_name.to_s
```

### Resolution behavior
A single name (`@method_sym` / `@method_str`) is reused in multiple places:

1. first-stage dispatch on the field receiver via `public_send(@method_sym)`
2. fallback `resolve_field_method`:
   - if underlying `object` is a `Hash`: key lookup (`@method_sym`, else `@method_str`)
   - else if underlying `object` responds to `@method_sym`: call method
   - else raise

### Consequence
In `1_8_14`, `method:` is effectively overloaded: it influences both receiver-method dispatch and underlying-object/hash fallback.

---

## 2) Branch `1_8_15` (this branch patch)

### Accepted kwargs
- `method:`
- `hash_key:`
- `resolver_method:` (**new, compatibility alias**)

### Validation
- `method:` + `hash_key:` => raises `ArgumentError` (unchanged)
- `method:` + `resolver_method:` => raises `ArgumentError` (new)
- `hash_key:` + `resolver_method:` => raises `ArgumentError` (new)

### Internal wiring
Still one shared lookup name, now with alias support:

```ruby
method_name = method || resolver_method || hash_key || underscore(field_name)
@method_sym = method_name.to_sym
@method_str = method_name.to_s
```

### Resolution behavior
Same runtime behavior as `1_8_14` (single-name model), because this is a syntax-compatibility patch, not full semantic backport.

### Consequence
`resolver_method:` works on `1_8_15`, but behaves like old `method:` semantics.

---

## 3) Branch `1_10_14`

### Accepted kwargs
- `method:`
- `hash_key:`
- `resolver_method:`

### Validation
- `method:` + `hash_key:` => raises
- `method:` + `resolver_method:` => raises
- `hash_key:` + `resolver_method:` => raises

### Internal wiring
Two independent names are tracked:

```ruby
method_name     = method || hash_key || underscore(field_name)
resolver_method = resolver_method || underscore(field_name).to_sym

@method_sym      = method_name.to_sym
@method_str      = method_name.to_s
@resolver_method = resolver_method
```

### Resolution behavior
Two-stage dispatch:

1. Try receiver method:
   - if receiver responds to `@resolver_method`, call it
2. Else fallback (`resolve_field_method`) using `@method_sym`/`@method_str`:
   - hash key lookup if underlying object is `Hash`
   - else underlying object method lookup
   - else raise

### Consequence
- `resolver_method:` controls receiver-method name.
- `method:`/`hash_key:` control fallback name for underlying object/hash.
- This is a real split of intent compared to 1.8.

---

## Practical migration guidance for codemod/RuboCop

When rewriting old `method:` usage:

- Use `resolver_method:` when the intent is a resolver/type method on the GraphQL class.
- Keep `method:` when the intent is underlying application object method lookup.
- Keep `hash_key:` for explicit hash-key reads.

### Important caveat for static analysis
A same-file method-definition heuristic is useful, but incomplete. Ambiguous cases include:
- inherited methods
- included modules/concerns
- metaprogrammed methods (`define_method`)
- resolver classes (`resolver:`) with methods defined elsewhere

A robust migration should:
1. auto-fix high-confidence cases,
2. emit/flag ambiguous cases for manual review,
3. run runtime/tests to validate behavior before and after upgrade.
