---
title: "COM-COM Distance Analysis with PBC Correction for Dual-Cluster MD Systems"
date: 2026-07-03 20:00:00 +0530
categories: [Computational Chemistry, Analysis]
tags: [mdanalysis, pbc, com, cluster, aggregation, python, gromacs, distance]
description: >
  Why naive inter-cluster distance calculations give wrong results near
  periodic box boundaries — and the exact MDAnalysis pipeline using
  minimize_vectors to compute PBC-correct centre-of-mass distances,
  moving averages, and aggregation timescales from GROMACS trajectories.
---

## The Problem: Naive Distance is Wrong Near Box Boundaries

When two clusters are placed in a periodic simulation box, the apparent
distance between their centres-of-mass can be misleading. Consider two
clusters separated by 2 nm in a 6 nm cubic box — their direct distance is
2 nm, but the periodic image distance is also 4 nm. GROMACS wraps coordinates
into the primary box, so a cluster near the +x face is only 0.1 nm from a
cluster near the -x face via the periodic image — but the raw coordinate
difference gives 5.9 nm.

For aggregation analysis, you always want the **minimum image distance** —
the shortest path between clusters accounting for all periodic images. Using
the raw coordinate difference gives nonsensical spikes in the distance trace
wherever clusters cross box boundaries.

---

## 1. The Correct Tool: `minimize_vectors`

MDAnalysis provides `minimize_vectors` which applies the minimum image
convention to an array of displacement vectors:

```python
from MDAnalysis.lib.distances import minimize_vectors
import numpy as np

def com_com_distance_pbc(sel1, sel2, box):
    """
    PBC-correct COM-COM distance between two atom selections.

    Parameters
    ----------
    sel1, sel2 : MDAnalysis AtomGroup
    box        : np.ndarray shape (6,) — [a, b, c, alpha, beta, gamma]

    Returns
    -------
    distance : float, Angstrom
    """
    com1 = sel1.center_of_mass()
    com2 = sel2.center_of_mass()
    vec  = com2 - com1
    vec  = minimize_vectors(vec, box)   # minimum image correction
    return float(np.linalg.norm(vec))
```

The `minimize_vectors` call wraps the displacement vector into the primary
box, ensuring the returned distance is always the shortest possible path
between the two COMs across all periodic images.

---

## 2. Identifying the Two Clusters

For dual-cluster systems (two Pd₃ or two Pd₄ clusters in the same box),
the clusters must be selected by **residue index** rather than resname —
both clusters share the same residue name:

```python
import MDAnalysis as mda

u   = mda.Universe('2pd3_nmp/em.gro', '2pd3_nmp/production.xtc')
pd  = u.select_atoms('resname pd3')

# Get unique residue IDs
resids = sorted(set(pd.resids))
print(f"Pd residue IDs: {resids}")   # e.g. [1, 2]

# Select each cluster independently
cluster1 = u.select_atoms(f'resname pd3 and resid {resids[0]}')
cluster2 = u.select_atoms(f'resname pd3 and resid {resids[-1]}')

print(f"Cluster 1: {len(cluster1)} atoms")
print(f"Cluster 2: {len(cluster2)} atoms")
```

Verify both selections have the correct atom count (3 for Pd₃, 4 for Pd₄)
before proceeding.

---

## 3. Full Trajectory Analysis

```python
from MDAnalysis.lib.distances import minimize_vectors
import numpy as np

def analyze_com_com(gro, xtc, resname, skip=2):
    """
    Extract PBC-correct COM-COM distance over an entire trajectory.

    Parameters
    ----------
    gro     : str — path to structure file
    xtc     : str — path to trajectory file
    resname : str — residue name of the metal clusters (e.g. 'pd3')
    skip    : int — analyse every skip-th frame (default=2, ~20 ps)

    Returns
    -------
    times : np.ndarray — time in ns
    dists : np.ndarray — COM-COM distance in nm
    """
    u      = mda.Universe(gro, xtc)
    pd     = u.select_atoms(f'resname {resname}')
    resids = sorted(set(pd.resids))

    if len(resids) < 2:
        raise ValueError(f"Only found {len(resids)} residue(s) — "
                         f"need at least 2 clusters.")

    sel1 = u.select_atoms(f'resname {resname} and resid {resids[0]}')
    sel2 = u.select_atoms(f'resname {resname} and resid {resids[-1]}')

    times, dists = [], []

    for ts in u.trajectory[::skip]:
        box  = ts.dimensions
        com1 = sel1.center_of_mass()
        com2 = sel2.center_of_mass()
        vec  = com2 - com1
        vec  = minimize_vectors(vec, box)
        d_nm = np.linalg.norm(vec) / 10.0   # Å → nm

        times.append(ts.time / 1000.0)       # ps → ns
        dists.append(d_nm)

    return np.array(times), np.array(dists)
```

---

## 4. Moving Average for Aggregation Detection

The raw COM-COM trace is noisy — thermal fluctuations cause the distance
to oscillate ±0.5 nm even when clusters are in contact. A moving average
reveals the underlying aggregation state:

```python
def moving_average(data, window):
    """Simple boxcar moving average."""
    kernel = np.ones(window) / window
    return np.convolve(data, kernel, mode='valid')

# Usage
times, dists = analyze_com_com('2pd3_nmp/em.gro',
                                '2pd3_nmp/production.xtc', 'pd3')

# Window of 500 frames ≈ 5 ns smoothing
window  = 500
ma      = moving_average(dists, window)
t_ma    = times[window//2 : window//2 + len(ma)]
```

The moving average window should be large enough to suppress thermal noise
but small enough to resolve the aggregation event. For 10 ps/frame
trajectories, a 500-frame window (5 ns) works well for Pd₃/Pd₄ systems.

---

## 5. Aggregation Time Detection

The first aggregation event is defined as the first time the moving average
COM-COM distance drops below a threshold and **stays** below it for a
sustained period:

```python
def find_aggregation_time(times, dists, threshold=0.5, window=100):
    """
    Find first sustained aggregation event.

    Parameters
    ----------
    threshold : float, nm — COM-COM distance defining "aggregated"
    window    : int — minimum number of consecutive frames below threshold

    Returns
    -------
    t_agg : float, ns — aggregation time, or None if not observed
    """
    for i in range(len(dists) - window):
        if np.mean(dists[i:i+window]) < threshold:
            return times[i]
    return None

t_agg = find_aggregation_time(t_ma, ma)
if t_agg:
    print(f"First aggregation at: {t_agg:.1f} ns")
else:
    print("No sustained aggregation observed")
```

The 0.5 nm threshold is physically motivated — it corresponds to roughly
twice the Pd–Pd bond length, ensuring both clusters are in direct contact
rather than merely nearby.

---

## 6. Publication Plot

```python
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

PHASE_COLORS = {
    'NVT':        '#AED6F1',
    'NPT':        '#A9DFBF',
    'Production': '#FAD7A0',
}

def plot_com_com(times, dists, t_ma, ma, sys_name, out_file,
                 phases=None):
    """
    Publication-quality COM-COM distance plot.
    No title. Clean axes. Inset shows NVT+NPT equilibration.
    """
    fig, ax = plt.subplots(figsize=(10, 4))
    fig.subplots_adjust(left=0.09, right=0.98, top=0.92, bottom=0.14)

    # Phase shading
    if phases:
        for phase, t0, t1 in phases:
            ax.axvspan(t0, t1, alpha=0.30,
                       color=PHASE_COLORS.get(phase, '#DDDDDD'),
                       zorder=0, lw=0)

    # Raw trace + moving average
    ax.plot(times, dists, color='#555555', lw=0.5, alpha=0.45, zorder=2)
    ax.plot(t_ma,  ma,    color='#C0392B', lw=2.2, zorder=3)

    # Equilibration zoom inset (top-right corner)
    axins = ax.inset_axes([0.72, 0.45, 0.26, 0.50])
    equil_mask = times <= 5.0
    axins.plot(times[equil_mask], dists[equil_mask],
               color='#555555', lw=0.7, alpha=0.5)
    axins.set_xlim(0, 5.0)
    axins.set_ylim(bottom=0)
    axins.set_xlabel("")
    axins.set_ylabel("")
    axins.set_xticks([])
    axins.set_yticks([])
    axins.set_title("NVT + NPT", fontsize=7, pad=2, style='italic')
    axins.patch.set_alpha(0.80)

    ax.set_xlabel("Time (ns)", fontsize=12)
    ax.set_ylabel("Distance (nm)", fontsize=12)
    ax.set_xlim(0, times[-1] * 1.01)
    ax.set_ylim(bottom=0)
    ax.tick_params(labelsize=10)
    ax.grid(True, alpha=0.2, lw=0.5)
    ax.spines['top'].set_visible(False)

    plt.savefig(out_file, dpi=150)
    plt.close()
    print(f"Saved: {out_file}")
```

---

## 7. Batch Processing All Dual-Cluster Systems

```python
import os

HPC      = os.path.expanduser("~/Downloads/A_Ranjan/gromacs_hpc")
PLOT_DIR = os.path.join(HPC, "plots")
os.makedirs(PLOT_DIR, exist_ok=True)

SYSTEMS = {
    '2pd3_water':     'pd3',
    '2pd4_water':     'pd4',
    '2pd3_nmp':       'pd3',
    '2pd4_nmp':       'pd4',
    '2pd3_interface': 'pd3',
    '2pd4_interface': 'pd4',
}

for sysdir, resname in SYSTEMS.items():
    gro = os.path.join(HPC, sysdir, 'em.gro')
    xtc = os.path.join(HPC, sysdir, 'production.xtc')
    if not os.path.exists(xtc):
        print(f"SKIP {sysdir}: no xtc")
        continue

    print(f"Processing {sysdir}...")
    times, dists = analyze_com_com(gro, xtc, resname, skip=2)

    # Moving average
    win  = 500
    ma   = moving_average(dists, win)
    t_ma = times[win//2 : win//2 + len(ma)]

    # Aggregation time
    t_agg = find_aggregation_time(t_ma, ma)
    print(f"  Aggregation: {t_agg:.1f} ns" if t_agg
          else "  No aggregation")

    # Save xvg for reference
    xvg_out = os.path.join(HPC, sysdir, 'com_com.xvg')
    with open(xvg_out, 'w') as f:
        f.write('# COM-COM distance (nm) vs time (ns)\n')
        for t, d in zip(times, dists):
            f.write(f'{t:.3f}  {d:.4f}\n')

    # Plot
    out = os.path.join(PLOT_DIR, f"com_com_{sysdir}.png")
    plot_com_com(times, dists, t_ma, ma, sysdir, out)
```

---

## 8. Scientific Results

Aggregation timescales extracted from six dual-cluster systems (200 ns
for NMP and interface systems, 100 ns for water systems):

| System | Aggregation time | Post-aggregation behavior |
|--------|-----------------|--------------------------|
| 2Pd₃ + Pure Water | 6.9 ns | Permanent — no escape events |
| 2Pd₄ + Pure Water | 5.2 ns | Permanent — no escape events |
| 2Pd₃ + Interface 1:1 v/v | 21.9 ns | Persistent with transient escapes |
| 2Pd₄ + Interface 1:1 v/v | 16.7 ns | Progressive destabilization 75–200 ns |
| 2Pd₃ + Pure NMP | 57.3 ns | Dynamic equilibrium 85–200 ns |
| 2Pd₄ + Pure NMP | 39.9 ns | Dynamic equilibrium 65–200 ns |

**Hierarchy:** Water (5–7 ns) << Interface (17–22 ns) << NMP (40–57 ns)

The NMP result is physically the richest — after initial aggregation,
the clusters repeatedly separate and re-contact on a 10–20 ns timescale.
This dynamic equilibrium is only observable beyond 80 ns and demonstrates
that NMP provides **kinetic but not thermodynamic** stabilization at the
Pd₃/Pd₄ scale.

---

## 9. Common Pitfalls

**Pitfall 1 — Using `np.linalg.norm(com2 - com1)` directly:**
Gives the wrong distance when clusters are near box boundaries. Always
apply `minimize_vectors` before computing the norm.

**Pitfall 2 — Selecting all atoms of both clusters together:**
`u.select_atoms('resname pd3')` returns all 6 Pd atoms. You must split
by residue ID to get each cluster's COM independently.

**Pitfall 3 — Not converting units:**
MDAnalysis positions are in **Angstrom**. Divide by 10.0 before plotting
in nm. Time from `ts.time` is in **picoseconds** — divide by 1000.0 for ns.

**Pitfall 4 — `minimize_vectors` argument order:**
The function signature is `minimize_vectors(vectors, box)` where `vectors`
is a displacement (not a position). Pass `com2 - com1`, not the positions
themselves.

---

*This pipeline was used to analyse 600,000 frames of dual-cluster trajectory
data (6 systems × 100,000 frames each at 10 ps/frame) for a JCP manuscript
revision on solvent-controlled Pd nanocluster aggregation.*