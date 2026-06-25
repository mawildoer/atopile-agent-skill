---
name: layout-relinking
description: Use when renaming or refactoring components in .ato files that have existing PCB layout placements. Triggers on "layout broken", "components at origin", "lost placement after rename", "relink layout", or any .ato refactor that changes component names or module structure.
---

# Layout Relinking

When you rename a component in `.ato` code, its `atopile_address` in the `.kicad_pcb` becomes stale. On the next build, atopile can no longer match the placed footprint to the renamed component, so the component loses its placement (moved to origin, unrouted).

> **Critical: relink BEFORE building.** If you build with stale addresses, atopile orphans the old placements and creates new unplaced footprints. Relinking after that means recovering from backups.

## The Underlying Concept

Each footprint in the `.kicad_pcb` carries an `atopile_address` property — the dot-path of instance names from the build entry point down to that component. If `Board` has `charger = new BQ25756Charger` and `BQ25756Charger` has `_richg = new Resistor`, the address is `charger._richg`. Renaming any instance in that chain changes the address; relinking rewrites the stored address to match the new path so the placement is preserved.

## Helper Scripts

This workflow relies on two small scripts that operate on the `.kicad_pcb`. Many projects vendor them under `tools/`; if yours doesn't, they're trivial to write (parse the s-expression / KiCad file, read or rewrite the `atopile_address` property on each footprint):

- **list-addresses** — print every `atopile_address` in a `.kicad_pcb` (optionally with the footprint each maps to).
- **rename-address** — rewrite one address prefix to another. Should be **prefix-aware**: renaming a parent catches all children automatically.

The commands below assume `tools/layout-list-addresses.py` and `tools/layout-rename-address.py`. Substitute your project's equivalents.

## Workflow

1. **Make your .ato changes** (renames, submodule extraction, etc.) — do NOT build yet.

2. **Examine what changed:**
   ```bash
   git diff -- "*.ato"
   ```

3. **List current PCB addresses:**
   ```bash
   python tools/layout-list-addresses.py <pcb_file>
   ```

4. **Figure out the mapping.** Construct new paths from the module hierarchy in the `.ato` source. The address path is the chain of instance names from the build entry point down.

   Note: the LSP tools (`get_hover`, `get_definition`, `get_references`) do NOT provide `atopile_address` paths. Use `git diff` and the `.ato` source to determine mappings.

5. **Relink each renamed component:**
   ```bash
   python tools/layout-rename-address.py <pcb_file> "old.path" "new.path"
   ```
   A prefix-aware tool renames all children of a renamed parent in one call.

6. **Now run `ato build`** — addresses match, placements preserved.

7. **Verify:**
   ```bash
   python tools/layout-list-addresses.py <pcb_file>
   ```

## Examples

Extracting a submodule — moving `charger.richg` into `charger._sense._shunt`:
```bash
python tools/layout-rename-address.py <pcb> "charger.richg" "charger._sense._shunt"
```

Renaming a parent module (catches all children):
```bash
python tools/layout-rename-address.py <pcb> "charger.acov_div" "charger._acov_divider"
# Renames charger.acov_div.chain.resistors[0], [1], etc. automatically
```

## Tips

- Back up the `.kicad_pcb` (or rely on a tool that writes a `.bak` on first modification) before bulk renames — restore with `cp file.bak file`.
- A "no matches" result means the old path is wrong — check for typos against the address listing.
- List addresses with the mapped footprint to confirm which physical part each address points to.
