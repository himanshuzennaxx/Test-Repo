# ServiceOps — Add `internal_lang` field (addition to MVP tasks)

**To:** India Developer
**Date:** 2026-07-30
**Context:** Follow-up to SERVICEOPS_MVP_TASKS_for_India.md — one more META field.

## Why

We have two distinct audiences with different languages:

- **Customers** read the Issue-side replies → controlled by `reply_lang` (already exists).
- **The team (including you and other developers)** read the PR-side internal output — the RFC, PM scoring, reviews. These are in English so everyone can read them.

Right now these two are tangled. We're separating them with a second field.

## What to add

Add **`internal_lang`** to the META block, alongside `reply_lang`:

```
<!-- SERVICEOPS:META -->
ticket_id: TK-8
reply_lang: en
internal_lang: en
...
<!-- END SERVICEOPS:META -->
```

- **`reply_lang`** = customer's language (Issue-side, customer-facing replies). Unchanged.
- **`internal_lang`** = language for internal output on the PR (RFC, PM scoring, reviews).

## Default value

**Default `internal_lang` to `en`, set in the database** (a column default is fine). Rationale: PR content is read by the whole team including developers who may not read Chinese, so English is the safe default. A ticket can override it if needed, but the DB default should be `en`.

## Key point: the two are independent

A customer writing in Chinese (`reply_lang: zh-TW`) must NOT force the RFC into Chinese — the developer still needs English. That's the whole reason for a separate field. So:

- Customer-facing replies follow `reply_lang`
- Internal PR output follows `internal_lang` (default en)
- If an internal staff member wants to write one comment in another language, that individual comment carries its own `lang` — only then does the PR show mixed languages.

## Scope

Just add the field + DB default. The automation side (scan.sh) already reads `internal_lang` and applies it to RFC/PM/review output — that part is done on our end. We just need ServiceOps to emit the field.
