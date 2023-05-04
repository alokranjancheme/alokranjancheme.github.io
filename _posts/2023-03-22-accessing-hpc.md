---
author:
title: Accessing HPC
categories: [softwere,guidlines]
tags: [tutorials]
---
# Using Supercomputer 
In this section you will learn how to submit or run your calculations on HPC (Supercomputer) using [WinSCP](https://winscp.net/eng/index.php)
## Install WinSCP
Go to official [Website](https://winscp.net/eng/index.php) of WinSCP and install on your computer
No extra instruction is required for the installation just leave everything to default.
## Logging in to your HPC account
Click on the new session in the pop up window fill your user name and password as shown.

![image](https://user-images.githubusercontent.com/125783050/226891525-99a6f74c-4daa-4119-a300-e47c54922c2c.png)

Host name for iit kharagpur is paramshakti.iitkgp.ac.in leave port number as default value (unless you have good reason to change it)
You can save your password for quick relogin but WinSCP never recommends that
Once you have filled the required field another pop up will come to enter the Captcha as shownj

![image](https://user-images.githubusercontent.com/125783050/226901210-20b2d20e-d77f-4a91-96ed-43a44265e718.png)

After entering the captcha you will land to your home directory . As per Paramshakti guidlines user can only run their calculation in scratch directory
home directory is only for storing your files of results of you calculation
### Classification of different directory
1. Root : for softwere installation
2. Scratch : for running calculation,capacity is 2 TB,validity 30 daya
3. Home : for storing datas,capacity is 40 GB,validity infinite
Actually Root directory conntains both Home and Scratch directory both as shown 

![image](https://user-images.githubusercontent.com/125783050/226903290-6c403933-e91f-49dc-a027-82f8bbdfde33.png)

## Submitting task
To submit any job first you have to be in /scratch directory if not run following command in terminal (Press ctrl+shit+T)

```shell
cd ../../scratch/rollnumber/
```
Now we are in the scratch directory, at this point let us assume we want to submit a task on HPC
Paramshakti uses slurmm task manager to manage the job submitted by the user so for more info about the command please refer to thr official website of the [slurm](https://slurm.schedmd.com/pdfs/summary.pdf)
here is the example to submit a job using slurm manager the code given here is shell script to ask module and cpus

```shell
#!/bin/bash

#SBATCH -J lammps_job         # name of the job

#SBATCH -p standard-low       # name of the partition: available options "standard, standard-low, gpu, gpu-low, hm"

#SBATCH -n 150                # no of processes or tasks

#SBATCH --cpus-per-task=1     # no of threads per process or task

#SBATCH -t 72:00:00           # walltime in HH:MM:SS, max value 72:00:00

#list of modules you want to use, for example

module load apps/lammps/14Dec2021/impi2020v4/cpu

#name of the executable

exe="lmp"

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

#run the application

mpirun -bootstrap slurm -n $SLURM_NTASKS $exe -in in.input  # specify the application command-line options, if any, after $exe
```
save the following file as .sh extersion this is important for example job.sh

again open the terminal (ctrl+alt+T) and to put you job in HPC queue you have to type following command in the terminal

```shell
sbatch job.sh
```
now to see the queue of you task type the following

```shell
squeue
```
this command will show the job id and the time of your task
