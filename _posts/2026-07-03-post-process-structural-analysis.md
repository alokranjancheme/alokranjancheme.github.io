---
title: "Post-Processing and Structural Analysis Pipeline for GROMACS MD Trajectories"
date: 2026-07-03 23:00:00 +0530
categories: [Computational Chemistry, Analysis]
tags: [gromacs, python, mdanalysis, rdf, msd, diffusion, rmsd, post-processing, structural-analysis]
description: >
  The complete post-processing and structural analysis pipeline for GROMACS
  MD trajectories — extracting thermodynamic properties, RDF, coordination
  number, MSD, diffusion coefficients, RMSD, and solvent shell analysis
  using gmx tools and Python, with the exact GROMACS 2026.2 syntax changes.
---

## Overview

After a production run completes, two categories of analysis are required:

1. **Post-processing** — thermodynamic validation (temperature, pressure,
   density, potential energy) confirming the system is equilibrated
2. **Structural analysis** — physically meaningful quantities (RDF, CN, MSD,
   diffusion, RMSD) characterizing the solvation and dynamics

This post consolidates the complete pipeline used for 90 independent
trajectories of Pd nanocluster systems in water, NMP, and mixed solvents.

---

## 1. Thermodynamic Post-Processing

### 1.1 Extract from Energy File

```bash
#!/bin/bash
# post_process.sh
# Usage: bash post_process.sh <system_name>

SYSTEM="$1"
cd "$SYSTEM"
source ~/software/gromacs-install/bin/GMXRC
export GMX_MAXBACKUP=-1
mkdir -p post_process_raw

# Temperature
echo "Temperature" | gmx energy \
    -f production.edr \
    -o post_process_raw/temperature.xvg \
    -b 0 -e 100000

# Pressure
echo "Pressure" | gmx energy \
    -f production.edr \
    -o post_process_raw/pressure.xvg

# Density
echo "Density" | gmx energy \
    -f production.edr \
    -o post_process_raw/density.xvg

# Potential energy
echo "Potential" | gmx energy \
    -f production.edr \
    -o post_process_raw/potential.xvg

# Kinetic energy
echo "Kinetic-En." | gmx energy \
    -f production.edr \
    -o post_process_raw/kinetic.xvg

echo "Post-processing complete: $SYSTEM"
```

### 1.2 Plot Thermodynamics (Python)

```python
#!/usr/bin/env python3
"""
plot_post_process.py
Usage: python3 plot_post_process.py <system_name>
Reads XVG files from <system_name>/post_process_raw/
Saves figures to <system_name>/post_process_plots/
"""

import sys
import os
import numpy as np
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

SYSTEM = sys.argv[1]
RAW_DIR  = os.path.join(SYSTEM, 'post_process_raw')
PLOT_DIR = os.path.join(SYSTEM, 'post_process_plots')
os.makedirs(PLOT_DIR, exist_ok=True)

WINDOW = 100   # running average window (steps)

QUANTITIES = {
    'temperature': ('Temperature (K)',       'Temperature'),
    'pressure':    ('Pressure (bar)',         'Pressure'),
    'density':     ('Density (kg m⁻³)',       'Density'),
    'potential':   ('Potential Energy (kJ/mol)', 'Potential Energy'),
    'kinetic':     ('Kinetic Energy (kJ/mol)',   'Kinetic Energy'),
}

def read_xvg(path):
    t, y = [], []
    with open(path) as f:
        for line in f:
            if line.startswith(('#', '@')): continue
            cols = line.split()
            if len(cols) >= 2:
                try: t.append(float(cols[0])); y.append(float(cols[1]))
                except: pass
    return np.array(t) / 1000.0, np.array(y)  # ps → ns

def running_avg(y, w):
    return np.convolve(y, np.ones(w)/w, mode='valid')

for key, (ylabel, title) in QUANTITIES.items():
    xvg = os.path.join(RAW_DIR, f'{key}.xvg')
    if not os.path.exists(xvg): continue

    t, y = read_xvg(xvg)
    fig, ax = plt.subplots(figsize=(10, 4))
    ax.plot(t, y, color='#AAAAAA', lw=0.5, alpha=0.7, label='Raw')

    # Running average
    if len(y) > WINDOW:
        ya = running_avg(y, WINDOW)
        ta = t[WINDOW//2 : WINDOW//2 + len(ya)]
        ax.plot(ta, ya, color='#C0392B', lw=2.0, label=f'Avg ({WINDOW} steps)')

    ax.set_xlabel('Time (ns)', fontsize=12)
    ax.set_ylabel(ylabel, fontsize=12)
    ax.legend(fontsize=10)
    ax.grid(True, alpha=0.2)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)

    plt.tight_layout()
    out = os.path.join(PLOT_DIR, f'{key}.png')
    plt.savefig(out, dpi=150, bbox_inches='tight')
    plt.close()
    print(f"  -> {out}")
```

---

## 2. Radial Distribution Function and Coordination Number

### 2.1 GROMACS 2026.2 RDF Syntax

**Important:** GROMACS 2026.x changed `gmx rdf` — `-ref` and `-sel` are
now **selection strings**, not index group numbers. The old syntax
(`echo "1\n2" | gmx rdf`) no longer works.

```bash
#!/bin/bash
# structural_analysis.sh
# Usage: bash structural_analysis.sh <system_name>

SYSTEM="$1"
cd "$SYSTEM"
source ~/software/gromacs-install/bin/GMXRC
export GMX_MAXBACKUP=-1

# Analysis window: last 50 ns of 100 ns production
B=50000   # ps
E=100000  # ps

# ── Pd-O(water) RDF + CN ─────────────────────────────────────
if [[ "$SYSTEM" == *water* || "$SYSTEM" == *interface* ]]; then
    gmx rdf \
        -f production.xtc -s production.tpr \
        -ref "resname pd3 pd4 PDA ICO" \
        -sel "resname OPC and name O" \
        -o rdf_Pd_Ow.xvg \
        -cn rdf_Pd_Ow_cn.xvg \
        -b $B -e $E -bin 0.01 -rmax 1.2 \
        > rdf_Pd_Ow.log 2>&1
    echo "  Pd-Ow RDF done"
fi

# ── Pd-O(NMP) RDF + CN ───────────────────────────────────────
if [[ "$SYSTEM" == *nmp* || "$SYSTEM" == *interface* ]]; then
    gmx rdf \
        -f production.xtc -s production.tpr \
        -ref "resname pd3 pd4 PDA ICO" \
        -sel "resname NMP and name O" \
        -o rdf_Pd_Onmp.xvg \
        -cn rdf_Pd_Onmp_cn.xvg \
        -b $B -e $E -bin 0.01 -rmax 1.2 \
        > rdf_Pd_Onmp.log 2>&1
    echo "  Pd-O(NMP) RDF done"
fi

# ── MSD for solvent diffusion ─────────────────────────────────
# Build index with OPC oxygen group
printf "a O & r OPC\nname 4 OPC_O\nq\n" | gmx make_ndx \
    -f em.gro -o msd.ndx > /dev/null 2>&1

# Note: -skip flag removed in GROMACS 2026.x — use -dt instead
if [[ "$SYSTEM" == *water* || "$SYSTEM" == *interface* ]]; then
    echo "4" | gmx msd \
        -f production.xtc -s production.tpr \
        -n msd.ndx \
        -o msd_OPC_O.xvg \
        -mol -trestart 200 \
        -b $B -e $E \
        > msd_OPC_O.log 2>&1
    echo "  OPC-O MSD done"
fi

if [[ "$SYSTEM" == *nmp* || "$SYSTEM" == *interface* ]]; then
    echo "3" | gmx msd \
        -f production.xtc -s production.tpr \
        -n msd.ndx \
        -o msd_NMP.xvg \
        -mol -trestart 200 \
        -b $B -e $E \
        > msd_NMP.log 2>&1
    echo "  NMP MSD done"
fi

# Pd MSD
echo "2" | gmx msd \
    -f production.xtc -s production.tpr \
    -n msd.ndx \
    -o msd_Pd.xvg \
    -mol -trestart 200 \
    -b $B -e $E \
    > msd_Pd.log 2>&1
echo "  Pd MSD done"
```

### 2.2 Extract Coordination Number at Cutoff

```python
import numpy as np
import os

def get_cn_at_cutoff(cn_xvg, cutoff_ang=4.5):
    """
    Read cumulative CN xvg and return CN at specified cutoff.

    Parameters
    ----------
    cn_xvg     : str   — path to gmx rdf -cn output file
    cutoff_ang : float — cutoff in Angstrom (default 4.5 for Pd-O)

    Returns
    -------
    cn : float — coordination number at cutoff
    """
    r, cn = [], []
    with open(cn_xvg) as f:
        for line in f:
            if line.startswith(('#', '@')): continue
            cols = line.split()
            if len(cols) >= 2:
                try:
                    r.append(float(cols[0]) * 10)  # nm → Å
                    cn.append(float(cols[1]))
                except: pass
    r  = np.array(r)
    cn = np.array(cn)
    if len(r) == 0: return None
    idx = np.argmin(np.abs(r - cutoff_ang))
    return float(cn[idx])

# Example usage
systems = ['pd3_water', 'pd4_water', 'pd3_nmp', 'pd4_nmp',
           'pd3_interface', 'pd4_interface']

print(f"{'System':<20} {'CN(Pd-Ow)':>10} {'CN(Pd-Onmp)':>12}")
print("─" * 45)
for sys in systems:
    cn_ow  = get_cn_at_cutoff(f'{sys}/rdf_Pd_Ow_cn.xvg')   \
             if os.path.exists(f'{sys}/rdf_Pd_Ow_cn.xvg')   else None
    cn_nmp = get_cn_at_cutoff(f'{sys}/rdf_Pd_Onmp_cn.xvg') \
             if os.path.exists(f'{sys}/rdf_Pd_Onmp_cn.xvg') else None
    ow_s  = f"{cn_ow:.2f}"  if cn_ow  else "—"
    nmp_s = f"{cn_nmp:.2f}" if cn_nmp else "—"
    print(f"  {sys:<18} {ow_s:>10} {nmp_s:>12}")
```

---

## 3. Diffusion Coefficients from MSD

GROMACS 2026.2 no longer prints D to the log when using `gmx msd -mol`.
Extract D from the MSD slope in Python:

```python
def calc_diffusion(xvg_file, fit_frac=0.5):
    """
    Extract diffusion coefficient D from MSD slope.
    Uses last fit_frac of the data for linear fitting.
    MSD = 6*D*t (3D isotropic diffusion).

    Returns D in cm²/s.
    """
    t, msd = [], []
    with open(xvg_file) as f:
        for line in f:
            if line.startswith(('#', '@')): continue
            cols = line.split()
            if len(cols) >= 2:
                try:
                    t.append(float(cols[0]))    # ps
                    msd.append(float(cols[1]))  # nm²
                except: pass

    if len(t) < 10: return None

    t   = np.array(t)
    msd = np.array(msd)
    n   = len(t)
    i0  = int(n * (1 - fit_frac))

    slope, _ = np.polyfit(t[i0:], msd[i0:], 1)
    D_nm2ps  = slope / 6.0          # nm²/ps
    D_cm2s   = D_nm2ps * 1e-2       # cm²/s
    return D_cm2s

# Print diffusion table
print(f"\n{'System':<20} {'D(Pd)':>12} {'D(OPC-O)':>12} {'D(NMP)':>12}")
print("─" * 60)
for sys in systems:
    d_pd  = calc_diffusion(f'{sys}/msd_Pd.xvg')
    d_ow  = calc_diffusion(f'{sys}/msd_OPC_O.xvg')
    d_nmp = calc_diffusion(f'{sys}/msd_NMP.xvg')
    print(f"  {sys:<18} "
          f"{d_pd:.3e if d_pd else '—':>12} "
          f"{d_ow:.3e if d_ow else '—':>12} "
          f"{d_nmp:.3e if d_nmp else '—':>12}")
```

**Reference values at 298 K:**

| Species | Experimental D (cm²/s) | Simulated (this work) |
|---------|------------------------|----------------------|
| OPC water | 2.30×10⁻⁵ (Izadi 2014) | 2.26–2.34×10⁻⁵ ✅ |
| NMP | ~3–4×10⁻⁶ (Stokes-Einstein) | 3.34–3.39×10⁻⁶ ✅ |

---

## 4. RDF Overlay Plot: 298 K vs 350 K

```python
def plot_rdf_comparison(sys298, sys350, rdf_file, title, out_file):
    """
    Overlay RDF at two temperatures for thermal stability comparison.
    """
    def read_rdf(path):
        r, g = [], []
        with open(path) as f:
            for line in f:
                if line.startswith(('#','@')): continue
                cols = line.split()
                if len(cols) >= 2:
                    try:
                        r.append(float(cols[0]) * 10)  # nm → Å
                        g.append(float(cols[1]))
                    except: pass
        return np.array(r), np.array(g)

    f298 = os.path.join(sys298, rdf_file)
    f350 = os.path.join(sys350, rdf_file)
    if not (os.path.exists(f298) and os.path.exists(f350)):
        return

    r298, g298 = read_rdf(f298)
    r350, g350 = read_rdf(f350)

    fig, ax = plt.subplots(figsize=(7, 4))
    ax.plot(r298, g298, color='#2980B9', lw=2.0, label='298 K')
    ax.plot(r350, g350, color='#E74C3C', lw=2.0, ls='--', label='350 K')

    ax.set_xlabel("r (Å)", fontsize=12)
    ax.set_ylabel("g(r)", fontsize=12)
    ax.legend(fontsize=11)
    ax.grid(True, alpha=0.3)
    ax.set_xlim(0, 12)
    ax.set_ylim(bottom=0)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)

    plt.tight_layout()
    plt.savefig(out_file, dpi=150, bbox_inches='tight')
    plt.close()
    print(f"  -> {out_file}")

# Generate all comparison plots
RDF_PAIRS = [
    ('pd3_water',     'pd3_water_350K',     'rdf_Pd_Ow.xvg',
     'Pd₃ + Water: Pd–Ow RDF'),
    ('pd4_water',     'pd4_water_350K',     'rdf_Pd_Ow.xvg',
     'Pd₄ + Water: Pd–Ow RDF'),
    ('pd3_nmp',       'pd3_nmp_350K',       'rdf_Pd_Onmp.xvg',
     'Pd₃ + NMP: Pd–O(NMP) RDF'),
    ('pd4_nmp',       'pd4_nmp_350K',       'rdf_Pd_Onmp.xvg',
     'Pd₄ + NMP: Pd–O(NMP) RDF'),
    ('pd3_interface', 'pd3_interface_350K', 'rdf_Pd_Onmp.xvg',
     'Pd₃ + Interface: Pd–O(NMP) RDF'),
    ('pd4_interface', 'pd4_interface_350K', 'rdf_Pd_Onmp.xvg',
     'Pd₄ + Interface: Pd–O(NMP) RDF'),
]

for sys298, sys350, rdf_file, title in RDF_PAIRS:
    tag = rdf_file.replace('.xvg','').replace('rdf_','')
    out = f"plots/rdf_350K_{sys298}_{tag}.png"
    plot_rdf_comparison(sys298, sys350, rdf_file, title, out)
```

---

## 5. RMSD Analysis

```bash
# RMSD of Pd cluster relative to initial structure
echo "2 2" | gmx rms \
    -f production.xtc \
    -s production.tpr \
    -n index.ndx \
    -o rmsd_Pd.xvg \
    -b 0 -e 100000

# RMSD of solvent (drift indicator)
echo "3 3" | gmx rms \
    -f production.xtc \
    -s production.tpr \
    -n index.ndx \
    -o rmsd_solvent.xvg \
    -b 0 -e 100000
```

```python
def plot_rmsd(xvg_files, labels, out_file):
    fig, ax = plt.subplots(figsize=(10, 4))

    colors = ['#2980B9', '#E74C3C', '#27AE60', '#E67E22', '#8E44AD']

    for i, (xvg, label) in enumerate(zip(xvg_files, labels)):
        if not os.path.exists(xvg): continue
        t, rmsd = [], []
        with open(xvg) as f:
            for line in f:
                if line.startswith(('#','@')): continue
                cols = line.split()
                if len(cols) >= 2:
                    try:
                        t.append(float(cols[0])/1000)       # ps → ns
                        rmsd.append(float(cols[1])*10)      # nm → Å
                    except: pass
        ax.plot(t, rmsd, color=colors[i % len(colors)],
                lw=1.5, label=label)

    ax.set_xlabel("Time (ns)", fontsize=12)
    ax.set_ylabel("RMSD (Å)", fontsize=12)
    ax.legend(fontsize=10)
    ax.grid(True, alpha=0.2)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)
    plt.tight_layout()
    plt.savefig(out_file, dpi=150, bbox_inches='tight')
    plt.close()
```

---

## 6. Complete Structural Analysis Script

```python
#!/usr/bin/env python3
"""
plot_structural_analysis.py
Runs all structural analysis plots for all single-cluster systems.
Usage: python3 plot_structural_analysis.py
"""

import os
import numpy as np
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

HPC      = os.path.expanduser("~/Downloads/A_Ranjan/gromacs_hpc")
PLOT_DIR = os.path.join(HPC, "plots")
os.makedirs(PLOT_DIR, exist_ok=True)

SYSTEMS = ['pd3_water','pd4_water','pd3_nmp',
           'pd4_nmp','pd3_interface','pd4_interface']

# 1. Diffusion coefficients
print("=== Diffusion Coefficients ===")
print(f"{'System':<20} {'D_Pd':>12} {'D_OPC-O':>12} {'D_NMP':>12}")
print("─" * 60)
for sys in SYSTEMS:
    d_pd  = calc_diffusion(f'{HPC}/{sys}/msd_Pd.xvg')
    d_ow  = calc_diffusion(f'{HPC}/{sys}/msd_OPC_O.xvg')
    d_nmp = calc_diffusion(f'{HPC}/{sys}/msd_NMP.xvg')
    pd_s  = f"{d_pd:.3e}"  if d_pd  else "—"
    ow_s  = f"{d_ow:.3e}"  if d_ow  else "—"
    nmp_s = f"{d_nmp:.3e}" if d_nmp else "—"
    print(f"  {sys:<18} {pd_s:>12} {ow_s:>12} {nmp_s:>12}")

# 2. Coordination numbers
print("\n=== Coordination Numbers (cutoff 4.5 Å) ===")
print(f"{'System':<20} {'CN(Pd-Ow)':>12} {'CN(Pd-Onmp)':>14}")
print("─" * 48)
for sys in SYSTEMS:
    cn_ow  = get_cn_at_cutoff(f'{HPC}/{sys}/rdf_Pd_Ow_cn.xvg')
    cn_nmp = get_cn_at_cutoff(f'{HPC}/{sys}/rdf_Pd_Onmp_cn.xvg')
    ow_s  = f"{cn_ow:.2f}"  if cn_ow  else "—"
    nmp_s = f"{cn_nmp:.2f}" if cn_nmp else "—"
    print(f"  {sys:<18} {ow_s:>12} {nmp_s:>14}")

print("\nDone. All plots saved to plots/")
```

---

## 7. Key GROMACS 2026.2 Syntax Changes

| Feature | Old syntax (≤2023) | New syntax (2026.2) |
|---------|-------------------|---------------------|
| `gmx rdf` reference | `echo "1" \| gmx rdf -ref 1` | `gmx rdf -ref "resname pd3"` |
| `gmx rdf` selection | `echo "2" \| gmx rdf -sel 2` | `gmx rdf -sel "resname OPC and name O"` |
| `gmx msd` skip | `gmx msd -skip 5` | Removed — use `-dt` instead |
| `gmx msd` diffusion | Printed to log | Must extract from slope |
| Energy groups + GPU | Works | Incompatible — remove `energygrps` |

---

*This pipeline was applied to 90 independent 100 ns trajectories across
30 Pd₅₅ system configurations and 6 single-cluster reference systems,
generating all thermodynamic and structural analysis figures for a JCP
manuscript major revision on solvent-dependent Pd nanocluster stability.*