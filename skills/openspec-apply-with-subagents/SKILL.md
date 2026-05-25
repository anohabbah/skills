---
name: openspec-apply-with-subagents
description: Use when implementing a non-trivial multi-slice OpenSpec change and you want per-slice quality gates — dispatches a fresh implementer subagent per slice, gated by a spec-compliance reviewer (which runs `openspec validate`) and a code-quality reviewer. For trivial changes, prefer `openspec-apply-change`.
---

# Apply an OpenSpec change with subagent orchestration

Heavy-process variant of `openspec-apply-change`. Inherits steps 1–5 from
that skill (change selection, status, apply instructions, read context
files, show progress). Step 6 is replaced with a slice-by-slice
subagent loop that combines OpenSpec's task protocol with
`superpowers:subagent-driven-development`'s implementer + two-stage
review pattern.

Choose this over `openspec-apply-change` when the change has ≥2 slices
or design-sensitive work. For trivial changes the lighter skill wins —
reviewer overhead dominates.

## Vocabulary

- **Checkbox** — one line in `tasks.md`, one red-green-refactor cycle.
- **Slice** — a heading-grouped block of checkboxes in `tasks.md` that
  implements one vertical feature. Slices are the **dispatch unit**.
  Checkboxes inside a slice are tightly coupled and stay together.

## Preconditions

- `tasks.md` uses headings to group checkboxes into slices.
- Each checkbox is fine-grained (one red-green cycle).
- The project follows a TDD discipline (encoded somewhere — `CLAUDE.md`,
  `AGENTS.md`, or `GEMINI.md`).
- `openspec` CLI is available.

If those don't hold, fall back to `openspec-apply-change`.

## Process

### Steps 1–5: inherit from `openspec-apply-change`

Run them as written: select change, `openspec status --json`,
`openspec instructions apply --json`, read every `contextFiles` entry,
show progress.

### Step 5.5: parse `tasks.md` into slices

Build an in-memory slice list. For each slice, record:

- Slice heading text and ordinal (`slice N/M`).
- Ordered list of pending checkboxes (skip already-checked ones).
- Which spec delta section(s) under `contextFiles.specs` this slice
  touches (match by heading / feature name).
- Which `design.md` section(s) are relevant, if any.
- Brief scene-setting: what prior slices established, what this slice
  must establish, what later slices will depend on.

If a slice has >8 pending checkboxes, split it at a natural layer seam
(for example, a data-access half and a feature-wiring half) and treat
the halves as separate dispatch units. Note the split in the
scene-setting so each half knows what the other did.

Create TodoWrite items, one per slice.

### Step 6: per-slice loop

For each slice, in order, follow `superpowers:subagent-driven-development`
with these per-step specifics.

#### 6a. Dispatch the implementer subagent

Construct the prompt with these sections, **inlined** (the subagent
must not be told to read `tasks.md` or proposal artifacts — controller
curates the bundle):

1. **Scene-setting** — slice N/M, what's already built, what this slice
   must deliver, what comes next.
2. **Relevant spec delta excerpts** — only the sections this slice
   touches, copied verbatim from `contextFiles.specs`.
3. **Relevant design excerpts** — only if the slice is design-sensitive.
4. **Ordered checkbox list** — the pending checkboxes for this slice,
   as the TDD plan to walk.
5. **Pinned project rules** — load-bearing rules from `CLAUDE.md` /
   `AGENTS.md` / `GEMINI.md` that the subagent must not violate.
   Examples of the kind of thing that belongs here (paraphrase
   whatever the project actually uses):
   - Test runner invocation and flag quirks the subagent will get
     wrong if left to default (e.g. monorepo task runners, deprecated
     flags).
   - Test granularity rules (e.g. "feature-level behaviour goes in
     e2e, not unit tests stitching multiple layers").
   - Styling / component-library preferences.
   - Architecture boundary rules.

   These *are* in `CLAUDE.md` and the subagent inherits it, but
   inlining the load-bearing ones makes them impossible to miss.
6. **TDD discipline** — instruct the subagent to invoke
   `superpowers:test-driven-development`, walk checkboxes
   red-green-refactor, commit per cycle.
7. **Bookkeeping instruction** — update `- [ ]` → `- [x]` in
   `tasks.md` as each cycle lands. Do not touch checkboxes outside
   this slice.
8. **Return contract** — report status as `DONE`,
   `DONE_WITH_CONCERNS`, `NEEDS_CONTEXT`, or `BLOCKED`, per
   `superpowers:subagent-driven-development`.

**Model selection.** Cheap model for mechanical slices (single-layer,
clear spec), standard model for slices spanning layers, capable model
for slices that require design judgement.

Handle the four return statuses per `superpowers:subagent-driven-development`.

#### 6b. Dispatch the spec-compliance reviewer subagent

Prompt contents:

- Slice heading and what the implementer claimed to deliver.
- Git diff of the slice's commits (gather SHAs from the implementer
  report).
- **Raw, uncurated** artifacts: `proposal.md`, the **full** spec delta
  file(s) for this slice from `contextFiles.specs`, and `design.md` if
  present. The reviewer reads these itself. This is the **one place**
  the "don't make subagents read files" rule is suspended, because the
  files *are* the canonical spec, not a plan — and reusing the
  implementer's curated excerpts would create an echo-chamber review.
- Instruction: run `openspec validate --change <name>` and include
  its output in the verdict.
- Verdict format: ✅ approved, or ❌ with a concrete list of
  **spec gaps** (missing requirements) and **spec violations**
  (extras the spec didn't ask for).

Loop: implementer fixes → spec reviewer re-reviews. Repeat until ✅.

#### 6c. Dispatch the code-quality reviewer subagent

Prompt contents:

- Git diff of the slice's commits.
- A project-conventions checklist paraphrased from `CLAUDE.md` /
  `AGENTS.md` / `GEMINI.md` (idiomatic language/framework usage,
  architecture boundaries, styling rules, etc.).
- Verdict: ✅ approved, or ❌ with concrete issues by severity.

Loop until ✅.

#### 6d. Mark slice complete in TodoWrite

### Step 7: whole-change final review

After all slices: dispatch a final code-reviewer subagent over the
entire change diff (from branch divergence to HEAD). Surface anything
slice-local reviewers couldn't see — cross-slice consistency, dead
re-exports, drift between slices, integration gaps.

### Step 8: show status

Per `openspec-apply-change`'s step 7. If all done, suggest
`/opsx:archive` (or whatever the project's archive command is).

## Guardrails

- **Never** dispatch multiple implementer subagents in parallel for the
  same change — they will conflict on `tasks.md` and the workspace.
- **Never** skip spec review, and **never** run code-quality review
  before spec compliance is ✅. Order matters.
- **Never** inline proposal/spec artifacts into the spec reviewer's
  prompt — it must read raw artifacts to avoid echo-chamber review.
- **Never** edit `tasks.md` from the controller; the implementer
  subagent owns checkbox updates.
- If a slice's red-green cycles reveal a design issue, the implementer
  returns `BLOCKED`; the controller pauses and prompts the user to
  update artifacts (per `openspec-apply-change`'s fluid workflow stance).

## When to fall back to `openspec-apply-change`

- Change has only one slice and ≤4 checkboxes.
- Change is purely mechanical (renames, dependency bumps, doc edits)
  with no spec semantics to review.
- You're iterating fast on an experimental change and the reviewer
  overhead outweighs the benefit.

## Integration

- Inherits orchestration patterns from
  `superpowers:subagent-driven-development` (implementer + 2-stage
  review + final review).
- Implementer subagents invoke
  `superpowers:test-driven-development`.
- Inherits steps 1–5 and the "fluid workflow" stance from
  `openspec-apply-change`.
