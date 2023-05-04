---
title: Installing GPU Driver
Author: Abhishek Patel
categories: [Drivers,GPU]
tags: [linux]
---

# GPU driver and CUDA Toolkit

1. Install GPU driver using GUI 
    
    GPU driver can be installed via other methods too but this is the easiest. 
    
    - Go to “Software & Updates”, go to “Additional Drivers” tab.
    - Select the latest driver. In this case, nvidia-driver-470(proprietary, tested)
    - “Apply changes” and Reboot.
    
    ![Screenshot_from_2023-03-21_17-56-10](https://user-images.githubusercontent.com/125783050/229369688-9978fb7b-4f28-4ca6-96d6-afde5538b75d.png)
    

- Open the terminal application and type `nvidia-smi` to see GPU info:

![Screenshot_from_2023-03-22_10-04-11](https://user-images.githubusercontent.com/125783050/229369707-2af48d04-811a-433c-9fbf-edaf6f7bfc67.png)

Here, as you can see the recommended CUDA version is 11.4. Download it from [https://developer.nvidia.com/cuda-toolkit-archive](https://developer.nvidia.com/cuda-toolkit-archive). 

- For installation in Ubuntu: select Linux > x86_64 > Ubuntu

Then you’ll see that it’s only available for Ubuntu 18.04 and 20.04. If the you’ve installed some other version of ubuntu, reinstall one of this, and repeat the whole process. 

![Untitled](https://user-images.githubusercontent.com/125783050/229369714-90ec056e-e0f1-48f3-97f3-c0c9b54108ae.png)

Installation instructions are given there. Just run those commands. 

```bash
wget https://developer.download.nvidia.com/compute/cuda/11.4.0/local_installers/cuda_11.4.0_470.42.01_linux.run
sudo sh cuda_11.4.0_470.42.01_linux.run
```

- During the installation of the CUDA toolkit, it will tell that you have already installed a driver, bla bla bla. Select “**continue**”.
- **Untick** the **driver**. and continue with the installation.
- Open the terminal application and type `nvcc -V`
- It will show some error and recommend a command `sudo apt install cuda-toolkit` or something like that. Run that command.
- retype `nvcc -V`. And you’ll see that CUDA toolkit is now installed.
    
    ![Screenshot_from_2023-03-22_10-03-59](https://user-images.githubusercontent.com/125783050/229369725-2bc72a88-6024-4ea8-80fe-f769f50751b6.png)
    
And voila, it’s done. (If not, just reboot)

# LAMMPS with GPU

Everything is almost the same as regular installation. While building the lammps just change this command.

```bash
cmake -D PKG_GPU=on -D CUDA_ARCH=sm_30 -D CMAKE_CXX_COMPILER=g++ -D CMAKE_C_COMPILER=gcc -D CMAKE_INSTALL_PREFIX=/home/'computer_name'/lammps/lammps-install ../cmake 
```

run the input file:

```cpp
lmp -sf gpu -pk gpu 0 omp 40 -in in.lammps
```

where, 40 = number of processors 

in.lammps = input file

gpu 0 will use all the available gpus

You’ll also need to make some change in the input script. For that, read the lammps manual.
