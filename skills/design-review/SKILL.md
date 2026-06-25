---
name: design-review
description: Use when reviewing or completing an atopile package or board design. Triggers on design review, pre-commit check, "is this ready", quality audit, finishing a package implementation, or verifying a design before build.
---

# Atopile Design Review

> **For agentic workers:** This skill orchestrates four parallel review agents. Do NOT delegate to generic software code-review skills — this is a hardware design review. Each agent uses the atopile LSP (`lsp` skill) tools and reads local datasheets — no internet searches for datasheets.

Dispatches four parallel review agents, each focused on a distinct type of reasoning. For the *why* behind design rules, see the `ato-language` skill's `design-patterns.md`.

## Before Dispatching Agents

1. Identify the `.ato` files under review.
2. **Ensure datasheets are local.** Check the project's local `datasheets/` directory for the relevant IC datasheet and application note. If missing, download them there first (`WebFetch`/`curl` the PDF URL from the manufacturer's site, save as `<manufacturer>_<part>.pdf`). Agents must read from the local copy, not hunt the internet.
3. Extract the module's design parameters (`v_max`, current targets, etc.) from the source.
4. Dispatch all four agents in a single message (parallel execution).

## The Four Review Agents

### Agent 1: Connectivity Audit

**Reasoning type:** Structural — are the right things connected?

Checks:
- Every IC pin that should be connected IS connected (cross-ref datasheet pin table)
- No unintended floating inputs (enable, config, power pins)
- Interface types match across connections (ElectricPower to ElectricPower, not to Electrical)
- `.reference` connected on every `ElectricLogic`
- Pull-ups/pull-downs present where required (I2C, active-low enables, open-drain outputs)
- Reset pins: correct polarity (active-low = pull-up, active-high = pull-down), decoupling cap present
- Config pins (address, mode select, enable) connected via 0-ohm or pull resistors, not tied directly to VCC/GND — allows reconfiguration without rework during early development
- Ground domains explicitly tied
- Bridge syntax used correctly (`~>` only on `can_bridge` modules)
- No external access to `_`-prefixed (private) members of other modules
- Every output action has a feedback/measurement path for automated self-test (ADC, sense line, read-back). Gaps documented in docstring.
- I2C address pins wired via Addressor with `base` matching datasheet, `assert addressor.address is i2c.address`, and `address_lines[].reference` connected
- No duplicate I2C addresses on the same bus (solver does NOT catch this — manual review only)

**Tools:** Read .ato files, `get_hover` to check interface types, `get_references` to trace connections, `get_diagnostics` for type mismatches.

### Agent 2: Datasheet Compliance

**Reasoning type:** Specification — does the design match the IC datasheet?

Checks:
- Read the local datasheet and application note from the project's `datasheets/` directory
- IC lifecycle status is Active (not EOL, NRND, or Last Time Buy) — check manufacturer product page
- Every programming resistor formula is encoded as an `assert` with the correct K-factor
- Assertions have both lower bound (design point) AND upper bound (IC abs max)
- Decoupling caps match datasheet: correct values, count, and which supply pins
- Voltage divider ratios produce the intended threshold voltages
- Both `ratio` and `total_resistance` constrained on every divider
- Operating conditions (voltage, frequency, temperature) within IC rated range

**Tools:** Read .ato files, Read datasheet PDF from the local `datasheets/` directory, `get_hover` to inspect solved values.

### Agent 3: Electrical Ratings

**Reasoning type:** Arithmetic — can every component handle its working conditions?

Checks:
- Every passive has `assert max_voltage >= ...` derived from design params (not magic numbers)
- Every current-carrying resistor has `assert max_power >= ...` with derating
- FET/semiconductor V_DS/V_CE headroom assertions present
- Inductor saturation current covers peak + ripple margin
- Capacitor voltage derating considered (2x for Class II ceramics on HV rails)
- Sense resistor power rating with 2x derating
- All rated param assertions reference module design parameters so they auto-update

**Tools:** Read .ato files, `get_hover` to check constraint values, arithmetic verification.

### Agent 4: Conformance & Build

**Reasoning type:** Convention — does it follow project standards and build clean?

Checks:
- `ato build` passes clean
- `get_diagnostics` returns empty for all files
- Every passive has `.package` locked
- Passives use smallest reasonable package (down to 0402) while meeting ratings — size up only when ratings demand it
- `hide_designators: true` in all build targets
- `package.identifier` set in ato.yaml
- A test/usage harness exists and instantiates all interfaces
- Current-sense amps' common-mode range covers the bus voltage
- Power rails generated/received as the architecture intends (no unintended self-generation)
- Module-level docstring present with design params, operating conditions, and how to re-rate
- Verification status declared in docstring: UNVERIFIED / BUILT / TESTED / PRODUCTION
- Usage example is complete and correct — a future agent must be able to use the module from its docs alone
- Non-obvious assertions have comments explaining the threshold source (datasheet section, calculation)
- Section headers group related circuitry
- BOM review: footprints match expectations

**Tools:** Read .ato files, `ato build`, `get_diagnostics`, read ato.yaml and BOM output.

## Agent Prompt Template

```
You are reviewing an atopile hardware design for correctness. Your focus is [CATEGORY].

IMPORTANT:
- Do NOT use generic software code-review skills.
  This is a hardware design review. Use atopile-specific tools only.
- Do NOT search the internet for datasheets. Use the local copy at the path provided below.

Tools to use:
- Read .ato source files directly
- mcp__atopile-lsp__get_hover — inspect types, members, constraints
- mcp__atopile-lsp__get_diagnostics — parse errors and type mismatches
- mcp__atopile-lsp__get_references — trace connections across files
- Read datasheet PDF from the local path below

Files under review:
- [list .ato files with full paths]

Datasheet (local): datasheets/[filename].pdf
Application note (local): datasheets/[filename].pdf  (if available)

Design parameters:
- [v_max, current targets, etc. from the module]

Confidence scoring (0-100):
- 0: false positive — do not report
- 25: might be real, possibly false positive
- 50: real issue but may be nitpick
- 75: confident, will matter in practice
- 100: certain, confirmed against datasheet or code

Pre-existing issues:
- Still report pre-existing issues — they are often critical and this may be
  the first time anyone has reviewed them properly
- Tag as `[PRE-EXISTING]` so they can be triaged separately
- Apply the same confidence scoring — a pre-existing blocker is still a blocker

Your job:
1. Read the .ato source files and the datasheet
2. [Category-specific checks]
3. For each finding with confidence >= 60, report:
   - BLOCKER / ISSUE / NOTE
   - Confidence: [0-100]
   - File and line number
   - What's wrong
   - Evidence (datasheet page, calculation, or code reference)
   - Suggested fix
4. Do NOT report findings below confidence 60
5. End with a verdict: PASS / PASS WITH ISSUES / FAIL
   and a summary: N blockers, N issues, N notes
```

## Confidence Threshold

Use a lower threshold (≥60) than typical software reviews (≥80) because hardware false negatives are more costly than false positives — a missed issue means a board respin, not a patch.

## After the Review

1. **Present findings to the user** — walk through each BLOCKER and ISSUE one by one. For each, the user decides: fix, accept as risk (with rationale), or defer.
2. **Create a remediation plan** — turn accepted findings into a plan with one task per finding.
3. **Execute the plan** — work through fixes task by task.
4. **Re-run the review** — after fixes, dispatch the four agents again to verify the fixes and catch any regressions. Repeat until PASS.

## Removing or Weakening Requirements During Remediation

When a fix involves **removing** an assertion, constraint, voltage/power rating, package lock, or any other design requirement — even if the removal is correct — document why inline in the commit message AND in a code comment at the removal site.

Every removal must answer:
- **Why was the requirement there?** (original intent)
- **Why is removal safe?** (the specific technical reason it doesn't apply)
- **What still protects against the failure mode?** (the remaining safeguard, if any)

If you can't answer all three, the requirement should be weakened (loosened bounds) rather than removed.

This applies even when the requirement was wrong from the start (e.g., asserting common-mode voltage on a parameter that means differential voltage). The explanation prevents future reviewers from re-adding it without understanding the nuance.

Verdicts:
- **FAIL (any blockers):** must remediate before committing
- **PASS WITH ISSUES:** fix or explicitly accept each issue with rationale
- **PASS:** commit
