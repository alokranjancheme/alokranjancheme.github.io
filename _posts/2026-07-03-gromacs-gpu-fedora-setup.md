---
title: "GROMACS 2026.2 with GPU Acceleration on Fedora 41: What Actually Works"
date: 2026-07-03 14:00:00 +0530
categories: [Computational Chemistry, HPC Setup]
tags: [gromacs, gpu, fedora, cuda, rtx5000, avx512, installation, performance]
description: >
  Exact build flags, GPU offload options, and the specific combinations that
  fail silently on GROMACS 2026.2 with RTX 5000 Ada on Fedora 41 —
  so you do not have to rediscover them the hard way.
---

## System Specification

- **OS:** Fedora 41 (kernel 6.x)
- **GPU:** NVIDIA RTX 5000 Ada Generation (SM_8.9, 32 GB VRAM)
- **CPU:** Dual Intel Xeon Gold 6542Y (96 threads total, AVX-512)
- **CUDA:** 12.9
- **GROMACS:** 2026.2
- **Install path:** `~/software/gromacs-install/`

---

## 1. Build Configuration

The CMake flags that produce a working AVX-512 + CUDA build:

```bash
mkdir build && cd build

cmake .. \
  -DCMAKE_INSTALL_PREFIX=$HOME/software/gromacs-install \
  -DGMX_BUILD_OWN_FFTW=ON \
  -DGMX_GPU=CUDA \
  -DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda-12.9 \
  -DGMX_SIMD=AVX_512 \
  -DGMX_MPI=OFF \
  -DGMX_OPENMP=ON \
  -DGMX_DOUBLE=OFF \
  -DCMAKE_BUILD_TYPE=Release

make -j$(nproc)
make install
```

Source the environment in every session:

```bash
source ~/software/gromacs-install/bin/GMXRC
export GMX_MAXBACKUP=-1   # suppress .bak file creation
```

---

## 2. GPU Flag Reference: What Works and What Doesn't

This is the table that took the longest to establish. GROMACS 2026.2 is
strict about what can and cannot be offloaded depending on your system
topology.

### For systems with virtual sites (OPC 4-site water, TIP4P, etc.)

```bash
# WORKS — use this
gmx mdrun -ntmpi 1 -ntomp 96 -nb gpu -pme gpu -gpu_id 0 -v -deffnm production

# FAILS — Fatal error: Update on GPU not supported with virtual sites
gmx mdrun -ntmpi 1 -ntomp 96 -nb gpu -pme gpu -update gpu -gpu_id 0 ...

# FAILS — Fatal error: Bonded interactions on GPU not supported with virtual sites
gmx mdrun -ntmpi 1 -ntomp 96 -nb gpu -pme gpu -bonded gpu -gpu_id 0 ...
```

### For systems with closed constraint networks (rigid Pd clusters)

```bash
# WORKS — stiff bonds instead of constraints (see separate post)
gmx mdrun -ntmpi 1 -ntomp 96 -nb gpu -pme gpu -gpu_id 0 -v -deffnm production

# FAILS — LINCS cannot handle closed constraint rings on GPU
gmx mdrun -ntmpi 1 -ntomp 96 -nb gpu -pme gpu -update gpu -gpu_id 0 ...
```

### For pure small-molecule systems (NMP only, no water, no VS)

```bash
# WORKS — all offloads available
gmx mdrun -ntmpi 1 -ntomp 96 -nb gpu -pme gpu -bonded gpu -update gpu -gpu_id 0 -v -deffnm production
```

**Rule of thumb:** Add flags one at a time. Start with `-nb gpu -pme gpu`.
Only add `-bonded gpu` and `-update gpu` if you have confirmed no virtual
sites and no closed constraint networks in your topology.

---

## 3. Threading Configuration

The RTX 5000 Ada sits alongside 96 CPU threads. The optimal threading
layout for GROMACS is one thread-MPI rank using all OpenMP threads:

```bash
gmx mdrun -ntmpi 1 -ntomp 96 ...
```

Do **not** use multiple thread-MPI ranks on a single GPU — GROMACS cannot
share GPU memory efficiently across ranks and you will see performance
degradation rather than improvement:

```bash
# WORSE — do not do this
gmx mdrun -ntmpi 4 -ntomp 24 ...
```

The performance difference is significant:

| Config | ns/day (Pd₃ + 7205 OPC) |
|--------|--------------------------|
| `-ntmpi 1 -ntomp 96 -nb gpu -pme gpu` | ~100 |
| `-ntmpi 4 -ntomp 24 -nb gpu -pme gpu` | ~65 |
| CPU only `-ntmpi 1 -ntomp 96` | ~12 |

---

## 4. PME Grid Tuning

GROMACS auto-tunes the PME grid at the start of each run. You will see
output like:

```
step 21520: timed with pme grid 48 48 48, coulomb cutoff 1.260: 56.4 M-cycles
              optimal pme grid 48 48 48, coulomb cutoff 1.260
```

Let auto-tuning complete — it typically takes 2,000–5,000 steps. Do not
interrupt the run during this phase. The optimal grid for a 60 Å cubic
box with ~28,000 atoms is typically 48×48×48 to 52×52×52.

If you want to skip auto-tuning for benchmark reproducibility:

```bash
gmx mdrun ... -notunepme -pme gpu
```

---

## 5. Monitoring a Running Simulation

```bash
# Live performance from the log file
tail -f production.log | grep -E "Performance|step [0-9].*will finish"

# Check GPU utilization (separate terminal)
watch -n 5 nvidia-smi

# Quick progress estimate from file size
# Each 100 ns ≈ 500 MB for ~25,000 atom systems at nstxout-compressed=5000
ls -lh production.xtc
```

Expected `nvidia-smi` output during a healthy run:
```
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
| GPU  GI  CI  PID     Type  Process name                GPU Memory Usage    |
|=============================================================================|
|   0  N/A N/A 123456  C     gmx                          1200MiB /  32768MiB|
+-----------------------------------------------------------------------------+
```

GPU memory usage for a typical ~28,000 atom system is 1–2 GB — well within
the 32 GB available on the RTX 5000 Ada.

---

## 6. The `energygrps` Gotcha

If your MDP file contains:

```ini
energygrps               = Pd  OPC
```

You will hit this error on any GPU-offloaded run:

```
Fatal error:
Unrecognized option -nb gpu when using energy groups
(Energy groups require CPU non-bonded calculations)
```

GROMACS cannot offload non-bonded interactions to GPU when energy groups
are active, because GPU non-bonded routines do not decompose energy by group.

**Fix:** Remove `energygrps` from all MDP files used with GPU offloading:

```python
# Automated fix across all MDP files in a directory
import os, re

for root, dirs, files in os.walk('.'):
    for fname in files:
        if fname.endswith('.mdp'):
            path = os.path.join(root, fname)
            with open(path) as f:
                content = f.read()
            fixed = re.sub(r'^energygrps.*\n', '',
                           content, flags=re.MULTILINE)
            if fixed != content:
                with open(path, 'w') as f:
                    f.write(fixed)
                print(f"Fixed: {path}")
```

---

## 7. Duplicate `ref-t` / `tau-t` Error

When MDP files are programmatically generated or patched, you can end up with:

```
ERROR 1 [file production.mdp, line 33]:
  Parameter "tau-t" doubly defined
ERROR 2 [file production.mdp, line 34]:
  Parameter "ref-t" doubly defined
```

This happens because some scripts append new thermostat values without
removing the old ones. The fix:

```python
import re

def clean_mdp_thermostats(path, ref_t=298.15, tau_t=0.1):
    with open(path) as f:
        lines = f.readlines()

    # Detect number of tc-grps
    n_grps = 0
    for line in lines:
        m = re.match(r'^tc-grps\s*=\s*(.+)', line)
        if m:
            n_grps = len(m.group(1).strip().split())
            break

    # Remove all existing ref-t and tau-t lines
    cleaned = [l for l in lines
               if not re.match(r'^(ref[_-]t|tau[_-]t)\b',
                               l.strip(), re.IGNORECASE)]

    # Insert once after tc-grps
    result = []
    inserted = False
    for line in cleaned:
        result.append(line)
        if re.match(r'^tc-grps\s*=', line) and not inserted:
            refs = '  '.join([str(ref_t)] * n_grps)
            taus = '  '.join([str(tau_t)] * n_grps)
            result.append(f'ref_t                 = {refs}\n')
            result.append(f'tau_t                 = {taus}\n')
            inserted = True

    with open(path, 'w') as f:
        f.writelines(result)
    print(f"Cleaned: {path}")

# Apply to all MDP files in current directory
import glob
for mdp in glob.glob('**/*.mdp', recursive=True):
    clean_mdp_thermostats(mdp)
```

---

## 8. Performance Summary

For the Pd nanocluster + OPC water systems in a 60 Å cubic box:

| System | Atoms | ns/day | Notes |
|--------|-------|--------|-------|
| Pd₃ + 7205 OPC | 28,823 | ~100 | SETTLE + GPU non-bonded/PME |
| Pd₄ + 7205 OPC | 28,824 | ~100 | Same config |
| Pd₃ + 1345 NMP | 21,523 | ~110 | No SETTLE, lighter non-bonded |
| Pd₃ + 3081 OPC + 576 NMP | 21,543 | ~95 | Mixed solvent |
| Pd₅₅ + 7198 OPC | 28,847 | ~85 | Larger Pd system, more bonded |
| Pd₅₅ + 1348 NMP | 21,623 | ~95 | — |

All runs use `-ntmpi 1 -ntomp 96 -nb gpu -pme gpu -gpu_id 0`.

Compare to LAMMPS on the same hardware for an equivalent Pd₃ + TIP4P/OPC
system: **~7 ns/day** — a 14× speedup purely from GROMACS's SETTLE +
GPU offload architecture.

---

## Summary Checklist

Before launching any production run on this setup:

- [ ] `source ~/software/gromacs-install/bin/GMXRC`
- [ ] Remove `energygrps` from all MDP files
- [ ] Remove duplicate `ref-t` / `tau-t` lines from MDP files
- [ ] Confirm no `[ constraints ]` closed loops in topology (use stiff bonds)
- [ ] Confirm OPC `[ virtual_sites3 ]` section present
- [ ] Use `-nb gpu -pme gpu` only (no `-bonded gpu`, no `-update gpu`)
- [ ] Use `-ntmpi 1 -ntomp 96` (single rank, all threads)
- [ ] Let PME auto-tuning complete before checking performance

---

*This guide documents the exact configuration used for 90 independent
100 ns production runs (30 systems × 3 replicas) of palladium nanoclusters
in water, NMP, and mixed solvents — totalling ~9 μs of aggregate simulation
time on a single workstation, generated during the revision of a JCP
manuscript on solvent-controlled Pd cluster stability.*