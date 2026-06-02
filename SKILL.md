---
name: voc-codeowners
version: 1.0.0
description: VOC codeowner tagging guidance for housecall-web. Use when the current git branch starts with VOCR-, when editing files under app/components/voice_of_customer/, or when the user mentions Voice of Customer / VOC / VOCR. Prompts to tag new VOC-owned controllers and sidekiq jobs with `codeowners CodeOwners.voice_of_customer` at the class level, and to scope VOC logic inside another team's existing class with a `CodeOwners.with_codeowners(:voice_of_customer) { ... }` block. NEVER replaces a host team's existing class-level codeowners declaration — that would silently steal on-call ownership of the entire class from the host team. Also covers frontend React/TypeScript: pass `Codeowners.VOICE_OF_CUSTOMER` to Sentry capture helpers (captureException, catchNetworkExceptions) on VOC's own call sites only, never flipping a host team's existing capture argument.
---

# VOC Codeowners

Tag Voice of Customer (VOC) team code paths with the `codeowners:voice_of_customer` attribution used by Sentry alert routing (to `#voc_monitoring`) and Datadog APM. The VOC team ships features *into* other teams' domains (primarily Pipeline/`campaigns` and Online Booking/`online-booking`), so most files VOC edits are owned by another team.

## When to apply

Activate this skill when **any** of the following is true:

- The current git branch name starts with `VOCR-`
- The file being created or edited lives under `app/components/voice_of_customer/`
- The user mentions "Voice of Customer", "VOC", or "VOCR" in the task

If none of these match, do not apply this guidance — defer to whatever codeowner conventions the host team already uses.

**Branch detection (always re-check, don't cache):** before suggesting any codeowner tag, run `git rev-parse --abbrev-ref HEAD` to confirm the current branch. Don't cache the result at session start — the user may switch branches mid-session (e.g., they start on `master`, then `git checkout VOCR-67-Glass`, then switch back), and the skill needs to follow the live state, not a stale snapshot. The branch check is cheap; re-run it whenever you're about to suggest a codeowner change.

## ⚠️ Critical: never overwrite a host team's class-level codeowners declaration

The `CodeOwners` platform supports exactly one codeowner per event (confirmed in `app/utilities/code_owners.rb`: `sentry_scope.set_tags(SENTRY_TAG => team_name)` sets one value). Replacing a host team's class-level `codeowners` declaration with `codeowners :voice_of_customer` would silently route every error in that class to VOC instead of the host team — breaking their on-call routing and Sentry alerts.

**This is never acceptable.** If VOC attribution is needed on specific logic inside a host-team's class, scope it with a `CodeOwners.with_codeowners(:voice_of_customer) { ... }` block instead. The block temporarily swaps the tag during its execution and restores the parent's tag after — non-destructive, additive in effect.

## Detection step (always run first)

Before suggesting any class-level `codeowners` declaration, grep the target file for an existing declaration:

```bash
grep -n "codeowners " <target_file>
```

Use the result to pick the right pattern:

| Detection result | Pattern to apply |
|---|---|
| No `codeowners` declaration anywhere in the file, AND the file is a new VOC-owned controller or Sidekiq worker | **Pattern 1** (class-level) |
| File already has `codeowners :some_other_team` declaration | **Pattern 2** (`with_codeowners` block around the VOC-specific code) |
| File already has `codeowners :voice_of_customer` declaration | Leave it alone, no change needed |
| File is a non-controller / non-Sidekiq class (module, Karafka consumer, service object, model, decorator, etc.) | **Pattern 4** (`with_codeowners` block per public method body) |

## Pattern 1 — Class-level (NEW classes VOC fully owns)

Place the declaration as the first line inside the class body, immediately after `include` statements if present.

**Controller:**

```ruby
module VoiceOfCustomer
  module Controllers
    class CustomerFeedbackController < ::Api::Private::AuthenticatedController
      codeowners CodeOwners.voice_of_customer

      def create
        ...
      end
    end
  end
end
```

**Sidekiq job:**

```ruby
class SendVocReminderJob
  include Sidekiq::Worker
  codeowners CodeOwners.voice_of_customer

  def perform(...)
    ...
  end
end
```

Only use this pattern when the class is brand-new in the MR and has no existing `codeowners` declaration. If the user is editing an existing host-team file, skip to Pattern 2.

## Pattern 2 — `with_codeowners` block (VOC logic inside an existing host-team class)

When VOC ships logic into a controller, job, or service that's already owned by another team, wrap **just the VOC-specific code** in a block. The class-level declaration stays untouched.

**Before:**

```ruby
class PipelineFilterController < ApplicationController
  codeowners CodeOwners.campaigns

  def apply
    cards = KanbanCard.for_org(...)
    cards = apply_business_unit_filter(cards) if business_unit_filter_enabled?
    render json: cards
  end
end
```

**After:**

```ruby
class PipelineFilterController < ApplicationController
  codeowners CodeOwners.campaigns       # ← unchanged; do not touch

  def apply
    cards = KanbanCard.for_org(...)
    if business_unit_filter_enabled?
      cards = CodeOwners.with_codeowners(CodeOwners.voice_of_customer) do
        apply_business_unit_filter(cards)
      end
    end
    render json: cards
  end
end
```

How the routing breaks down:
- Errors raised inside the `with_codeowners` block → tagged `codeowners:voice_of_customer` → route to VOC's Sentry alert / `#voc_monitoring`
- Errors raised outside the block (Pipeline's general code) → tagged `codeowners:campaigns` → route to Pipeline's existing alerts
- Both teams' on-call routing is preserved

Wrap at the **smallest meaningful scope** — usually a method body or a specific block of VOC logic. Don't wrap the entire action if only part of it is VOC code; that pulls Pipeline's logic into VOC's ownership unnecessarily.

## Pattern 3 — Explicit error captures

When VOC code paths call `Platform::ErrorReporting::Public.capture` directly (e.g., in a `rescue` block) and no `with_codeowners` block is active above the call, attach the tag inline:

```ruby
rescue StandardError => e
  Platform::ErrorReporting::Public.capture(
    e,
    tags: { 'codeowners' => CodeOwners.voice_of_customer }
  )
end
```

If the surrounding scope is already inside a `with_codeowners(:voice_of_customer)` block, the tag is already set on the Sentry scope — the explicit `tags:` arg is redundant but not harmful.

## Pattern 4 — `with_codeowners` blocks in other class types (service modules, Karafka consumers, models, etc.)

The `codeowners` class-level DSL is **only mixed into Rails controllers and Sidekiq workers** (via `Controllers::DSL` and `Job::DSL` in `app/utilities/code_owners.rb`). For any other class type — plain modules, service objects, Karafka consumers, ActiveRecord models, decorators — the class-level DSL doesn't exist, and calling `codeowners :voice_of_customer` at the top of the class body would raise `NoMethodError`.

For these classes, wrap each **public entry-point method body** with `CodeOwners.with_codeowners(:voice_of_customer) { ... }`.

**Service module / public API helper:**

```ruby
# Real-world example pattern — see app/components/business_financing.rb
# for the fintech_b2b_financing version of this shape.
module VoiceOfCustomer
  module QuietHours
    def self.should_defer?(message)
      CodeOwners.with_codeowners(CodeOwners.voice_of_customer) do
        QuietHoursPolicy.new(message).should_defer?
      end
    end

    def self.compute_deferral_time(timezone)
      CodeOwners.with_codeowners(CodeOwners.voice_of_customer) do
        TimezonePolicy.compute_next_safe_window(timezone)
      end
    end
  end
end
```

Each public method that callers from outside the module hit gets its own wrap. Private methods (`private def ...` or `def self.xyz` with no external callers) don't need wrapping — their callers (the public methods) already set the tag on the Sentry/Datadog scope, and the tag stays set for the duration of the public method's execution.

**Karafka consumer:**

```ruby
class VoiceOfCustomer::QuietHoursConsumer < Karafka::BaseConsumer
  def consume
    CodeOwners.with_codeowners(CodeOwners.voice_of_customer) do
      messages.each { |m| process(m) }
    end
  end
end
```

Same pattern — wrap the entry-point method (`consume`).

**Plain Ruby class (model, decorator, value object):**

```ruby
class VoiceOfCustomer::FeedbackDecorator
  def initialize(feedback)
    @feedback = feedback
  end

  def display_summary
    CodeOwners.with_codeowners(CodeOwners.voice_of_customer) do
      # ... transformation logic
    end
  end
end
```

**Quick reference for choosing the right pattern:**

| Class type | Pattern |
|---|---|
| Rails controller | **Pattern 1** (class-level `codeowners`) |
| Sidekiq worker (`include Sidekiq::Worker`) | **Pattern 1** (class-level `codeowners`) |
| Service module / public API helper (`module Foo; def self.bar; ...`) | **Pattern 4** (`with_codeowners` block per public method) |
| Karafka consumer | **Pattern 4** (wrap `consume`) |
| Plain class (model, decorator, value object, etc.) | **Pattern 4** (wrap each public method) |
| Inside another team's existing class (any type) | **Pattern 2** (`with_codeowners` block around the VOC-specific code only) |
| Direct `Platform::ErrorReporting::Public.capture` call in a VOC code path | **Pattern 3** (inline `tags:` arg) |
| Frontend React/TypeScript — VOC-owned error capture | **Pattern 5** (pass `Codeowners.VOICE_OF_CUSTOMER` to the capture helper) |
| Frontend React/TypeScript — VOC code inside a host-team file | **Pattern 5** (VOC enum on VOC's call sites only; leave the host team's captures untouched) |

## Anti-pattern — never do this

```ruby
# ❌ BAD: replaces the host team's class-level codeowners, silently steals ownership
class SomeExistingPipelineController < ApplicationController
  codeowners CodeOwners.campaigns               # ← was here
  codeowners CodeOwners.voice_of_customer       # ← NEVER add this; the second declaration overwrites the first
end
```

If you find yourself wanting to add a class-level VOC codeowner to a file that already declares another team's codeowner, **stop and use Pattern 2 instead**. If the user explicitly asks for the class-level replacement, push back: explain the consequence (stolen ownership, broken on-call for the host team) and propose the `with_codeowners` block alternative.

## Behavior rules

When this skill is active, follow these in priority order:

1. **Always run the detection step first** (grep for `codeowners` in the target file) before suggesting any tag.
2. **Auto-include the tag in generated code** when adding a new entrypoint or wrapping VOC-specific logic — don't ask "want me to add the codeowner tag?" every time. The point of this skill is to make tagging effortless.
3. **Always explain the choice in the conversation message.** Tell the user which pattern was applied and why — e.g., *"I added `codeowners CodeOwners.voice_of_customer` because this is a new VOC-owned controller"* or *"I wrapped just the filter logic in `with_codeowners` because `codeowners :campaigns` is already declared at the class level — preserves Pipeline's ownership of the rest of the file."*
4. **Refuse the anti-pattern.** If the user asks to replace a host team's class-level codeowners with VOC's (backend), or to flip the codeowner argument on a host team's existing frontend capture call, push back with the on-call-ownership explanation and propose the scoped alternative (Pattern 2 backend, Pattern 5 frontend).
5. **Don't tag code the user didn't ask you to touch.** Tagging code Claude is already generating is in-scope. Going back and tagging unrelated existing files (or proposing diffs the user didn't request) is out of scope.

## Pattern 5 — Frontend exception capture (React / TypeScript)

The frontend has **no class-level codeowner and no `with_codeowners` block.** Every capture helper takes the codeowner as a **required positional argument**, so the tag is chosen *per call site*. That makes scoping naturally additive: you decide which individual captures carry `Codeowners.VOICE_OF_CUSTOMER` — there is no file-wide or module-wide default to overwrite, so the "only what we touch" rule is enforced by where you pass the enum.

The enum is `Codeowners.VOICE_OF_CUSTOMER = 'voice_of_customer'` in `app/assets/javascripts/components/Common/Error/Codeowners.ts`. The capture helpers all live in `app/assets/javascripts/utils/exception/` and set `tags: { codeowners }` on the Sentry event:

| Helper | Signature | Use for |
|---|---|---|
| `captureException` | `(error, codeOwners, tags?, level?)` | direct Sentry capture — try/catch, error boundaries |
| `captureWarning` | `(error, codeOwners, tags?)` | non-error warnings |
| `catchNetworkExceptions` | `(func, codeOwners, errorHandler)` | wrapping an API call |
| `rethrowAxiosError` | `(func, codeOwners, errorHandler?)` | API call that rethrows after handling |

**VOC owns its own frontend code → pass the VOC enum on that call:**

```ts
import { captureException } from 'utils/exception/captureException';
import { Codeowners } from 'components/Common/Error/Codeowners';

try {
  await submitVocSurveyResponse(payload);
} catch (error) {
  captureException(error, Codeowners.VOICE_OF_CUSTOMER);
  throw error;
}
```

**VOC ships into a host-team file → tag only VOC's own call sites, leave the host team's untouched:**

```ts
// PipelineCardApi.ts — owned by Pipeline (campaigns)
export const fetchCards = () =>
  catchNetworkExceptions(() => get('/pipeline/cards'), Codeowners.CAMPAIGNS, handler); // ← unchanged

// VOC-added filter call in the same file → VOC's tag on THIS call only
export const applyVocBusinessUnitFilter = (cardId: string) =>
  catchNetworkExceptions(
    () => post(`/pipeline/cards/${cardId}/voc_filter`),
    Codeowners.VOICE_OF_CUSTOMER,
    handler
  );
```

Errors from `applyVocBusinessUnitFilter` route to `#voc_monitoring`; everything else stays tagged `campaigns` and routes to Pipeline. This is the frontend equivalent of Pattern 2 — scope to the smallest meaningful unit (here, the individual capture call).

### Frontend anti-pattern — never do this

```ts
// ❌ BAD: flipping the host team's existing capture to VOC's tag
// re-routes Pipeline's errors to #voc_monitoring and breaks their alerting
export const fetchCards = () =>
  catchNetworkExceptions(() => get('/pipeline/cards'), Codeowners.VOICE_OF_CUSTOMER, handler);
//                                                      ↑ was Codeowners.CAMPAIGNS — do not change it
```

Only add `Codeowners.VOICE_OF_CUSTOMER` to call sites VOC introduces. Never flip the codeowner argument on a host team's existing capture, and never relabel a whole shared API module to VOC's tag — that silently steals their error routing exactly like the backend class-level overwrite (the anti-pattern above). If you're adding VOC logic and the capture helper isn't called yet, add the capture with the VOC enum; if the host team already captures there for their own logic, add a *separate* VOC-tagged capture for the VOC failure path rather than editing theirs.

The activation conditions and behavior rules above apply identically.

## Maintenance contract

This skill makes the following assumptions. If any change, update this file and re-distribute to the team.

1. **Branches use the `VOCR-` prefix.** If the Jira project key or branch convention changes, update the activation signal.
2. **`CodeOwners` DSL accepts a single team_name.** Multi-codeowner support is not currently in the platform; "use a `with_codeowners` block to scope ownership" is the workaround. If the platform adds multi-codeowner support, the guidance here can be relaxed.
3. **Sentry tag key is `codeowners`** (set by `app/utilities/code_owners.rb:SENTRY_TAG = 'codeowners'`).
4. **VOC's slug is `voice_of_customer`** in `TEAM_NAMES`. If the team is renamed, update this skill, `TEAM_NAMES` in `code_owners.rb`, the Sentry alert filter, the TS enum, and any code paths already using the tag.
5. **VOC ships into Pipeline (`campaigns`) and Online Booking (`online-booking`).** If VOC takes ownership of its own domain at `app/components/voice_of_customer/` and Pattern 2 becomes rare, simplify accordingly.
6. **Frontend capture helpers take the codeowner as a required positional argument** and live in `app/assets/javascripts/utils/exception/` (`captureException`, `captureWarning`, `catchNetworkExceptions`, `rethrowAxiosError`). There is no class-level/block scoping on the frontend. If these signatures or locations change, update Pattern 5.

## Reference

- **VOC codeowner foundation:** [VOCR-83](https://housecall.atlassian.net/browse/VOCR-83) — adds `voice_of_customer` to `TEAM_NAMES` in `app/utilities/code_owners.rb`
- **Sentry alert rule:** [`monitors/alerts/3500612`](https://codefied.sentry.io/monitors/alerts/3500612/) — routes `codeowners:voice_of_customer` errors to `#voc_monitoring` (production only), auto-creates a Jira issue
- **Platform doc:** [Codeowners in housecall-web (Confluence)](https://housecall.atlassian.net/wiki/spaces/ENG/pages/1480034769/Codeowners+in+housecall-web)
- **Source code:** `app/utilities/code_owners.rb` (Ruby DSL), `app/assets/javascripts/components/Common/Error/Codeowners.ts` (TypeScript enum)
