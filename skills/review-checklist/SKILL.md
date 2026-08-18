---
name: review-checklist
description: Language- and stack-specific review checklists (PHP, Symfony, Laravel, TypeScript/React, SQL/migrations, shell). Use when reviewing code and you need to know which failure modes are worth checking for that specific stack, or when the user asks what to look for in a review.
---

# Review checklist

Load the section matching the changed files. Skip the rest.

## Every stack

- Error paths: every `catch`/`if err` either handles or rethrows — never swallows silently.
- Boundary values: empty collection, single element, null, zero, negative, max length.
- Concurrency: two requests hitting this path at once — is any shared state mutated without a lock or a transaction?
- Secrets: no key, token, or password literal. Check test fixtures too.
- Deleted code: was the caller updated, or does something still reference it?

## PHP

- Type juggling: loose `==`/`switch` on user input, or `in_array`/`array_search` without the strict flag.
- Error suppression: `@` hiding a real failure instead of handling it.
- Superglobals: `$_GET`/`$_POST`/`$_REQUEST` read directly instead of through validated/typed input.
- Deserialization: `unserialize()` on user-controlled data (object injection) instead of `json_decode`.
- Dynamic include/require path built from user input (LFI/RFI).
- Autoload/namespace mismatch after a file move or rename — check `composer dump-autoload` isn't papering over it.

## Symfony

- Doctrine N+1: an entity relation (`OneToMany`/`ManyToOne`) accessed in a loop with no join fetch (DQL `JOIN`, `fetch: EAGER`, or explicit batch load).
- Authorization: new controller action/route missing `#[IsGranted]`, a `security.yaml` `access_control` entry, or a Voter check.
- Form binding: a Symfony Form mapped straight onto an entity with no DTO or explicit allowed-fields list, or `csrf_protection` disabled.
- Validation: entity persisted/flushed before Validator constraints run.
- Doctrine migrations: generated `down()` not hand-checked to be the true inverse of `up()`.
- Query builder: user input concatenated into DQL instead of bound via `setParameter()`.
- Services: new service without correct autowiring/visibility, or constructor injection forming a circular dependency.
- Messenger: handler does non-idempotent work with no retry/dedup guard.

## Laravel

- Mass assignment: `$model->fill($request->all())` without `$fillable`/`$guarded` covering the new columns.
- N+1: a relation accessed inside a loop with no `with()` eager load.
- Authorization: new controller action or route with no policy, gate, or middleware.
- Validation: request data used before a `validate()`/FormRequest call.
- Query builder: user input concatenated into `DB::raw()` or `whereRaw()` instead of bound.
- Migrations: `down()` missing or not the true inverse of `up()`.
- Queues: job payload holds a full model (serialization drift) instead of an id.

## TypeScript / JavaScript

- `any` or a cast (`as X`, `!`) that hides a real nullable — trace where the value comes from.
- `await` missing on a promise-returning call; floating promise with no `.catch`.
- Equality: `==` where the operands can differ in type.
- Array mutation (`sort`, `reverse`, `splice`) on a value the caller still holds.
- Optional chaining masking a bug: `a?.b()` where `a` being undefined is itself the error.

## React

- Hook dependency array missing a value the effect reads (stale closure) or including an object recreated each render (loop).
- State derived from props in `useState` — never updates when the prop changes.
- Key on a list item that is an index while the list can reorder.
- Effect with no cleanup for a subscription, timer, or listener.
- `useMemo`/`useCallback` added with no measured cost — noise, not optimization.

## SQL / migrations

- New column on a large table: nullable or with a default, or the migration locks writes.
- Added index — does it duplicate an existing one by leading-column prefix?
- Foreign key without the matching `ON DELETE` behavior decided.
- Data backfill inside a schema migration with no batching.
- Migration not idempotent and not safe to re-run after a partial failure.

## Shell / CI

- Missing `set -euo pipefail`.
- Unquoted `$var` where the value can contain a space or be empty.
- `rm -rf "$dir/"` where `$dir` can be empty.
- Secret echoed into logs, or passed as a CLI arg (visible in `ps`).
