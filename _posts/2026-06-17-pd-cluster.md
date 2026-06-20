---
layout: post
title: "Setting Up Pd Nanocluster MD Simulations: What Actually Worked"
date: 2026-06-20
categories: computational-chemistry molecular-dynamics
tags: [gromacs, palladium, opc-water, nmp, lincs, cuda, vmd]
---

# Setting Up Pd Nanocluster MD Simulations: What Actually Worked

This post documents the complete, working setup for molecular dynamics
simulations of palladium nanoclusters (Pd3 and Pd4) solvated in OPC water,
NMP, and mixed water/NMP environments. It covers the build pipeline, the
topology corrections that were actually necessary, a serious constraint
algorithm bug that silently destroyed every Pd-cluster trajectory in the
project until it was found, and the two compute environments used: a
SLURM HPC cluster (PARAM Shakti, IIT Kharagpur) and a local GPU
workstation.

The goal here is to record what *actually worked*, with the reasoning
behind each fix, so the next person debugging a similar rigid-cluster
topology doesn't have to rediscover all of this from scratch.

---

## 1. System Overview

Eleven distinct simulation systems were built, all sharing a common
60×60×60 Å cubic box at 298.15 K, 1 bar:

| System class | Solute | Solvent | Atom count |
|---|---|---|---|
| Single cluster in water | Pd3 or Pd4 | OPC water | 28,823 / 28,824 |
| Two clusters in water | 2×Pd3 or 2×Pd4 | OPC water | 28,706 / 28,708 |
| Single cluster in NMP | Pd3 or Pd4 | NMP | 21,523 / 21,524 |
| Two clusters in NMP | 2×Pd3 or 2×Pd4 | NMP | 21,446 / 21,448 |
| Cluster at water/NMP interface | Pd3 or Pd4 | OPC + NMP (slab split) | 21,543 / 21,544 |
| Two clusters, one per solvent | 2×Pd3 or 2×Pd4 | OPC + NMP | 21,470 / 21,472 |

Twelve systems in total, each built through the same `fftool` →
`Packmol` → topology-correction pipeline, with the same energy
minimisation → NVT → NPT → production sequence.

---

## 2. The Build Pipeline

### 2.1 Tools

- **fftool** — generates per-molecule coordinate files and a raw
  GROMACS topology from z-matrix + force field files
- **Packmol** — packs the molecules into the simulation box according
  to geometric constraints
- **GROMACS 2022.2** (cluster) / **2026.2** (local workstation) — the
  actual MD engine

### 2.2 Force fields

Two force fields were combined in every system:

- **Pd clusters**: Heinz 2008 INTERFACE force field (12-6 LJ,
  parameters fit to bulk Pd properties), with cluster geometries from
  local DFT optimisation
- **OPC water**: the 4-site Izadi & Onufriev (2014) water model
- **NMP**: OPLS-AA, generated directly by fftool from a hand-built
  z-matrix

### 2.3 The standard topology corrections

fftool's raw output needed the same handful of corrections on every
single water-containing system, regardless of solute:

1. **`comb-rule 3` → `comb-rule 2`** in `[ defaults ]`. fftool defaults
   to geometric combining (comb-rule 3), but both the INTERFACE Pd
   parameters and OPC water were fit under Lorentz-Berthelot mixing
   (comb-rule 2, arithmetic σ, geometric ε). Leaving this at 3 silently
   produces the wrong Pd–water cross-term interaction.
2. **OPC's charge moved from the oxygen onto a massless `MW` virtual
   site.** fftool's raw 3-site water output puts the negative charge on
   the oxygen atom directly; real OPC requires a 4th massless site
   offset along the H-O-H bisector.
3. **`[ settles ]` replacing fftool's harmonic water angle.** SETTLE is
   GROMACS's analytic, non-iterative rigid-water solver — far more
   stable than a harmonic angle constraint, and it's what makes OPC
   genuinely rigid rather than springy.
4. **Explicit `[ exclusions ]`** for every atom in the OPC virtual-site
   group (mandatory for SETTLE, since it creates no `[ bonds ]` for
   grompp's automatic exclusion generation to trace).

For NMP-containing systems, the comb-rule correction direction was the
same (3 → 2), needed specifically for the Pd–NMP cross term; NMP's own
intramolecular OPLS-AA parameters (bonds, angles, dihedrals, 1-4 pairs)
were correct as generated and never needed touching.

### 2.4 A real, accepted limitation: two missing NMP dihedrals

fftool consistently reported two missing dihedral types for NMP:
`HC-CT-C-N` and `HC-CT-C-O` — the torsions of the four hydrogens on the
ring's exocyclic CH₂ group, rotating around the C–C(=O) bond. The
*heavy-atom* torsion across that same bond **is** present and correctly
parameterised; only the four hydrogen-specific torsions are absent from
the OPLS-AA parameter file used. This is a genuine, minor force-field
coverage gap — not a build error — and is worth stating explicitly in
any methods section: the rotational barrier across that bond is still
captured by the heavy-atom torsion and 1-4 nonbonded terms, just without
the small additional contribution from the hydrogen-specific terms.

### 2.5 Multi-cluster placement geometry

For every "2×" system, the two clusters were placed along the box's 3D
diagonal, symmetric about the box center, separated by exactly 45 Å:

```
half_offset = 45 / (2 × √3) = 12.9904 Å
Cluster 1: (30 − 12.9904, 30 − 12.9904, 30 − 12.9904)
Cluster 2: (30 + 12.9904, 30 + 12.9904, 30 + 12.9904)
```

This gives >11 Å of clearance from each cluster's exclusion sphere to
the simulation box's periodic boundary, comfortably inside the minimum-
image convention for a 60 Å box with ~12 Å nonbonded cutoffs.

### 2.6 Building a genuine water/NMP interface

The interface systems split the box into two slabs along x — water in
`x ∈ [1.5, 30]`, NMP in `x ∈ [30, 58.5]` — with the Pd cluster centered
exactly on the slab boundary (x = 30). Molecule counts were derived from
each solvent's real bulk density applied to its own half-volume, minus
the volume displaced by the cluster's exclusion sphere at the interface.

One thing worth being explicit about: water and NMP are fully miscible.
A sharp slab interface is a legitimate, deliberately chosen *initial
condition* for studying interdiffusion kinetics — it is not a
persistent feature of the system. With a 60 Å box, the diffusion length
scale exceeds the box half-width within roughly the first 10–15 ns of
simulation, after which the system is better understood as a
homogeneously mixed solution with the Pd cluster inside it, not an
"interface" system anymore. This is worth stating plainly in any write-
up rather than letting the system's name imply something that's only
true for the early trajectory.

For the "one cluster per solvent" variant, the same diagonal placement
used for the multi-cluster water/NMP systems was reused directly — it
happened to put one cluster cleanly inside each slab with ~7.5 Å of
clearance to the boundary, with no additional geometric engineering
needed.

---

## 3. The Constraint-Clique Bug

This was the most consequential finding of the entire project, and it
affected every single Pd3/Pd4 system that had been built up to that
point — on both the HPC cluster and the local workstation, identically.

### 3.1 The symptom

NVT equilibration would fail within the first few steps with severe
LINCS constraint deviation:

```
Step 2, time 0.004 (ps)  LINCS WARNING
relative constraint deviation after LINCS:
rms 0.151703, max 0.151704 (between atoms 2 and 4)
```

A healthy simulation shows LINCS deviations around 10⁻⁵–10⁻⁴. A
deviation of 0.15 is roughly 1,000–3,000× too large. Left running, the
Pd cluster's temperature would climb into the millions of Kelvin and the
simulation box would explode to hundreds of nanometers within a few
hundred steps — full, unambiguous failure, not a recoverable warning.

### 3.2 Ruling out the obvious suspects

The investigation went through several reasonable-seeming causes before
finding the real one:

- **Not a starting-geometry problem.** Direct measurement of the
  EM-minimised Pd-Pd distance (0.2716 nm vs. a 0.2675 nm target, ~1.5%
  off) was well within what LINCS should handle trivially.
- **Not GPU-specific.** The identical failure, at the identical step,
  with the identical magnitude, occurred running the same system
  CPU-only with no GPU flags at all. This ruled out any GPU/CPU force-
  communication or precision issue.
- **Not `lincs-order`.** Raising `lincs-order` from 6 to 8 (a real fix
  for a different, earlier problem — see below) reduced the *frequency*
  of warnings but did not address the underlying instability.

### 3.3 The actual root cause

GROMACS's LINCS solver is not designed for general rigid-body motion.
From the GROMACS reference manual: LINCS "can only be used with bond
constraints and isolated angle constraints" — not closed,
fully-connected constraint networks. A Pd3 cluster with all three
pairwise distances constrained is a closed triangle of constraints; a
Pd4 cluster with all six pairwise distances constrained is a
fully-connected tetrahedron — a constraint "clique" in graph terms. This
is a documented, known failure mode. A GROMACS developer addressing an
essentially identical case (a fully-constrained tetrahedral zinc dummy
model, on the GROMACS mailing list) put it plainly: *"GROMACS is not
built for general rigid-body motion, which is what a fully-constrained
tetrahedron would [require]."*

Notably, raising `lincs-order` to 8 earlier in the project (for a
different system on the HPC cluster) had appeared to fix a similar
symptom — but in hindsight that almost certainly only reduced the
failure *rate*, rather than eliminating the underlying clique problem.
The same topology, run for long enough or on different hardware, was
always going to hit this again.

### 3.4 The fix

A triangle (Pd3, 3 atoms) and a tetrahedron (Pd4, 4 atoms) are both
complete graphs in distance-geometry terms. By the Cayley-Menger
rigidity condition, specifying *all* pairwise distances for 3 or 4
points uniquely and fully determines the 3D shape — no separate angle
terms are needed once every edge length is fixed.

The fix: remove `[ constraints ]` entirely from the Pd moleculetype, and
replace it with `[ bonds ]` on the exact same atom pairs, using a very
stiff harmonic force constant instead of a hard constraint:

```
[ bonds ]
;  ai   aj   func    b0(nm)      kb(kJ/mol/nm^2)
    1    2     1     0.26754     346331.0
    1    3     1     0.26754     346331.0
    1    4     1     0.26754     346331.0
    2    3     1     0.26754     346331.0
    2    4     1     0.26754     346331.0
    3    4     1     0.26754     346331.0
```

The force constant (346,331 kJ/mol/nm²) was chosen to give ~1% RMS
thermal fluctuation in bond length at 298.15 K, with a resulting
vibrational period roughly 39× longer than the 2 fs timestep —
comfortably inside the standard ≥10–20× stability margin for explicit
integration. For reference, this is in the same general range as the
stiffest bonds already present in the project's own NMP topology (the
ring carbonyl C=O bond sits at 476,980 kJ/mol/nm²), so it's not an
unusual value for GROMACS to integrate.

One easy-to-miss trap when applying this fix: **`constraints = all-bonds`
in the mdp file will silently convert the new stiff `[ bonds ]` straight
back into LINCS constraints at grompp time**, recreating the exact same
clique problem through a different code path. The mdp setting needs to
be `constraints = none` (for systems where the Pd cluster sits alongside
OPC water, which is rigid via SETTLE regardless of this setting) or left
at `constraints = h-bonds` (for NMP-containing systems, where it only
auto-constrains genuine hydrogens and never touches the Pd-Pd bonds in
the first place).

### 3.5 Validation

The fix was confirmed clean on two independent system combinations —
different cluster size, different solvent, different mdp constraint
mode:

- **Pd4 in OPC water**, 100 ps (50,000 steps): zero LINCS warnings, zero
  constraint errors, 497.6 ns/day on the local GPU workstation.
- **Pd3 in NMP**, 2 ps (1,000 steps): zero LINCS warnings, zero
  constraint errors, 526.2 ns/day.

Both runs' degrees-of-freedom counts confirmed the topology was
processed correctly — `T-Coupling group pd4 is 12.00` (4 atoms × 3 DOF,
fully flexible, no constraint removing any) versus the old, broken
version's `6.00`.

A small Python conversion script automated the same fix across the
remaining systems, with one important wrinkle: the first version of the
script converted *every* `[ constraints ]` block in a topology file,
which incorrectly swept up NMP's own (correct, necessary) C-H bond
constraints in any system containing both a Pd cluster and NMP. The
corrected version tracks which `[ moleculetype ]` block it's currently
inside and only converts blocks belonging to a molecule literally named
`pd3` or `pd4`, leaving every other molecule's constraints untouched.

---

## 4. PARAM Shakti (HPC Cluster) Setup

### 4.1 Environment

```bash
module load lib/intel/2022/mkl/oneapi-2022.0.2
source /home/apps/gromacs/gromacs-2022.2/installCPUIMPI/bin/GMXRC.bash
```

Binary: `gmx_CPUIMPI` (Intel-MPI build, no thread-MPI layer — `-ntmpi`
and `-ntomp` are invalid flags here). All job execution had to happen
under `/scratch/$USER`; `/home` is not on the fast parallel filesystem.

### 4.2 The grompp segfault saga

A long-running, eventually-inconclusive problem: `gmx_CPUIMPI grompp`
would segfault when invoked inside an `sbatch` batch script, on most but
not all compute nodes, while running successfully via direct interactive
`srun --pty bash` sessions on at least one node. Multiple workarounds
were tried — unsetting SLURM/PMI environment variables before the
grompp call, wrapping it in a subshell, forcing it through
`srun --ntasks=1` — none resolved it. The final, conclusive test was
running `gmx_CPUIMPI --version` with zero arguments on a "broken" node:
it segfaulted immediately, before doing any work at all, proving the
issue lived in node-level library availability (likely inconsistent
mounting of the MKL/oneAPI library paths across compute nodes), not in
anything about the job, the topology, or the launch method.

The practical workaround that kept work moving: generate every `.tpr`
file interactively (`srun --partition=shared --ntasks=1 --pty bash`,
then run grompp directly), and submit only the `mdrun` step via
`sbatch`. This split the pipeline into a manual "prep" step and an
automated "run" step, and reliably avoided the broken code path.

### 4.3 SLURM partition rules

| Partition | Scope | Scale | Max walltime |
|---|---|---|---|
| `shared` | Serial/OpenMP/small tests | 1–36 cores | 3 days |
| `medium` | Production | 1–10 nodes | 3 days |
| `large` | Long-running production | 1–10 nodes | 7 days |

`medium` and `large` both require `#SBATCH --ntasks-per-node=40`
exactly (40 physical cores per node on the standard CPU partition) —
this value is fixed by site policy, not something to tune per job.

---

## 5. Local GPU Workstation Setup

### 5.1 Hardware

- 2× Intel Xeon Gold 6542Y (24 cores/socket, 48 cores / 96 threads
  total), full AVX-512 support including `avx512_bf16` and
  `avx512_fp16`
- NVIDIA RTX 5000 Ada Generation, 32 GB VRAM, CUDA compute capability
  8.9
- 503 GB RAM
- Fedora 41, GCC 14.3.1, CMake 3.30.8

This combination — modern OS, modern compiler, a real GPU, abundant
RAM — made a from-source build dramatically more straightforward than
the HPC cluster's CentOS 7 / GCC 4.8.5 environment.

### 5.2 Build

```bash
cmake .. \
  -DGMX_BUILD_OWN_FFTW=ON \
  -DGMX_GPU=CUDA \
  -DCMAKE_CUDA_ARCHITECTURES=89 \
  -DGMX_SIMD=AVX_512 \
  -DGMX_MPI=OFF \
  -DGMX_THREAD_MPI=ON \
  -DGMX_OPENMP=ON \
  -DCMAKE_INSTALL_PREFIX=$HOME/software/gromacs-install \
  -DCMAKE_BUILD_TYPE=Release
```

`CMAKE_CUDA_ARCHITECTURES=89` targets the exact Ada Lovelace compute
capability natively, avoiding PTX JIT compilation overhead on first
launch. `GMX_THREAD_MPI=ON` (not real MPI) is the correct, GROMACS-
recommended configuration for a single multi-core node with one GPU —
domain decomposition across MPI ranks isn't needed; OpenMP threads
within one thread-MPI rank handle the CPU-side parallelism.

Verified working build output:

```
GPU support:         CUDA
SIMD instructions:   AVX_512
CUDA targets:        89
CUDA driver:         12.90
CUDA runtime:        12.90
```

### 5.3 GPU offload flags — what actually works for this topology class

The obvious choice — offload everything possible to the GPU — turned
out to be wrong for two specific flags, both directly because of the
rigid-cluster topology:

```bash
gmx mdrun -ntmpi 1 -ntomp 96 -nb gpu -pme gpu -gpu_id 0 -v -deffnm nvt
```

- **`-bonded gpu` fails outright**: *"Bonded interactions can not be
  computed on a GPU: None of the bonded types are implemented on the
  GPU."* The system's bonded interactions are dominated by
  `[ constraints ]`/`[ settles ]`/`[ bonds ]`, and GROMACS's GPU bonded
  kernel coverage doesn't include this combination.
- **`-update gpu` fails outright**, for two stated reasons:
  *"Virtual sites are not supported"* (OPC's MW site) and *"Triangle
  constraints are not supported"* (exactly the Pd-cluster clique
  problem, independently flagged by a completely different part of
  GROMACS before the constraint fix above was even applied).

Both conditions apply to every system in this project, so neither flag
can ever be used here, regardless of which specific system is running.
The stable, validated configuration offloads only nonbonded
short-range forces and PME electrostatics to the GPU, leaving bonded
forces, constraints, and the integration step on the CPU's 96 OpenMP
threads — still a substantial speedup over CPU-only, and the
configuration both validation runs above were measured on.

### 5.4 House-keeping environment variables

```bash
export GMX_MAXBACKUP=-1      # overwrite files directly instead of
                              # renaming to #file.N# on every rerun
export GMX_SUPPRESS_DUMP=1   # suppress EM's stepNb.pdb/stepNc.pdb
                              # debug dumps (harmless but voluminous)
```

Both default GROMACS behaviors are reasonable for a one-off run, but
became significant clutter once the same system was being re-run
repeatedly during debugging.

### 5.5 VMD

VMD has no official RPM and gates downloads behind a UIUC registration
form, so installation is manual regardless of platform. The 2.0.x
release line ships without a bundled `install.sh` (unlike the older
1.9.x line's documented `./configure && make install` flow), so the
working install was assembled by hand:

```bash
sudo mkdir -p /opt/vmd2.0.0
sudo cp -r LINUXAMD64 scripts plugins lib shaders /opt/vmd2.0.0/
sudo cp -r lib/scripts/tcl8.6 lib/scripts/tk8.6 /opt/vmd2.0.0/lib/
sudo cp lib/redistrib/lib_LINUXAMD64/liboptix*.so* /opt/vmd2.0.0/optix_only/
sudo cp lib/redistrib/lib_LINUXAMD64/libhdf5*.so* /opt/vmd2.0.0/extra_libs/
```

Two non-obvious fixes were needed beyond simply copying files:

- **Scope `LD_LIBRARY_PATH` narrowly.** Putting VMD's entire bundled
  `lib_LINUXAMD64/` redistributable directory into `LD_LIBRARY_PATH`
  caused an unrelated GNOME desktop service (Tracker, the file
  indexer) to load VMD's bundled `libsqlite3.so.0` instead of the
  system's, crashing with a missing-symbol error completely unrelated
  to VMD itself. The fix was extracting only the specific libraries
  VMD's binary actually needs (OptiX, HDF5) into their own narrow
  directories, rather than exposing the whole redistributable bundle
  globally.
- **The main Tk control window doesn't appear automatically on
  startup**, even though Tk itself initialises without any error.
  Typing `menu main on` at the VMD console (which does appear) opens it
  immediately, confirming the GUI machinery is completely functional —
  only the automatic "show it on launch" step inside the startup script
  is skipped. The practical fix: bake that command into the launcher
  itself, via `-e <(echo "menu main on")` on the command line.

```bash
exec /opt/vmd2.0.0/LINUXAMD64/vmd_LINUXAMD64 -e <(echo "menu main on") "$@"
```

---

## 6. Summary of Validated, Working Parameters

For anyone building a similar Pd-cluster-in-solvent system, here's the
condensed, working parameter set:

**Pd cluster topology** (replace any `[ constraints ]` clique with
this):
- `[ bonds ]`, func 1, on every pairwise atom combination
- `b0` from DFT-optimised geometry (0.26760 nm for Pd3, 0.26754 nm for
  Pd4)
- `kb = 346331.0` kJ/mol/nm²
- No `[ angles ]` needed — fully determined by the bond network alone

**mdp constraint settings**:
- Water-only systems: `constraints = none`
- NMP-containing systems: `constraints = h-bonds`
- `lincs-order = 8` for any remaining genuine LINCS-handled bonds
  (OPC/NMP hydrogens) — though this is now a much smaller concern since
  it no longer needs to handle the Pd cluster at all

**GPU offload** (workstation with GPU):
- `-nb gpu -pme gpu` — yes
- `-bonded gpu -update gpu` — no, both fail outright on this topology
  class, independent of the constraint-clique fix

**Standard nonbonded/electrostatics** (consistent across every system):
- `coulombtype = PME`, `rcoulomb = 1.2`, `pme-order = 4`,
  `fourierspacing = 0.12`
- `vdwtype = Cut-off`, `vdw-modifier = Potential-shift`, `rvdw = 1.2`
- `dt = 0.002` (2 fs), standard for h-bonds-constrained systems

This setup is now running cleanly across all twelve systems, on both
the HPC cluster and the local GPU workstation, with the root-cause
constraint fix validated independently on two different cluster/solvent
combinations.