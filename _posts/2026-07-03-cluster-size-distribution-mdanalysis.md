---
title: "Cluster-Size Distribution Analysis for Metal Nanoclusters Using MDAnalysis and SciPy"
date: 2026-07-03 18:00:00 +0530
categories: [Computational Chemistry, Analysis]
tags: [mdanalysis, scipy, cluster, aggregation, python, palladium, connected-components]
description: >
  A complete Python pipeline for tracking how metal nanocluster fragments
  evolve and coalesce over an MD trajectory — from distance matrix to
  connected components to fractional frequency histograms — using MDAnalysis
  and scipy.sparse.csgraph.
---

## Overview

Tracking cluster-size distributions over an MD trajectory answers a
fundamental question in nanocluster chemistry: *how does aggregation proceed
over time?* The measurement requires identifying, at each frame, which metal
atoms are in direct contact — and grouping them into connected clusters.

This post documents the complete Python pipeline used to analyse 90
independent 100 ns trajectories of dispersed Pd₅₅ nanoclusters in five
solvent environments, producing the cluster-size distribution plots required
for a JCP manuscript revision.

---

## 1. The Theory: Graph Connectivity

At each trajectory frame, the problem reduces to:

1. Build a **distance matrix** between all N Pd atoms: shape (N, N)
2. Convert to an **adjacency matrix**: `adj[i,j] = 1` if `dist[i,j] < cutoff`
3. Find **connected components** of the adjacency graph — each component is one cluster
4. Record the **size** (number of atoms) of each cluster

The cutoff distance (4.0 Å for Pd) is taken from the **first minimum of the
Pd–Pd radial distribution function** — the natural boundary between the
first coordination shell and the surrounding medium.

This approach is physically grounded: two Pd atoms are "in the same cluster"
if and only if they are within first-shell bonding distance. Cluster identity
follows transitively — if A contacts B and B contacts C, all three are in
the same cluster even if A and C are not in direct contact.

---

## 2. Dependencies

```bash
pip install MDAnalysis scipy numpy matplotlib --break-system-packages
```

| Package | Purpose |
|---------|---------|
| `MDAnalysis` | Reading GROMACS `.gro` + `.xtc` trajectories |
| `scipy.sparse.csgraph` | Efficient connected-components on sparse graphs |
| `scipy.sparse` | CSR matrix format for large adjacency matrices |
| `MDAnalysis.lib.distances` | PBC-aware distance array calculation |

---

## 3. Core Functions

### 3.1 PBC-Aware Distance Matrix

```python
from MDAnalysis.lib.distances import distance_array
import numpy as np

def get_pd_distances(positions, box):
    """
    Compute all pairwise Pd-Pd distances with minimum image convention.

    Parameters
    ----------
    positions : np.ndarray, shape (N, 3)
        Pd atom positions in Angstrom
    box : np.ndarray, shape (6,)
        GROMACS box dimensions [a, b, c, alpha, beta, gamma]

    Returns
    -------
    dists : np.ndarray, shape (N, N)
        Symmetric distance matrix with PBC correction
    """
    return distance_array(positions, positions, box=box)
```

The `distance_array` function from MDAnalysis applies the **minimum image
convention** automatically — it finds the nearest periodic image of each
atom, which is essential for clusters that may be split across box boundaries.

### 3.2 Cluster Identification

```python
from scipy.sparse import csr_matrix
from scipy.sparse.csgraph import connected_components

def get_clusters(positions, box, cutoff=4.0):
    """
    Identify clusters at one trajectory frame.

    Parameters
    ----------
    positions : np.ndarray, shape (N, 3)
    box       : np.ndarray, shape (6,)
    cutoff    : float, Angstrom (default 4.0 for Pd-Pd first shell)

    Returns
    -------
    sizes : list of int
        Cluster sizes sorted largest first
    """
    dists = distance_array(positions, positions, box=box)

    # Adjacency matrix: 1 if within cutoff, 0 otherwise
    adj = (dists < cutoff).astype(int)
    np.fill_diagonal(adj, 0)   # exclude self-contacts

    # Sparse representation for efficiency at large N
    graph = csr_matrix(adj)

    # Find connected components
    n_components, labels = connected_components(graph, directed=False)

    # Count atoms in each component
    sizes = sorted(
        [int(np.sum(labels == i)) for i in range(n_components)],
        reverse=True   # largest cluster first
    )
    return sizes
```

### 3.3 Mean Pd-Pd Coordination Number

```python
def get_pdpd_cn(positions, box, cutoff=4.0):
    """
    Mean Pd-Pd coordination number — average number of Pd neighbors
    within cutoff for each Pd atom.

    Equivalent to the first-shell CN from the Pd-Pd RDF.
    """
    dists = distance_array(positions, positions, box=box)
    np.fill_diagonal(dists, np.inf)   # exclude self
    cn_per_atom = np.sum(dists < cutoff, axis=1)
    return float(np.mean(cn_per_atom))
```

---

## 4. Full Trajectory Analysis

```python
import MDAnalysis as mda
import numpy as np

CUTOFF       = 4.0   # Angstrom — Pd-Pd first shell cutoff
PS_PER_FRAME = 10.0  # ps per trajectory frame (nstxout-compressed=5000, dt=0.002)
SKIP         = 500   # analyse every SKIP frames (~5 ns per sample)

def analyze_trajectory(gro_file, xtc_file, pd_resname='PDA'):
    """
    Analyse cluster evolution over an entire trajectory.

    Returns
    -------
    times     : np.ndarray  — time points in ns
    cn_values : np.ndarray  — mean Pd-Pd CN at each time point
    all_sizes : list of list — cluster size lists at each time point
    """
    u  = mda.Universe(gro_file, xtc_file)
    pd = u.select_atoms(f'resname {pd_resname}')

    times, cn_values, all_sizes = [], [], []

    for ts in u.trajectory[::SKIP]:
        t_ns   = ts.time / 1000.0           # ps → ns
        box    = ts.dimensions
        pos    = pd.positions

        cn     = get_pdpd_cn(pos, box, CUTOFF)
        sizes  = get_clusters(pos, box, CUTOFF)

        times.append(round(t_ns, 1))
        cn_values.append(cn)
        all_sizes.append(sizes)

    return np.array(times), np.array(cn_values), all_sizes
```

---

## 5. Stacked Bar Chart

The most informative visualization for cluster-size evolution is a
**stacked bar chart** where each bar represents one time point and each
segment represents one cluster, ranked largest-to-smallest.

```python
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

CMAP   = plt.cm.Blues
N_MAX  = 18
COLORS = [CMAP(0.95 - 0.70 * i / (N_MAX - 1)) for i in range(N_MAX)]
N_PD   = 55

def plot_cluster_evolution(times, all_sizes, out_file):
    fig, ax = plt.subplots(figsize=(6, 6))
    fig.subplots_adjust(left=0.15, right=0.85,
                        top=0.95, bottom=0.14)

    spacing = np.min(np.diff(times)) if len(times) > 1 else 5.0
    bar_w   = spacing * 0.78

    for t, sizes in zip(times, all_sizes):
        bottom = 0
        for ri, sz in enumerate(sizes):
            color = COLORS[min(ri, len(COLORS) - 1)]
            ax.bar(t, sz, width=bar_w, bottom=bottom,
                   color=color, edgecolor='white', linewidth=0.6,
                   zorder=3)
            # Label inside bar if tall enough
            if sz >= 3:
                txt_color = 'white' if ri < 4 else '#1A1A1A'
                ax.text(t, bottom + sz / 2.0, str(sz),
                        ha='center', va='center',
                        fontsize=7, fontweight='bold',
                        color=txt_color, zorder=5)
            bottom += sz

    # Right y-axis: number of clusters
    n_clusters = [len(s) for s in all_sizes]
    ax2 = ax.twinx()
    ax2.plot(times, n_clusters, color='#E74C3C',
             lw=2.0, ls='--', marker='o', ms=4.0,
             alpha=0.9, zorder=6)
    ax2.set_ylabel("Number of clusters", fontsize=11, color='#E74C3C')
    ax2.tick_params(labelsize=10, colors='#E74C3C')
    ax2.set_ylim(0, max(n_clusters) * 1.4 + 1)
    ax2.spines['right'].set_color('#E74C3C')
    ax2.yaxis.label.set_color('#E74C3C')

    ax.set_ylim(0, N_PD + 2)
    ax.set_xlim(times[0] - bar_w * 1.5, times[-1] + bar_w * 1.5)
    ax.set_ylabel("Number of Pd atoms", fontsize=12)
    ax.set_xlabel("Time (ns)", fontsize=12)
    ax.tick_params(labelsize=11)
    ax.grid(True, alpha=0.15, lw=0.5, axis='y', zorder=0)
    ax.spines['top'].set_visible(False)

    plt.savefig(out_file, dpi=150, bbox_inches='tight')
    plt.close()
    print(f"Saved: {out_file}")
```

---

## 6. Fractional Frequency Histogram

For the final state at t=100 ns, a **fractional frequency histogram** shows
what fraction of all observed clusters fall in each size bin:

```python
BINS       = [0, 10, 20, 30, 40, 50, 55]
BIN_LABELS = ['1–10', '11–20', '21–30', '31–40', '41–50', '51–55']

def cluster_histogram(sizes_at_last_frame, out_file):
    """
    Fractional frequency histogram of cluster sizes at last trajectory frame.
    """
    n_bins  = len(BINS) - 1
    x       = np.arange(n_bins)
    counts  = np.zeros(n_bins)
    sizes   = np.array(sizes_at_last_frame)

    for i in range(n_bins):
        lo = BINS[i] + (1 if i > 0 else 0)
        hi = BINS[i + 1]
        counts[i] = np.sum((sizes >= lo) & (sizes <= hi))

    freq = counts / len(sizes) if len(sizes) > 0 else counts

    fig, ax = plt.subplots(figsize=(6, 6))
    fig.subplots_adjust(left=0.18, right=0.95,
                        top=0.95, bottom=0.18)

    ax.bar(x, freq, color='#2980B9', edgecolor='white',
           linewidth=0.8, width=0.75, zorder=3)

    for xi, f in zip(x, freq):
        if f > 0.005:
            ax.text(xi, f + 0.015, f"{f:.2f}",
                    ha='center', va='bottom',
                    fontsize=13, fontweight='bold', color='#1A1A1A')

    ax.set_xticks(x)
    ax.set_xticklabels(BIN_LABELS, fontsize=13, rotation=45, ha='right')
    ax.set_xlim(-0.5, n_bins - 0.5)
    ax.set_xlabel("Cluster size (number of Pd atoms)", fontsize=14)
    ax.set_ylabel("Fractional frequency", fontsize=14)
    ax.set_ylim(0, 1.25)
    ax.tick_params(labelsize=13)
    ax.grid(True, alpha=0.15, axis='y', zorder=0)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)

    plt.savefig(out_file, dpi=150, bbox_inches='tight')
    plt.close()
    print(f"Saved: {out_file}")
```

---

## 7. Choosing the Cutoff Distance

The cutoff must correspond to the **first minimum of the Pd–Pd RDF** —
the distance at which the first coordination shell ends and the second
begins. Choosing too small a cutoff breaks real clusters into fragments;
too large merges distinct clusters.

```bash
# Compute Pd-Pd RDF in GROMACS
gmx rdf -f production.xtc -s production.tpr \
    -ref "resname PDA" -sel "resname PDA" \
    -o rdf_PdPd.xvg -bin 0.01 -rmax 1.0 \
    -b 50000 -e 100000

# Find first minimum in Python
import numpy as np
data = np.loadtxt('rdf_PdPd.xvg', comments=['#','@'])
r, g = data[:,0]*10, data[:,1]   # nm → Ang

# First minimum after first peak
first_peak = np.argmax(g[:50])
first_min  = first_peak + np.argmin(g[first_peak:first_peak+30])
print(f"First minimum at r = {r[first_min]:.2f} Ang → use as cutoff")
# Typical output: First minimum at r = 3.85 Ang → use cutoff = 4.0 Ang
```

For Pd clusters in water and NMP with the Heinz 2008 INTERFACE FF,
the first RDF minimum falls consistently at **3.8–4.2 Å**. A cutoff of
**4.0 Å** is used throughout.

---

## 8. Scientific Results: Aggregation Hierarchy

Applying this pipeline to all five Pd₅₅ dispersed systems at t=100 ns:

| System | Final clusters | Histogram interpretation |
|--------|----------------|--------------------------|
| Vacuum | [55] | 100% in 51–55 bin — complete aggregation |
| Pure Water | [55] | 100% in 51–55 bin — complete aggregation |
| 1:1 v/v | [55] | 100% in 51–55 bin — NMP insufficient at this ratio |
| 6:1 v/v | [55] | 100% in 51–55 bin — water-rich, fully aggregated |
| Pure NMP | [28, 14, 13] | 67% in 11–20 bin, 33% in 21–30 bin — metastable |

The pure NMP result is the key finding: **three distinct sub-clusters persist
to 100 ns**, confirming NMP provides kinetic stabilization of the dispersed
state via preferential carbonyl-oxygen coordination of Pd surface sites.

---

*This analysis pipeline was developed for the JCP major revision of a study
on solvent-dependent Pd₅₅ nanocluster aggregation. The connected-components
approach is general and applies to any metal nanoparticle system in GROMACS
format where cluster identity is defined by a distance cutoff.*