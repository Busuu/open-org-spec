# Governance at scope

A capability of open-org-spec describing how governance content is organised across the scopes of a conformant repository: where governance lives, what every governance surface must declare, how scopes relate, and how contradictions between scopes are handled.

**Status:** Draft (0.1.0)

## Purpose

Organisations conforming to this standard have governance needs at multiple scopes: the whole repository, individual modules, cross-module coordinations, individual projects. This capability codifies the recurring pattern so adopters don't re-derive it at each scope.

## Pattern

Governance is **scope-hierarchical**. It can emerge at any scope that *operates* — that produces decisions, accepts contributions, or has identifiable ownership. The shape is consistent across scopes; contents vary.

### Where governance appears

- **Repo-wide governance** — rules applying across modules: routing, cross-cutting conventions, repo-wide decisions.
- **Module-level governance** — operational policy specific to a module: compliance, licensing, module-specific rules.
- **Cross-module / sub-module governance** — coordination patterns spanning modules without belonging to any one (e.g., a shared consolidator role).
- **Project-level governance** — rarely needs its own folder; most projects are small enough that higher-scope rules suffice.

### Convention: always a dedicated folder

If governance exists at a scope, it lives in a dedicated `governance/` folder at that scope. **Inline governance is not a permitted shape.** When content is governance, it lives in the folder. Content in a module's README is operational spec (describing what the module does), not governance (describing how it operates).

Absence of a `governance/` folder at a scope means that scope has no declared governance of its own — it is governed only by higher-scope rules.

### Convention: visible decisions at the same scope

Each scope's ADRs live in a sibling `decisions/` folder at the same scope — peer to `governance/`, not nested inside. Decisions are discoverable from the scope's root.

**The `decisions/` folder inherits DACI from the sibling `governance/` folder**; it does not re-declare ownership. A `decisions/README.md` describes only the recording convention.

**When `risk-at-scope` is active:** a `risks/` folder sits alongside `decisions/` at the same scope, as a peer. Risk records live there. The same DACI that governs decisions at this scope governs risk acceptance — the Approver declared in `governance/README.md` is the authority for `accepted` risk transitions at this scope.

#### Decision record schema

Every decision record is a markdown file named `YYYY-MM-DD-short-title.md`. The opening date is encoded in the filename. Required fields in the file body (prose header or frontmatter — adopter choice):

| Field | Required | Meaning |
|---|---|---|
| `Status` | Yes | Current state: `proposed \| draft \| open \| accepted \| decided \| superseded \| closed \| archived` |
| `Owner` | Yes | The person accountable for driving this decision to resolution |
| `decided_at` | When status is closed | ISO date (`YYYY-MM-DD`) when the decision reached its final state. Enables time-to-decision measurement. Required as soon as `Status` transitions to any resolved value. |
| `risk_number` | No | Identifier of a related risk register entry (e.g. `R-042`). Leave blank or omit when the decision has no associated risk. |

**`decided_at` discipline.** When a decision's status changes to a resolved value (`accepted`, `decided`, `superseded`, `closed`, `archived`), the author must add `decided_at` in the same commit. This is the only reliable way to measure decision velocity — git log approximations exist but are lossy (the status field may have been touched for reasons other than resolution). The explicit date is the authoritative source.

For decisions predating this convention: `decided_at` may be backfilled using `git log --follow -p <file>` to identify the commit where the status last changed to a resolved value.

### Convention: every governance folder declares, at minimum

The governance folder's README declares:

- **Owner** — name and role of the person accountable for the governed function. Ownership tracks *accountability*, not *stakeholder interest*: a stakeholder who consumes a governed function is not its Owner.
- **DACI** — Driver, Approver, Contributors, Informed for decisions about this scope's governance. Each entry carries name + role.
- **Scope** — a declaration of what this governance applies to.

### Convention: scope discipline

DACI participants at a scope must have accountability *at that scope*. A module-level contributor does not appear in repo-wide DACI; a repo-wide approver does not appear in module-level DACI unless they carry explicit module-level accountability.

### Convention: Owner-as-default DACI at initial adoption

At initial adoption, if no separate Driver or Approver is named, the Owner fills both roles. This is the sanctioned default; adopters can assign Driver and Approver separately as the scope's operating model matures. Contributors and Informed default to empty (no participants implied).

### Shape of a governance file

Governance is **machine-readable**. A governance folder's README is a markdown file with **YAML frontmatter + prose body**:

```
---
scope: <repo-wide | module | cross-module | project>
applies_to: <path or scope identifier — "." for repo-wide; module path otherwise>
owner:
  name: <name>
  role: <role>
daci:
  driver:
    name: <name>
    role: <role>
  approver:
    name: <name>
    role: <role>
  contributors:
    - name: <name>
      role: <role>
  informed:
    - name: <name>
      role: <role>
cross_references:
  - <path to a higher-scope governance folder>
---

# Governance

[Prose body: scope statement, what lives here, what doesn't, cross-references.]
```

All DACI entries may be `{ name: TBD, role: TBD }` if a role hasn't been assigned yet; the Owner is accountable for resolving TBDs. `contributors` and `informed` may be empty arrays.

The frontmatter is the **authoritative source** for ownership and DACI; the prose body carries the scope narrative, rationale, and cross-reference context that doesn't fit the structured schema.

**Representation is extension-overridable.** The YAML frontmatter shown above is the default. Adopters may declare a different header form (e.g., a prose markdown header) in their extension spec, provided every required field (`scope`, `applies_to`, `owner`, `daci`, `cross_references`) remains present, addressable, and named in the extension's mapping. The capability requires the fields, not the syntactic form. Adding, renaming, or removing required fields is not authorised by this override — only the form changes; the contract does not.

### Precedence and contradictions

**Upper-scope governance takes precedence over lower.** Lower scopes may *add* rules specific to their scope; they may not *contradict* higher-scope rules.

When an LLM or tool operating on a conformant repository reads the governance hierarchy and detects a contradiction — a rule in a lower-scope governance file that would require different behaviour at the same field than a higher-scope rule — the contradiction is **flagged**, not auto-resolved. The governance owner of the relevant scope decides whether the lower-scope rule is rewritten to conform, or the higher-scope rule is proposed to change.

A first-pass definition of contradiction: a rule in a lower-scope governance file that, if followed, would produce behaviour at the same field or object that conflicts with a higher-scope rule's requirement. Detection is best-effort at this stage of the standard; precision sharpens as use cases accumulate.

## What governance is not

Governance at a scope is distinct from:

- **The scope's operational specs** — workflows, capabilities, what the module does.
- **Content the scope produces** — project deliverables, artifacts, raw material (these route to systems of record or the scope's own folders, not into governance).

Governance describes *how* the scope operates, not *what* it does. This distinction is what lets a module's operational content grow unboundedly without the governance folder growing in proportion.

## Not prescribed

- **Whether every scope has governance.** Scopes without a `governance/` folder are governed only by their higher-scope rules. Projects and small modules commonly have no governance folder at all.
- **Contents beyond the required minimum.** Owner + DACI + Scope are mandatory frontmatter. Beyond that, a scope adds whatever operational rules it needs.
- **Cluster Leads, team leads, or similar operational-ownership declarations** on module READMEs. These are **not** governance content: a module README may name its Lead and Mission as part of the module's operational spec. Governance content lives in the module's `governance/` folder if one exists.
- **A specific folder name other than `governance/`.** The name is convention-bound; adopters may use a different name if their operating model demands, but they lose LLM/tool interop if they do.

## Rationale

Every organisation adopting this standard will face the same shape of question at each scope: where does governance live here, and how does it relate to other scopes? Codifying the pattern — with explicit permission for contents to vary (but not the form) across scopes, and clear precedence rules — gives adopters a shared starting point without forcing uniformity on operational content.

Making governance machine-readable (frontmatter) enables mechanical contradiction detection across scopes, without which the precedence rule degenerates into a manual audit. Structured data at the file level also positions adopters to later build (or consume) a repo-wide catalogue of governance state cheaply, as captured in [`../../backlog.md`](../../backlog.md).

## Adoption

See [`adopt.md`](./adopt.md) for the guided flow that scaffolds a governance folder at a chosen scope.

## Related

- [`adopt.md`](./adopt.md) — the adoption command for this capability.
- [`tools/decision-escalation/spec.md`](./tools/decision-escalation/spec.md) — governance maintenance tool; escalates open decisions past their staleness threshold.
- [`tools/scope-elevation/spec.md`](./tools/scope-elevation/spec.md) — governance maintenance tool; detects artefacts placed at a narrower scope than their content warrants.
- [`../../templates/governance.md`](../../templates/governance.md) — the scaffolding template for a governance folder's README.
- [`../../backlog.md`](../../backlog.md) — open questions, including command-protocol and adopter-state-detection entries surfaced while designing this capability, plus the catalogue proposal that would reduce contradiction-detection cost.
- [`../people/spec.md`](../people/spec.md) — companion capability describing who holds roles at a scope. The two are complementary: governance answers "who decides here?" (DACI, per-scope decision authority); people answers "who is here?" (standing membership, lead accountability). When both exist at a scope, neither duplicates the other.
