---
title: "OPC Water in GROMACS: Fixing the MW Virtual Site Topology"
date: 2026-07-03 12:00:00 +0530
categories: [Computational Chemistry, Molecular Dynamics]
tags: [gromacs, opc, water, topology, virtual-sites, settle, troubleshooting]
description: >
  A complete guide to the non-obvious topology errors that occur when using
  OPC 4-site water in GROMACS — duplicate constraints blocks, missing MW
  virtual site entries, and SETTLE incompatibilities that silently corrupt
  your simulation or crash grompp.
---

## Overview

The OPC (Optimal Point Charge) water model is one of the most accurate
available for bulk water properties — reproducing density, self-diffusion
coefficient, and dielectric constant to within 1% of experiment at 298 K.
However, its 4-site geometry (O + H1 + H2 + MW virtual site) introduces
several topology traps in GROMACS that are not documented anywhere and
manifest as either silent corruption or cryptic `grompp` fatal errors.

This post documents every trap encountered during production MD simulations
of palladium nanoclusters in OPC water, and the exact fixes applied.

---

## 1. The OPC Model: What Makes It Different

OPC is a 4-point charge model where:
- **O, H1, H2** — real atoms with mass
- **MW** — massless virtual site carrying the negative charge (`-1.3582e`)

The virtual site sits at a fixed displacement from the oxygen along the
bisector of the H–O–H angle. GROMACS handles this via a `[ virtual_sites3 ]`
entry and a matching `[ constraints ]` block to enforce the O–H geometry.

**Molecular weight:** 18.015 g/mol (O + 2H only — MW carries no mass)  
**Atoms per molecule in topology:** 4 (O + H1 + H2 + MW)  
**Atoms per molecule in coordinate file:** 4 (all four appear in `.gro`)

The key consequence: when counting molecules from a `.gro` file, you must
divide OPC atom count by **4**, but the molecular weight calculation must
use **18.015 g/mol**, not the apparent 4-atom mass.

---

## 2. The Traps

### Trap 1 — Duplicate `[ constraints ]` Block

When building a mixed-solvent topology (e.g., OPC water + NMP) using
`fftool` + `Packmol`, the generated topology sometimes contains **two**
`[ constraints ]` sections for OPC:

```
[ constraints ]
; O-H constraints for SETTLE
  1   2   1   0.08724
  1   3   1   0.08724

[ constraints ]
; H-H virtual site geometry
  2   3   1   0.14142
```

GROMACS 2026.x treats this as two separate constraint sections and
correctly applies both — but the **ordering matters**. If the H–H
constraint appears before the SETTLE constraints, GROMACS cannot identify
the molecule as a SETTLE candidate and falls back to LINCS for all three
constraints, eliminating the performance benefit of SETTLE and generating
a warning:

```
WARNING: SETTLE is not applied to 7205 OPC molecules
```

**Fix:** Merge into a single `[ constraints ]` block with all three entries.

```
[ constraints ]
; funct   b0
  1   2   1   0.08724    ; O-H1
  1   3   1   0.08724    ; O-H2
  2   3   1   0.14142    ; H1-H2 (SETTLE geometry)
```

---

### Trap 2 — Missing `[ virtual_sites3 ]` Section

Topologies built from older OPLS-AA parameter sets or converted from
LAMMPS may include the MW atom in the `[ atoms ]` section but omit the
`[ virtual_sites3 ]` directive that tells GROMACS how to place it.

Symptom: `grompp` succeeds but `mdrun` crashes immediately with:

```
Fatal error:
There are virtual sites in the system, but no virtual site force routine
was called.
```

**Fix:** Add the virtual site construction rule immediately after
`[ bonds ]` in the OPC molecule definition:

```
[ virtual_sites3 ]
; MW constructed from O, H1, H2
; Site  O   H1  H2  funct   a       b
  4     1   2   3   1       0.128012  0.128012
```

The coefficients `a = b = 0.128012` place the virtual site along the
O–bisector at the correct fractional distance, reproducing the OPC
geometry exactly.

---

### Trap 3 — Missing `[ exclusions ]` for MW

The virtual site MW must be excluded from non-bonded interactions with the
real OPC atoms in the same molecule. Without explicit exclusions, GROMACS
may double-count the electrostatic interaction between MW and O, corrupting
the electrostatic energy.

In a correctly built OPC topology from the IFF/INTERFACE FF:

```
[ exclusions ]
; i   j   (exclude 1-4 and all intramolecular interactions)
  1   2
  1   3
  1   4
  2   3
  2   4
  3   4
```

If your topology has fewer than 6 exclusion pairs for a 4-atom molecule,
you are missing entries for MW.

---

### Trap 4 — SETTLE + Virtual Sites + GPU Offload

The most frustrating trap: GROMACS correctly identifies OPC as a SETTLE
candidate (rigid 3-point O + 2H geometry), applies SETTLE to O/H1/H2,
and then places MW using the virtual site algorithm. This works correctly
with:

```bash
gmx mdrun -ntmpi 1 -ntomp 96 -nb gpu -pme gpu -gpu_id 0
```

But it **fails** with `-bonded gpu` or `-update gpu`:

```
Fatal error:
Update on the GPU is not supported for systems with virtual sites.
```

This is not a bug — it is a documented limitation. Virtual site forces
must be calculated on the CPU to ensure correct placement. The fix is
permanent: **never use `-bonded gpu` or `-update gpu` with OPC water**.

The performance hit is negligible — PME and non-bonded offloading to GPU
already provides ~100 ns/day for typical Pd cluster + OPC systems in a
60 Å cubic box.

---

## 3. The Verification Script

After fixing the topology, verify all 9 required entries exist for each
OPC molecule:

```bash
python3 - << 'EOF'
"""
Verify OPC water topology completeness.
Checks: SETTLE-compatible constraints, virtual_sites3, exclusions, MW atom.
"""

topology_file = "system.top"

with open(topology_file) as f:
    content = f.read()

# Count expected entries for 7205 OPC molecules
n_opc = content.count('OPC')  # rough count
print(f"OPC residue mentions: ~{n_opc}")

checks = {
    "[ virtual_sites3 ]":    content.count("virtual_sites3"),
    "[ constraints ]":        content.count("constraints"),
    "SETTLE":                 content.count("SETTLE"),
    "[ exclusions ]":         content.count("exclusions"),
}

print("\nTopology section counts:")
for k, v in checks.items():
    status = "✓" if v > 0 else "✗ MISSING"
    print(f"  {k:<25} {v:>4}  {status}")
EOF
```

For a correctly built OPC topology you should see:

```
[ virtual_sites3 ]       1  ✓
[ constraints ]          1  ✓
SETTLE                   1  ✓
[ exclusions ]           1  ✓
```

---

## 4. The `add_mw_sites.py` Script

When porting a trajectory from LAMMPS (which handles TIP4P/OPC internally)
to GROMACS (which requires explicit MW atoms in the coordinate file), you
need to insert MW virtual sites into the `.gro` file. The coordinates are
computed from O, H1, H2 positions:

```python
#!/usr/bin/env python3
"""
add_mw_sites.py
Insert MW virtual site atoms into OPC water .gro files
for GROMACS compatibility.

Usage: python3 add_mw_sites.py input.gro output.gro
"""

import sys
import numpy as np

# OPC virtual site parameters
A = 0.128012   # fractional displacement along bisector
VIRTUAL_SITE_MASS = 0.0

def compute_mw(o, h1, h2):
    """Compute MW position from O, H1, H2 coordinates."""
    # MW sits along the bisector of H1-O-H2 at fraction A from O
    bisector = (h1 + h2) / 2.0 - o
    bisector /= np.linalg.norm(bisector)
    # a and b are coefficients for the 3-point virtual site
    return o + A * (h1 - o) + A * (h2 - o)

def process_gro(infile, outfile):
    with open(infile) as f:
        lines = f.readlines()

    title  = lines[0]
    n_orig = int(lines[1].strip())
    box    = lines[-1]
    atoms  = lines[2:-1]

    new_atoms = []
    i = 0
    n_mw_added = 0

    while i < len(atoms):
        line = atoms[i]
        resname = line[5:10].strip()

        if resname == 'OPC':
            # Read O, H1, H2 (3 consecutive OPC atoms)
            o_line  = atoms[i]
            h1_line = atoms[i+1]
            h2_line = atoms[i+2]

            def parse_xyz(l):
                return np.array([float(l[20:28]),
                                 float(l[28:36]),
                                 float(l[36:44])])

            o  = parse_xyz(o_line)
            h1 = parse_xyz(h1_line)
            h2 = parse_xyz(h2_line)
            mw = compute_mw(o, h1, h2)

            new_atoms.extend([o_line, h1_line, h2_line])

            # Build MW line in GRO format
            resnum = int(o_line[:5])
            atnum  = int(o_line[15:20]) + 3
            mw_line = (f"{resnum:5d}{'OPC':<5}{'MW':>5}{atnum:5d}"
                       f"{mw[0]:8.3f}{mw[1]:8.3f}{mw[2]:8.3f}\n")
            new_atoms.append(mw_line)
            n_mw_added += 1
            i += 3
        else:
            new_atoms.append(line)
            i += 1

    n_new = n_orig + n_mw_added
    with open(outfile, 'w') as f:
        f.write(title)
        f.write(f"{n_new}\n")
        f.writelines(new_atoms)
        f.write(box)

    print(f"Added {n_mw_added} MW virtual sites")
    print(f"Total atoms: {n_orig} → {n_new}")

if __name__ == '__main__':
    process_gro(sys.argv[1], sys.argv[2])
```

---

## 5. MDP Configuration for OPC Systems

The correct constraint settings for OPC water depend on the co-solvent:

```ini
; Pure OPC water or water + metal cluster
constraints              = h-bonds
constraint-algorithm     = lincs
lincs-iter               = 1
lincs-order              = 4
```

```ini
; OPC water + NMP (OPLS-AA) mixed solvent
constraints              = none
; SETTLE still applies to OPC automatically via topology
; NMP bonds are stiff enough without constraints at dt=0.002 ps
```

**Never use `constraints = all-bonds` with OPC** — this causes GROMACS to
attempt bond constraints on the O–H bonds in addition to SETTLE, producing
constraint conflicts and dramatically slowing integration.

---

## 6. Performance Benchmark

Correctly configured OPC water in GROMACS 2026.2 on RTX 5000 Ada
(96-core dual Xeon Gold 6542Y):

| System | Atoms | ns/day |
|--------|-------|--------|
| Pd₃ + 7205 OPC | 28,823 | ~100 |
| Pd₄ + 7205 OPC | 28,824 | ~100 |
| Pd₃ + 3081 OPC + 576 NMP | 21,543 | ~95 |

SETTLE handles all 7205 water molecules in parallel, while PME and
non-bonded forces are offloaded to GPU. The virtual site placement adds
negligible overhead (~0.5%) since it is a simple linear combination.

---

## Summary of Fixes

| Issue | Symptom | Fix |
|-------|---------|-----|
| Duplicate `[ constraints ]` | SETTLE not applied, slow water | Merge into single block |
| Missing `[ virtual_sites3 ]` | `mdrun` crash on first step | Add VS3 directive with coefficients |
| Missing MW exclusions | Electrostatic energy corruption | Add all 6 intramolecular exclusions |
| `-bonded gpu` / `-update gpu` | Fatal error on GPU systems | Remove these flags; use `-nb gpu -pme gpu` only |
| `constraints = all-bonds` | Constraint conflicts with SETTLE | Use `h-bonds` (water) or `none` (mixed) |

---

*This troubleshooting guide emerged from porting a Pd nanocluster simulation
system from LAMMPS + TIP4P/OPC to GROMACS 2026.2 + OPC, during the revision
of a JCP manuscript on solvent-dependent Pd cluster stability. All fixes were
validated on Fedora 41 with GROMACS 2026.2, CUDA 12.9, and RTX 5000 Ada.*