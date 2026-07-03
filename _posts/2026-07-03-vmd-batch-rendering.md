---
title: "VMD Batch Snapshot Rendering for Publication Figures: The capture_frames.tcl Workflow"
date: 2026-07-03 16:00:00 +0530
categories: [Computational Chemistry, Visualization]
tags: [vmd, tcl, rendering, publication, molecular-dynamics, snapshots, batch]
description: >
  A complete guide to automated VMD snapshot rendering for publication-quality
  figures — extracting exact view matrices from a live VMD session, drawing
  PBC boxes as graphics objects, and batch-processing 30+ trajectory systems
  without manual intervention.
---

## Overview

Generating publication-quality snapshots from MD trajectories in VMD typically
involves manually adjusting the view, setting representations, and clicking
"Render" for each frame. For a project with 30 systems × 6 time points = 180
images, this is impractical.

This post documents a complete automated workflow — a single TCL script that
reproduces your exact manually-adjusted view across any number of systems and
saves PNG snapshots at regular time intervals.

---

## 1. The Core Problem: Reproducing Your View

VMD stores the current view as four 4×4 matrices accessible via `molinfo`:

```tcl
set mol [molinfo top]
molinfo $mol get center_matrix
molinfo $mol get rotate_matrix
molinfo $mol get scale_matrix
molinfo $mol get global_matrix
```

Once you have adjusted the view manually to look exactly right, extract these
values from the Tk Console and hardcode them into your script. The matrices
are stable across systems of the same box size — you only need to compute
them once.

**Extract your current view:**

```tcl
# Run in VMD Tk Console after setting your view manually
set mol [molinfo top]
puts "center: [molinfo $mol get center_matrix]"
puts "rotate: [molinfo $mol get rotate_matrix]"
puts "scale:  [molinfo $mol get scale_matrix]"
puts "box:    [molinfo $mol get a] [molinfo $mol get b] [molinfo $mol get c]"
```

Save the output — these are your canonical view matrices for all figures
in the paper.

---

## 2. The `capture_frames.tcl` Script

The complete script. Source it from the Tk Console or pass the system name
via `-args` for unattended batch use:

```tcl
# capture_frames.tcl
# ------------------
# Two usage modes:
#
# 1. Command-line (no prompt):
#    vmd <sys>/em.gro <sys>/production.xtc -dispdev win \
#        -e capture_frames.tcl -args pd55_disp_water_r1
#
# 2. Tk Console (prompts for name):
#    source /path/to/capture_frames.tcl

set t_step_ns    20
set ps_per_frame 10.0
set hpc_root     "/home/cmrl/Downloads/A_Ranjan/gromacs_hpc"

# ── Get system name ───────────────────────────────────────────
if {$argc >= 1} {
    set sys_name [lindex $argv 0]
} else {
    puts -nonewline "Enter system name: "
    flush stdout
    gets stdin sys_name
    set sys_name [string trim $sys_name]
    if {$sys_name == ""} { puts "ERROR: No name."; return }
}

set out_dir "${hpc_root}/snapshots/${sys_name}"
file mkdir $out_dir
set mol [molinfo top]

# ── Display settings ──────────────────────────────────────────
color change rgb 0    0.0 0.0 1.0   ;# ColorID 0 = blue
color Display Background white
display projection   Orthographic
display depthcue     on
display cuestart     0.500000
display cueend       10.000000
display cuedensity   0.320000
display cuemode      Exp2
display shadows      off
display resize       900 900
axes location        off

# ── Clear default representations ─────────────────────────────
set nreps [molinfo $mol get numreps]
for {set i 0} {$i < $nreps} {incr i} { mol delrep 0 $mol }

# ── Box dimensions ────────────────────────────────────────────
animate goto 0
set bx [molinfo $mol get a]
set by [molinfo $mol get b]
set bz [molinfo $mol get c]

# ── Apply view matrices (replace with your extracted values) ──
set cx [expr {-$bx / 2.0}]
set cy [expr {-$by / 2.0}]
set cz [expr {-$bz / 2.0}]

molinfo $mol set center_matrix \
    "{{1 0 0 $cx} {0 1 0 $cy} {0 0 1 $cz} {0 0 0 1}}"
molinfo $mol set rotate_matrix \
    "{{0.962141 0.0524163 0.267466 0} \
      {0.0427886 0.940115 -0.338159 0} \
      {-0.269174 0.336801 0.90228 0} \
      {0 0 0 1}}"
molinfo $mol set scale_matrix \
    "{{0.0320941 0 0 0} \
      {0 0.0320941 0 0} \
      {0 0 0.0320941 0} \
      {0 0 0 1}}"
molinfo $mol set global_matrix \
    "{{1 0 0 0} {0 1 0 0} {0 0 1 0} {0 0 0 1}}"

# ── Draw blue PBC box (12 edges, solid, width 3) ──────────────
graphics $mol color 0
graphics $mol line "0 0 0"       "$bx 0 0"       style solid width 3
graphics $mol line "0 0 0"       "0 $by 0"       style solid width 3
graphics $mol line "0 0 0"       "0 0 $bz"       style solid width 3
graphics $mol line "$bx 0 0"     "$bx $by 0"     style solid width 3
graphics $mol line "$bx 0 0"     "$bx 0 $bz"     style solid width 3
graphics $mol line "0 $by 0"     "$bx $by 0"     style solid width 3
graphics $mol line "0 $by 0"     "0 $by $bz"     style solid width 3
graphics $mol line "0 0 $bz"     "$bx 0 $bz"     style solid width 3
graphics $mol line "0 0 $bz"     "0 $by $bz"     style solid width 3
graphics $mol line "$bx $by $bz" "$bx $by 0"     style solid width 3
graphics $mol line "$bx $by $bz" "$bx 0 $bz"     style solid width 3
graphics $mol line "$bx $by $bz" "0 $by $bz"     style solid width 3

# ── Representations (auto-detect solvent from system name) ────
if {[string match "*water*" $sys_name] || \
    [string match "*1to1*"  $sys_name] || \
    [string match "*6to1*"  $sys_name]} {
    mol representation Points 2.500000
    mol color ColorID 1
    mol selection {resname OPC and name O}
    mol material Opaque
    mol addrep $mol
}
if {[string match "*nmp*"     $sys_name] || \
    [string match "*pureNMP*" $sys_name] || \
    [string match "*1to1*"    $sys_name] || \
    [string match "*6to1*"    $sys_name]} {
    mol representation Points 2.500000
    mol color ColorID 7
    mol selection {resname NMP and name N}
    mol material Opaque
    mol addrep $mol
}
mol representation VDW 1.000000 12.000000
mol color ColorID 10
mol selection {resname PDA ICO}
mol material Opaque
mol addrep $mol

display update
after 500

# ── Capture t=0 from em.gro (frame 0, pre-EM) ────────────────
animate goto 0
display update
after 300
set base "${out_dir}/${sys_name}_0ns"
render snapshot ${base}.tga
catch {exec convert ${base}.tga ${base}.png && file delete -force ${base}.tga}
puts "-> ${sys_name}_0ns.png (pre-EM)"

# ── Capture production frames every t_step_ns ─────────────────
set total_frames [molinfo $mol get numframes]
set prod_total   [expr {$total_frames - 1}]
set t_max_ns     [expr {int($prod_total * $ps_per_frame / 1000.0)}]

for {set t_ns $t_step_ns} {$t_ns <= $t_max_ns} \
        {set t_ns [expr {$t_ns + $t_step_ns}]} {
    set prod_frame [expr {int($t_ns * 1000.0 / $ps_per_frame)}]
    set abs_frame  [expr {$prod_frame + 1}]
    if {$abs_frame >= $total_frames} {
        set abs_frame [expr {$total_frames - 1}]
    }
    animate goto $abs_frame
    display update
    after 200
    set base "${out_dir}/${sys_name}_${t_ns}ns"
    render snapshot ${base}.tga
    catch {exec convert ${base}.tga ${base}.png && \
           file delete -force ${base}.tga}
    puts "-> ${sys_name}_${t_ns}ns.png"
}

puts "Done: $out_dir"
if {$argc >= 1} { quit }
```

---

## 3. Batch Processing All Systems

With the script saved as `capture_frames.tcl`, run all 10 pd55 r1 systems
unattended:

```bash
cd ~/Downloads/A_Ranjan/gromacs_hpc

for sys in \
    pd55_disp_vacuum_r1 pd55_disp_water_r1 pd55_disp_pureNMP_r1 \
    pd55_disp_1to1_r1   pd55_disp_6to1_r1 \
    pd55_ico_vacuum_r1  pd55_ico_water_r1  pd55_ico_pureNMP_r1 \
    pd55_ico_1to1_r1    pd55_ico_6to1_r1; do

    echo "=== $sys ==="
    vmd ${sys}/em.gro ${sys}/production.xtc \
        -dispdev win \
        -e capture_frames.tcl \
        -args $sys
done
```

VMD opens, renders all frames, saves PNGs, and closes automatically before
the next system starts. No manual input required.

---

## 4. Key Lessons and Pitfalls

### Use `graphics` objects, not `pbc box`, for the PBC box

The `pbc box` command draws using the trajectory box dimensions at the
current frame. For systems where the box fluctuates during NPT,
this gives slightly inconsistent box sizes across snapshots.

The `graphics` approach uses a **fixed box** drawn from the equilibrated
dimensions — more reproducible for publication figures where all panels
must show the same box geometry.

### `color change rgb 0` must come before `graphics $mol color 0`

VMD processes TCL sequentially. If you set `graphics $mol color 0` before
calling `color change rgb 0 0.0 0.0 1.0`, the box is drawn in the
**default** color 0 (black), not blue. The color table is mutable at
runtime — set it first, draw second.

### Load `em.gro` as the structure file, not `npt_center.gro`

```bash
# CORRECT — em.gro as frame 0 gives the pre-EM dispersed state
vmd ${sys}/em.gro ${sys}/production.xtc -dispdev win -e capture_frames.tcl

# WRONG — npt_center.gro skips the t=0 dispersed snapshot
vmd ${sys}/npt_center.gro ${sys}/production.xtc ...
```

Frame 0 in VMD corresponds to the structure file. By loading `em.gro`,
frame 0 = the initial Packmol placement (before any physics), which is
scientifically the correct t=0 snapshot for a dispersed cluster.

### NMP atom name is `N`, not `N1`

When selecting NMP atoms for representation:

```tcl
# CORRECT — nitrogen in OPLS-AA NMP topology
mol selection {resname NMP and name N}

# WRONG — no atoms selected, NMP invisible in snapshot
mol selection {resname NMP and name N1}
```

Always verify atom names with `grep "NMP" system/em.gro | head -3` before
setting representations.

### `-dispdev win` is required for `render snapshot`

The `snapshot` renderer requires an active OpenGL display context. Running
with `-dispdev text` causes a segmentation fault at the render call.

```bash
# CORRECT
vmd ... -dispdev win -e capture_frames.tcl

# FAILS — segfault at render snapshot
vmd ... -dispdev text -e capture_frames.tcl
```

---

## 5. Extracting the Exact View from a Live Session

The most reliable workflow for getting the canonical view:

1. Open one representative system in VMD interactively
2. Adjust rotation, scale, representations manually until it looks right
3. In the Tk Console, run `save_state /path/to/view.tcl`
4. Open `view.tcl` and extract the `set viewpoints(...)` entry
5. Copy the four matrices into `capture_frames.tcl`

The `set viewpoints` entry looks like:

```tcl
set viewpoints([molinfo top]) {
    {{1 0 0 -30.005} {0 1 0 -30.0} {0 0 1 -30.0} {0 0 0 1}}    ;# center
    {{0.962 0.052 0.267 0} {0.043 0.940 -0.338 0}
     {-0.269 0.337 0.902 0} {0 0 0 1}}                           ;# rotate
    {{0.032 0 0 0} {0 0.032 0 0} {0 0 0.032 0} {0 0 0 1}}       ;# scale
    {{1 0 0 0} {0 1 0 0} {0 0 1 0} {0 0 0 1}}                   ;# global
}
```

---

## 6. Output Structure

Each system produces a folder of PNGs:

```
snapshots/
├── pd55_disp_water_r1/
│   ├── pd55_disp_water_r1_0ns.png    ← pre-EM (Packmol placement)
│   ├── pd55_disp_water_r1_20ns.png
│   ├── pd55_disp_water_r1_40ns.png
│   ├── pd55_disp_water_r1_60ns.png
│   ├── pd55_disp_water_r1_80ns.png
│   └── pd55_disp_water_r1_100ns.png
├── pd55_ico_water_r1/
│   └── ...
```

Six images per system, assembled into a publication-ready panel using
ImageMagick's `montage`:

```bash
montage snapshots/pd55_disp_water_r1/*.png \
    -geometry 300x300+4+4 \
    -tile 6x1 \
    -background white \
    snapshots/panel_pd55_disp_water.png
```

---

*This workflow was developed to generate 180 publication figures (30 systems
× 6 time points) for a JCP manuscript revision on Pd₅₅ nanocluster
aggregation in water-NMP mixed solvents — all rendered consistently from
a single canonical view matrix extracted once from a live VMD session.*