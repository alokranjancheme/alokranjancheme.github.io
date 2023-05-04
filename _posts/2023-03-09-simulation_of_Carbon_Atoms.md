---
title: Simulation of C atoms in LAMMPS
categories: [project,C_atoms]
tags: [tutorials]
---


# Problem
- In this section we will try to simulate 500 atoms of carbon, in periodic(x,y,z) cubic box of dimentions 60*60*60.Using LAMMPS code

## Disclaimer:
- Please dont rely on the accuracy of data generated here in this sections, these examples are here only for a basic understanding of lammps.

## Avogadro
- first open Avogadro GUI based softwere and then select Caron atom place any were in your work environment.

![image](https://user-images.githubusercontent.com/125783050/224025923-084a98e7-3943-498e-9a4a-17b74c163a54.png)

- Now go to Files > Export > Molecule 

![image](https://user-images.githubusercontent.com/125783050/224026112-c413e035-3315-4914-bf1a-2d97ff629ec5.png)

![image](https://user-images.githubusercontent.com/125783050/224026293-049f0bba-52bd-4317-aece-2282d8fcdc5c.png)

- And select save as type Gaussian Z-matrix input
- Now we have single atom in Z matrix format and it will look like this

![image](https://user-images.githubusercontent.com/125783050/224026974-3579dbb3-7078-49f3-9666-9f79105c8337.png)

- Now save this as .zmat format as required by fftool

![image](https://user-images.githubusercontent.com/125783050/224027285-ecbee789-026b-4a5c-931e-83ccba20083a.png)

- Now we will make c.ff file that contain about mass charge and force field parameters about atoms

![image](https://user-images.githubusercontent.com/125783050/224028038-6e73fd14-a3b8-4e6f-85bb-aaf27fce23ae.png)

- Now we will use these two file as input for fftool and we will create lammps data file
- type the following code

```bash
python3 fftool 500 c.zmat -b 60
```

## FFtool for lammps data file

- At this point you have lammps data file (Please do proper research to find LJ parameters for C atom)
- Now we will lammps script for the simulation

```cpp
clear
units 		real
boundary 	p p p

atom_style 	full

pair_style 	lj/cut 12.0

read_data 	data.lmp
#read_restart	restart.nve.2000100

pair_coeff    	1    1  0.065583     3.400000  # C C

minimize 	1.0e-4 1.0e-6 100 1000

neighbor 	2.0 bin
neigh_modify 	delay 0 every 1 check yes

timestep 	1.0

thermo_style	custom step time etotal ke pe evdwl temp press vol density 
thermo		10000

dump 		dcd all dcd 1000 C_500.dcd
dump_modify	dcd unwrap yes 

velocity        all create 298.15 90066 dist gaussian

fix 		mynve all nve
run		2000000
unfix		mynve
write_restart   restart.nve.*

fix             mynvt all nvt temp 298.15 298.15 100.0 
run             2000000
unfix           mynvt
write_restart   restart.npt.*

fix             mynpt all npt temp 298.15 298.15 100.0 iso 1.0 1.0 1000
run             10000000
unfix           mynpt
write_restart   restart.npt.*
```

- Above given script will run the simulation for total of 14 nanosecond
- 2 ns in NVE, 2ns in NVT and 10ns in NPT

## Visualisation in VMD softwere

- After the simulation is over you will get .dcd file (snapshot of coordinates)  
- Open VMD and go to Externsion > TkConsole and type the following

```bash
topo readlammpsdata data.lmp
```

```bash
pbc box
```

- After that select the molecule and go to File > Load Data into Molecule after all frames are loaded then again in console type
```bash
pbc wrap -all
```

