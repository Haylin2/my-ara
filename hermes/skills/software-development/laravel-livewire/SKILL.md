---
name: laravel-livewire
description: "Laravel 13 + Livewire 4 conventions, pitfalls, testing."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [laravel, livewire, php, pest, testing]
    category: software-development
---

# Laravel 13 + Livewire 4 Development

## When to Use

Use this skill when writing PHP/Laravel code, Livewire components, Pest tests,
API Resource transformers, or E2E tests for a Laravel 13 + Livewire 4 project.

Standing conventions and pitfalls for Laravel/Livewire projects. Applies to
PHP code changes, test writing, and API resource transformers.

## Always-On Rules

1. **Run Pint before commit.** `vendor/bin/pint --dirty` is enforced in CI.
2. **Clear config+route cache before running tests.** Stale `routes-v7.php`
   causes Livewire endpoint-hash mismatch — tests silently return 404 on
   `->set()`/`->call()`.
3. **Use `assertDatabaseHas` with all relevant fields.** Don't just assert on
   `title` — include `user_id`, `unit_id`, or any field the code was supposed
   to set. Partial assertions miss bugs.
4. **Test Resource transformers directly.** Don't rely solely on controller
   tests to cover API response shape — instantiate the Resource class and
   call `toArray(new Request())` to assert exact field contracts.
5. **E2E tests go in `tests/e2e/<feature>/`** with `.spec.ts` extension.
   Import from `../shared/fixtures` for `login`, `waitForLivewire`, etc.

## Pitfalls

- **`whenLoaded` returns `MissingValue`, not `null`.** When calling
  `toArray()` directly on a JsonResource (not through `toResponse()`),
  `whenLoaded('relation')` returns a `MissingValue` object. Tests must
  check `instanceof` or use `array_key_exists` — `assertNull()` fails.
- **Factory defaults ≠ nullable columns.** A migration with
  `->default('o-bell')` on a NOT NULL column means the factory must NOT
  pass `null` for that field. Use the default value, not `null`.
- **Faker `passthrough()` requires an argument.** Use
  `fake()->optional(0.6)->words(3)` or another generator for optional
  JSON fields — `passthrough()` needs a value parameter.
- **Carbon `toISOString()` ≠ ATOM format.** `toISOString()` returns
  `.000000Z` suffix; ATOM uses timezone offset. Use `strtotime()` for
  flexible ISO 8601 validation in tests.
- **UUID primary key models + HasFactory.** Models with manual UUID
  generation in `boot()` (via `Str::uuid()`) work with `HasFactory`.
  Don't add `HasUuids` trait — it would conflict with the manual boot
  logic.

## Testing Patterns

See `references/testing-pitfalls.md` for the full decision table on
assertion patterns, factory creation, and E2E test structure.