# Risk at scope

A capability of open-org-spec describing how risks are tracked at any scope of a conformant repository: where risk records live, what every risk declares, how its RAG status is derived, how it moves through its lifecycle, and how it escalates when it goes stale.

**Status:** Active

**Owner:** Javier Fernandez

## Purpose

Organisations conforming to this standard surface risks at multiple scopes: a project hits a dependency it cannot control, a module carries a compliance exposure, a cross-cutting concern threatens several workstreams at once. Without a shared shape, each scope re-invents how a risk is recorded, who owns it, and when it becomes urgent — and risks decay silently in chat, decks, and memory.

This capability codifies the recurring pattern so adopters don't re-derive it at each scope. It is deliberately **separate from [`governance-at-scope`](../governance-at-scope/spec.md)**: governance answers "who decides here?"; risk-at-scope answers "what threatens this scope, who owns it, and is it going stale?" A scope can carry risks without carrying its own governance folder — it needs only declared people (`people.md`) or a DACI to identify owners and accept dispositions.

## Dependencies

- **Requires [`feedback-inbox`](../feedback-inbox/spec.md) active at the scope (or a higher scope).** Escalation is the mechanism that keeps a risk from going stale; the escalation contract writes a disposition request to a scope's `feedback.md`. A scope tracking risks without a reachable feedback inbox has no escalation path and is outside this capability.
- **`accepted` transitions require [`governance-at-scope`](../governance-at-scope/spec.md) OR [`project`](../project/spec.md) active at the scope.** Accepting a risk is a decision authority act — someone with standing must own the trade-off and record it in a full ADR. A scope with neither active governance nor an active project has no authority to accept a risk; it may defer, mitigate, or close, but not accept (see [Status lifecycle](#status-lifecycle)).

## Risk record schema

A risk is a markdown file named `YYYY-MM-DD-slug.md`, living inside a `risks/` folder at the scope. The opening date is encoded in the filename. Fields are carried as YAML frontmatter:

| Field | Required | Meaning |
|---|---|---|
| `id` | Yes | Sequential identifier `R-NNN`. Declared by the author; suggested by [`/new-risk`](./new.md) from the risk registry. Unique within the registry's aggregation scope. |
| `title` | Yes | Short label for the risk. |
| `description` | Yes | Free-text statement of the risk: what could go wrong and why it matters. |
| `created_at` | Yes | ISO date (`YYYY-MM-DD`) on which the risk was first raised. Must match the date prefix in the filename. Canonical source for RAG age derivation — immune to filename changes. Set once at creation; never changed. |
| `rag` | Derived | `RED \| AMBER \| GREEN`. **Derived from objective criteria** (see [RAG derivation](#rag-derivation)), never manually declared. The author records the derived value; tools recompute it. |
| `owner` | Yes | Named person(s) accountable for dispositioning the risk, declared at this scope (present in the scope's `people.md` or DACI). May be multiple. |
| `status` | Yes | `open \| deferred \| mitigated \| accepted \| closed`. See [Status lifecycle](#status-lifecycle). |
| `escalation_threshold` | Yes | Integer days of inactivity (since creation, while `open`) before the risk escalates. Drives RAG derivation and the [escalation contract](#escalation-contract). |
| `disposition_at` | When dispositioned | ISO date (`YYYY-MM-DD`) of the last disposition action. Set whenever `status` changes or the risk is explicitly re-affirmed. |
| `disposition_decision_ref` | When `accepted` (required); when `deferred` / `mitigated` (recommended) | Relative link to the decision record that dispositioned the risk. Required when `status: accepted` (points to the ADR). Recommended when `status: deferred` (points to the lightweight disposition record) and `status: mitigated` (points to the project decision or action that resolved it). |
| `scope` | Required for `risks/` (repo root); optional elsewhere | A scope reference in `<type>/<slug>` format declaring which scope this risk belongs to. Used by the risk scanner to resolve the target feedback inbox via the scope registry. Type vocabulary: `cluster`, `function`, `project`, `module`, `programme` (no slug — use `programme` alone for cross-cutting risks). Examples: `cluster/product-development`, `function/revenue`, `project/agentic-coach-phase-3`, `programme`. |
| `mitigation` | No | Free-text description of the action or plan being taken to address the risk. Leave blank or omit when no specific mitigation is in place. |
| `related` | Optional | Links to relevant project specs, feedback entries, ADRs, or other context. |

**Representation is extension-overridable.** The YAML frontmatter shown above is the default. Adopters may declare a different header form in their extension spec, provided every required field remains present, addressable, and named in the extension's mapping. The capability requires the fields, not the syntactic form. Adding, renaming, or removing required fields is not authorised by this override — only the form changes; the contract does not.

### Required body sections

In addition to the YAML frontmatter, a conformant risk record must contain:

**`## Log`** — An append-only section recording every change to the risk's disposition. Each entry uses the strict heading format:

```
### YYYY-MM-DD — <author name> — <change type>
```

`<change type>` is one of: `confirmed open` | `status changed to <value>` | `owner changed to <name>` | `disposition date updated`.

The entry body is free text explaining what was found, what was decided, and why. Minimum: one sentence. The section header and heading format are machine-parseable; do not vary them.

A `## Log` entry is **required** whenever `disposition_at`, `status`, or `owner` changes. The entry date must match the new `disposition_at` value exactly.

Example:

```markdown
## Log

### 2026-06-17 — Jane Smith — confirmed open
Reviewed. Risk is still live. Mitigation plan in progress. Not ready to defer.

### 2026-06-03 — Jane Smith — status changed to open
Risk created.
```

## RAG derivation

`rag` is **derived, never manually declared.** It is a function of `status`, the days elapsed since `created_at`, and the `escalation_threshold`. Let `age` be the number of days since `created_at`:

- **GREEN** — `status` is not `open` (any terminal or deferred state is, by definition, no longer a live red flag), OR `status: open` with `age < 50%` of `escalation_threshold`.
- **AMBER** — `status: open` with `age` between `50%` and `100%` of `escalation_threshold` (i.e. `0.5 × threshold ≤ age < threshold`).
- **RED** — `status: open` with `age ≥ escalation_threshold`, OR escalation has been explicitly requested (a contributor or owner can force RED ahead of the threshold).

Because the value is derived, the `rag` field recorded in a file is a cached snapshot. Tools that read risks recompute it from `status`, the filename date, and `escalation_threshold`; a stale cached value is a `warning`, not a contradiction. The risk scanner is responsible for keeping cached values current.

## Status lifecycle

A risk opens at `open` and moves to exactly one terminal disposition. The transition gates:

- **`open → deferred`** — a conscious decision to revisit later. Requires a **lightweight disposition record** in `decisions/` at the scope: decider, rationale, review trigger, and `decided_at`. `disposition_decision_ref` should point to it. Also requires a `## Log` entry dated today explaining why the risk is being deferred and what the review trigger is. Deferral is reversible only by creating a new risk, not by reopening (see below).
- **`open → accepted`** — the scope decides to live with the risk. Requires a **full ADR** in `decisions/` at the scope: DACI, rationale, and `decided_at`. `disposition_decision_ref` is **required** and points to the ADR. Also requires a `## Log` entry dated today summarising the acceptance rationale. This transition **requires [`governance-at-scope`](../governance-at-scope/spec.md) OR [`project`](../project/spec.md) active at the scope** — acceptance is a decision-authority act.
- **`open → mitigated`** — the risk was resolved by a project decision or action. The risk traces to that resolution: `disposition_decision_ref` points to the project decision, ADR, or action record that closed the exposure. Requires a `## Log` entry dated today describing what was done and why the exposure is considered resolved.
- **`open → closed`** — administrative closure (the risk no longer applies; the context changed; it was a duplicate). A `disposition_at` date suffices for the frontmatter; no decision record is required. Requires a `## Log` entry dated today stating the closure reason (e.g., superseded by, no longer applicable, duplicate of).

**Terminal states do not reopen.** Any move from a terminal state (`mitigated`, `accepted`, `closed`) — or from `deferred` — back to `open` is **not permitted**. Risks don't reopen: if the concern resurfaces, a **new risk** is created with a new `id` and creation date. This keeps each risk record a faithful account of a single concern's life, and keeps RAG age measurement honest (an aged risk cannot be laundered back to GREEN by reopening).

Every disposition sets `disposition_at` to the date of the action.

### Disposition frame

When an owner dispositions a risk — at escalation, at review, or proactively — the standard disposition frame is:

> *Confirm with date / defer with reason / reassign*

Every escalation request, and every disposition recorded in `decisions/`, offers these three moves. "Confirm with date" re-affirms the risk and refreshes `disposition_at` (the risk stays `open` but its review clock is acknowledged) **and requires a `## Log` entry dated the same day explaining what was reviewed and why the risk was not deferred or closed** — a date change without a log entry is not a valid confirmation. "Defer with reason" moves it to `deferred` with a recorded rationale. "Reassign" hands ownership to a different declared person at the scope.

## Where risks live

Risks live in a dedicated `risks/` folder at the scope. **`risks/` is a sibling of `decisions/` at the same scope** — peer to it, not nested inside — so a risk and the decision that dispositions it sit side by side and are discoverable from the scope root.

- **At any scope:** `<scope>/risks/YYYY-MM-DD-slug.md`
- **Programme-level (cross-cutting, repo root):** `risks/YYYY-MM-DD-slug.md` — risks that span scopes and belong to no single one live at the repo root, alongside the repo-wide `decisions/`.

Absence of a `risks/` folder at a scope means that scope tracks no risks of its own. A risk that affects several scopes but is owned at one of them lives at the owning scope and links the others via `related`; a genuinely cross-cutting risk with no single owning scope lives at the programme level.

## Who can create risks

**Anyone declared in the scope's `people.md` or DACI** can create a risk at that scope. This is the same standing-membership test the [`people`](../people/spec.md) and [`governance-at-scope`](../governance-at-scope/spec.md) capabilities use: a contributor with accountability at the scope can raise a risk there; a passer-by routes through a feedback entry instead. [`/new-risk`](./new.md) verifies this membership before scaffolding.

Risk ownership is narrower than risk creation: an `owner` must be a named person declared at the scope. The creator need not be the owner.

## Escalation contract

When a risk breaches its `escalation_threshold` — `status: open` and `age ≥ escalation_threshold`, i.e. it derives to RED on the age criterion — the **risk scanner writes a disposition request to the scope's `feedback.md`** using the [disposition frame](#disposition-frame). The entry follows the [`feedback-inbox`](../feedback-inbox/spec.md) substantive-entry format, addressed to the risk's owner, and offers *confirm with date / defer with reason / reassign*.

**Feedback inbox routing.** The scanner resolves the target `feedback.md` using the following precedence:

1. **`scope` declared:** the scanner resolves the `scope` field via `governance/catalogue/scopes.yaml` (requires the `scope-registry` capability to be active and `scopes.yaml` to exist). The catalogue entry provides the `feedback_inbox` path; the scanner routes there, addressed to the risk's owner. If `scope-registry` is inactive or the slug is not found in the catalogue, fall back to path inference (step 2).
2. **`scope` absent:** infer the owning scope from the risk file's path (existing behaviour — unchanged). A risk at `clusters/product-development/risks/` is inferred to belong to the `product-development` cluster and its `feedback.md` is used.
3. **Fallback:** if neither step resolves a feedback inbox, the scanner routes to `governance/feedback.md` and prefixes the entry with a `[scope-unresolved]` warning so the governance owner can re-route manually.

**Multi-owner escalation.** When a risk has multiple owners, the risk scanner writes **separate disposition request entries to each owner's scope `feedback.md`** — each owner is addressed in their own inbox rather than relying on one shared entry. This guarantees every accountable party sees the request through their normal catchup flow.

The owner's response — recorded inline per the feedback-inbox resolution convention — drives the disposition: confirming refreshes `disposition_at`, deferring or reassigning moves the risk accordingly. An unanswered escalation stays RED and re-surfaces on the next scanner run.

## Project close requirement

A project cannot close while it carries live risks. **All risks in a closing project must be in a terminal state (`mitigated`, `accepted`, `closed`) before the project transitions to `closed`.** `open` and `deferred` risks **block project closure**: a deferred risk has, by definition, unfinished business, and a closing project is the wrong place to leave it — it must be dispositioned to a terminal state or migrated to a surviving scope (via the closing project's [migration audit](../project/spec.md#closing-a-project)) where it can carry on being tracked.

This is enforced as part of the [`project`](../project/spec.md) close flow: the closing audit walks the project's `risks/` folder, and any non-terminal risk is a gap that must be resolved before close. The risk registry excludes `projects/closed/` precisely because a closed project's risks are guaranteed terminal.

## Extension points

- **`escalation_threshold` default.** The capability requires the field but does not fix a default. Adopters set a scope-wide or repo-wide default (e.g. 30 days) in their extension spec; [`/new-risk`](./new.md) offers it as the suggested value while letting the author override per risk.
- **RAG derivation thresholds.** The `50%` AMBER boundary and the `100%` RED boundary are defaults. Adopters may declare different fractions in their extension spec (e.g. AMBER at 60%), provided the ordering holds (`GREEN < AMBER < RED` by age) and the derivation stays a pure function of `status`, age, and `escalation_threshold`. Replacing derivation with a manually declared `rag` is not authorised by the override — the contract that RAG is derived does not change.
- **Risk ID prefix.** `R-` is the default prefix for the `id` field. Adopters may declare a different prefix in their extension spec (e.g. `RISK-`), provided it is consistent across the registry's aggregation scope and the registry's next-ID suggestion uses it.

## What is not prescribed

- **Whether every scope tracks risks.** A scope without a `risks/` folder tracks none of its own. Most small scopes will carry no risks until one is raised.
- **A risk-scoring matrix (likelihood × impact).** The capability derives RAG from staleness, not from a probability/impact score. Adopters who want a scoring matrix add it in their extension as an optional field; it does not replace the derived `rag`.
- **The wording of `description`, `title`, or disposition rationale.** Free text; the capability requires the fields, not their contents.
- **A specific folder name other than `risks/`.** The name is convention-bound; adopters may use a different name in their extension, but lose tool interop (the registry's discovery walk looks for `risks/`).
- **The cadence of the risk scanner.** When and how often the scanner runs is an adopter operational choice; the capability defines the contract it must honour when it does run.

## Rationale

Three choices shaped this capability:

1. **Separate from governance.** Risks attach to scopes that operate, not only to scopes that decide. A project with no governance folder still carries dependency risks; a module with declared people but no DACI still has exposures to track. Coupling risk tracking to a governance folder would leave those scopes with nowhere to put a risk. The lighter membership test (`people.md` or DACI) is what most live scopes already satisfy.

2. **RAG is derived, not declared.** A manually coloured RAG is a vanity field — owners mark things GREEN to avoid attention. Deriving the colour from staleness makes the status honest: a risk that has sat `open` past its threshold goes RED whether or not anyone wants it to, and the only way to clear it is to actually disposition it. This is the same discipline `governance-at-scope` applies with `decided_at`: the explicit, mechanical signal beats the self-reported one.

3. **Risks don't reopen.** Allowing a terminal risk back to `open` would corrupt both the historical record (one file, one concern, one life) and the age-based RAG (a stale risk could be reset). Creating a new risk when a concern resurfaces costs almost nothing and keeps every record trustworthy.

## Automated tooling (forthcoming)

Two agents are defined by this capability but not yet specced. They are tracked in the standard's backlog and will be added as separate files in this directory when their design is stable.

**Risk registry agent** (`registry.md`) — walks all `risks/` folders across governed scopes (excluding `projects/closed/`), aggregates all risk records into a machine-readable registry file, and writes a delta log of what changed since the previous run. Runs daily, delta-based (see backlog B-004). Produces:

```yaml
---
generated: YYYY-MM-DDTHH:MM:00Z
tool: risk-registry
period_days: 1
key_metrics:
  open_risks: N
  red_risks: N
  amber_risks: N
  unowned_risks: N
  awaiting_disposition: N
---
```

**Risk scanner agent** (`scanner.md`) — scans the registry for risks that have breached their `escalation_threshold` and routes a disposition request to each owner's scope `feedback.md` using the standard disposition frame. Runs daily alongside the registry agent.

Both agents are Case B standard capability agents (no project spec required — the spec is this document).

## Invocation

See [`new.md`](./new.md) for the `/new-risk` command that scaffolds a new risk record.

## Related

- [`../feedback-inbox/spec.md`](../feedback-inbox/spec.md) — the escalation contract writes disposition requests to a scope's `feedback.md`; required dependency.
- [`../governance-at-scope/spec.md`](../governance-at-scope/spec.md) — sibling capability; source of the decision-authority test that gates `accepted`, and of the `decisions/` sibling-folder and `owner` conventions reused here.
- [`../project/spec.md`](../project/spec.md) — the project close requirement is enforced in the project close flow; mitigated risks trace to project decisions.
- [`../people/spec.md`](../people/spec.md) — the standing-membership test for who can create and own risks at a scope.
- [`new.md`](./new.md) — the `/new-risk` command for this capability.
