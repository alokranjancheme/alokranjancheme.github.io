---
title: "Simulation Pd₃ in OPC Water"
date: 2026-06-17 00:00:00 +0530
categories: [Molecular Dynamics, Force Field]
tags: [lammps, fftool, palladium, md-simulation, force-field, packmol]
math: true
---


# GROMACS Pipeline: Pd₃ in OPC Water

## Verified system state

```
config.pdb   — 21,618 ATOM records ✓   (3 Pd + 7205×3 OPC heavy/H sites)
field.top    — 2 × [ moleculetype ]  ✓   (pd3 and OPC)
run.mdp      — fftool template, sd integrator, Berendsen pcoupl
CRYST1       — 60.000 60.000 60.000 ✓
```

---

## Why config.pdb has 21,618 atoms, not 28,823

OPC is a 4-site water model: **O, H1, H2, M** where M is a massless virtual
(dummy) charge site. PDB format cannot store virtual sites with zero mass, so
fftool correctly **omits the M site from config.pdb**. The deficit is exactly
7,205 — one missing M per water molecule:

```
3 Pd  +  7205 × 3 (O+H+H)  =  21,618   ← config.pdb  ✓
3 Pd  +  7205 × 4 (O+H+H+M) =  28,823   ← full system with virtual sites
```

GROMACS **constructs the M site position at runtime** from the O and H
coordinates using the `[ virtual_sites2 ]` section in `field.top`.
You do not need to add M sites manually. This is correct behaviour.

---

## Audit of run.mdp

fftool writes one template `run.mdp` covering all stages. Reading it carefully:

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `integrator` | `sd` | Stochastic dynamics — built-in Langevin thermostat |
| `dt` | `0.001` ps | 1 fs timestep |
| `nsteps` | `10000` | **Only 10 ps** — template placeholder, must increase |
| `tcoupl` | `no` | Correct — `sd` integrator handles temperature, no separate thermostat needed |
| `pcoupl` | `Berendsen` | Pressure coupling **ON** — this is NPT as written |
| `gen-vel` | `yes` | Velocities generated from scratch |
| `continuation` | `no` | Fresh start |
| `rlist/rcoulomb/rvdw` | `1.2` nm | Correct cutoffs for OPC water |
| `DispCorr` | `EnerPres` | Correct long-range LJ correction for a water box |

**What this means for your pipeline:** you will create separate `.mdp` files
for each stage by modifying this template. The key changes per stage are
shown below.

---

## Step 4 — Energy minimisation

Create `em.mdp` from the template with two changes — integrator to `steep`
and pressure coupling off:

```ini
; em.mdp
integrator            = steep
nsteps                = 50000
emtol                 = 100.0
emstep                = 0.01
nstlog                = 500
nstxout-compressed    = 0

cutoff-scheme         = Verlet
rlist                 = 1.2
pbc                   = xyz
coulombtype           = PME
rcoulomb              = 1.2
ewald-rtol            = 1.0e-5
vdwtype               = Cut-off
rvdw                  = 1.2
DispCorr              = EnerPres

; No thermostat or barostat during minimisation
tcoupl                = no
pcoupl                = no

constraints           = h-bonds
constraint-algorithm  = LINCS
```

Run:

```bash
gmx grompp -f em.mdp -c config.pdb -p field.top -o em.tpr -maxwarn 2
gmx mdrun -v -deffnm em
```

Verify convergence:

```bash
grep "Potential Energy" em.log | tail -5
```

---

## Step 5 — NVT equilibration

Create `nvt.mdp`. Keep `sd` integrator (handles thermostat internally),
**turn off** pressure coupling, increase nsteps to 1 ns:

```ini
; nvt.mdp — 1 ns NVT at 300 K
integrator            = sd
dt                    = 0.001
nsteps                = 1000000       ; 1 ns
nstlog                = 1000
nstxout-compressed    = 1000

cutoff-scheme         = Verlet
rlist                 = 1.2
pbc                   = xyz
coulombtype           = PME
rcoulomb              = 1.2
ewald-rtol            = 1.0e-5
vdwtype               = Cut-off
rvdw                  = 1.2
DispCorr              = EnerPres

; sd integrator controls temperature — no separate tcoupl needed
tcoupl                = no
tc-grps               = System
tau-t                 = 1.0
ref-t                 = 300.0

; No pressure coupling in NVT
pcoupl                = no

; Velocities from energy minimised structure
gen-vel               = yes
gen-temp              = 300
gen-seed              = -1

constraints           = h-bonds
constraint-algorithm  = LINCS
continuation          = no
```

Run:

```bash
gmx grompp -f nvt.mdp -c em.gro -p field.top -o nvt.tpr -maxwarn 2
gmx mdrun -v -deffnm nvt
```

Check temperature:

```bash
gmx energy -f nvt.edr -o nvt_temp.xvg
# Select: Temperature
```

---

## Step 6 — NPT equilibration

Create `npt.mdp`. Keep `sd`, enable `Berendsen` barostat (same as fftool
template), switch off velocity generation:

```ini
; npt.mdp — 1 ns NPT at 300 K, 1 bar
integrator            = sd
dt                    = 0.001
nsteps                = 1000000       ; 1 ns
nstlog                = 1000
nstxout-compressed    = 1000

cutoff-scheme         = Verlet
rlist                 = 1.2
pbc                   = xyz
coulombtype           = PME
rcoulomb              = 1.2
ewald-rtol            = 1.0e-5
vdwtype               = Cut-off
rvdw                  = 1.2
DispCorr              = EnerPres

; Temperature via sd
tcoupl                = no
tc-grps               = System
tau-t                 = 1.0
ref-t                 = 300.0

; Pressure — Berendsen for equilibration (matches fftool template)
pcoupl                = Berendsen
pcoupltype            = isotropic
tau-p                 = 0.5
ref-p                 = 1.0
compressibility       = 4.5e-5

; Continue from NVT — no new velocities
gen-vel               = no
continuation          = yes

constraints           = h-bonds
constraint-algorithm  = LINCS
```

Run:

```bash
gmx grompp -f npt.mdp -c nvt.gro -p field.top -o npt.tpr -maxwarn 2
gmx mdrun -v -deffnm npt
```

Check density:

```bash
gmx energy -f npt.edr -o npt_density.xvg
# Select: Density
# Target: ~997 kg/m³ (OPC at 300 K, 1 bar)
```

---

## Step 7 — Production run

Switch barostat from `Berendsen` to `Parrinello-Rahman` for the production
run (Berendsen gives wrong pressure fluctuations for analysis):

```ini
; md.mdp — 100 ns production
integrator            = sd
dt                    = 0.002          ; 2 fs — safe with h-bonds constrained
nsteps                = 50000000       ; 100 ns
nstlog                = 5000
nstxout-compressed    = 5000          ; trajectory every 10 ps

cutoff-scheme         = Verlet
rlist                 = 1.2
pbc                   = xyz
coulombtype           = PME
rcoulomb              = 1.2
ewald-rtol            = 1.0e-5
vdwtype               = Cut-off
rvdw                  = 1.2
DispCorr              = EnerPres

; Temperature via sd
tcoupl                = no
tc-grps               = System
tau-t                 = 1.0
ref-t                 = 300.0

; Pressure — Parrinello-Rahman for production
pcoupl                = Parrinello-Rahman
pcoupltype            = isotropic
tau-p                 = 5.0
ref-p                 = 1.0
compressibility       = 4.5e-5

; Continue from NPT
gen-vel               = no
continuation          = yes

constraints           = h-bonds
constraint-algorithm  = LINCS
```

Run:

```bash
gmx grompp -f md.mdp -c npt.gro -p field.top -o md.tpr -maxwarn 2
gmx mdrun -v -deffnm md
```

---

## Complete command sequence

```bash
# ── Energy minimisation ──────────────────────────────────────────────────
gmx grompp -f em.mdp  -c config.pdb -p field.top -o em.tpr  -maxwarn 2
gmx mdrun   -v -deffnm em

# ── NVT equilibration ────────────────────────────────────────────────────
gmx grompp -f nvt.mdp -c em.gro     -p field.top -o nvt.tpr -maxwarn 2
gmx mdrun   -v -deffnm nvt

# ── NPT equilibration ────────────────────────────────────────────────────
gmx grompp -f npt.mdp -c nvt.gro    -p field.top -o npt.tpr -maxwarn 2
gmx mdrun   -v -deffnm npt

# ── Production ───────────────────────────────────────────────────────────
gmx grompp -f md.mdp  -c npt.gro    -p field.top -o md.tpr  -maxwarn 2
gmx mdrun   -v -deffnm md
```

---

## Expected grompp warnings and how to handle them

| Warning text | Reason | Action |
|-------------|--------|--------|
| `System has non-zero total charge` | Pd atoms q=0, OPC M site q≠0, may sum non-zero | Check `tail -20 em.log` — should be exactly 0.0 for neutral system |
| `1-4 interactions not defined for ...` | Metal cluster has no dihedrals | Normal — harmless, use `-maxwarn 2` |
| `No default Coulomb-14 scaling` | Same reason as above | Same fix |
| `virtual_sites or constraints` note | GROMACS building M sites from O/H | Normal — confirms virtual site section is working |

---

## Key numbers to verify at each stage

| Stage | Quantity | Command | Expected |
|-------|----------|---------|----------|
| After EM | Potential energy | `grep "Potential Energy" em.log \| tail -1` | Negative, converged |
| After NVT | Temperature | `gmx energy -f nvt.edr` → Temperature | 300 ± 5 K |
| After NPT | Density | `gmx energy -f npt.edr` → Density | 990–1005 kg/m³ |
| Production | Box size drift | `gmx energy -f md.edr` → Box-X | Stable ± 0.5 Å |