---
name: trade-study
description: Use when a design decision has multiple viable options with non-obvious trade-offs. Triggers on "should we use X or Y", connector/topology/IC selection dilemmas, architecture choices that constrain downstream work, or any hard-to-reverse decision where the best option depends on weighing competing priorities. Also use when asked to write, review, or update a trade study.
---

# Trade Studies

Trade studies capture structured comparisons when a design decision has multiple viable options. They make reasoning auditable for the team now and anyone revisiting the decision later.

## When to Write One

**Write one when:**
- Multiple options are viable and the best choice depends on weighing competing priorities
- The decision affects downstream work (connector choice constrains PCB stackup, firmware, mechanical design)
- The decision is hard to reverse once implemented

**Don't write one when:**
- One option is clearly dominant
- The decision is easily reversible
- The scope is too small to justify the overhead

## File Convention

Each trade study is a markdown file, dated and kebab-cased, kept in the repo (e.g. under `docs/trades/`):

```
docs/trades/YYYY-MM-DD-<topic-in-kebab-case>.md
```

## Structure

Every trade study has exactly four sections plus a header block:

### Header Block

```markdown
# Trade Study: <Title>

**Date:** YYYY-MM-DD
**Status:** OPEN | DECIDED
**Decision:** <one-line summary of chosen option>  ← only after decided
```

### 1. Context

What we're deciding and why it matters. Include:
- The problem or constraint that forced the decision
- Requirements the solution must meet (quantified where possible)
- Use cases affected by the decision

### 2. Options

Each viable approach as a subsection (`### Option A: ...`). Describe concisely — what the approach is, how it works, key characteristics. No advocacy — just facts.

### 3. Comparison Table

Rows are evaluation criteria, columns are options. Use qualitative ratings with explanation, not bare checkmarks.

```markdown
| Criteria | A: <name> | B: <name> |
|---|---|---|
| **<criterion>** | <rating + explanation> | <rating + explanation> |
```

**Good criteria to consider for hardware decisions:**
- JLCPCB/LCSC availability and stock
- Board space impact (mm² estimate)
- Component count / BOM cost
- ADC channels, MCU peripherals consumed
- New packages required (`ato create part`)
- Accuracy / precision at operating conditions
- Temperature drift behavior
- Failure modes and detectability
- Firmware complexity
- Reusability across other designs
- Power sequencing / hot-swap implications
- Mechanical constraints (PCB thickness, connector alignment)

Not all criteria apply to every decision — pick the ones that differentiate the options.

### 4. Recommendation

Which option and why, structured as:
- **Rationale:** why the chosen option wins on the criteria that matter most for this specific use case
- **What we're giving up:** explicitly acknowledge the trade-offs accepted
- **When to revisit:** conditions under which the decision should be reconsidered

### Optional: Research Findings

If research uncovered facts that eliminated options or changed the landscape, add a `## Research Findings` section between Comparison and Recommendation. This is for discovered constraints (e.g., "no 0.8mm card-edge power connectors exist on LCSC"), not for general background.

## After a Decision

Update the header:
- Change `**Status:**` from `OPEN` to `DECIDED`
- Add `**Decision:**` line with one-line summary
- Keep the full trade study in the repo as a record — don't delete it

## Workflow

1. **Research** options before writing — check datasheets, LCSC/JLCPCB availability, existing packages, and the atopile registry
2. **Draft** all four sections. Get the comparison table right — this is the core value
3. **Review** with the user before marking DECIDED (trade studies with Status: OPEN are proposals)
4. **Decide** — update status, add decision line
5. **Reference** the trade study from relevant design docs so future readers can find the reasoning
