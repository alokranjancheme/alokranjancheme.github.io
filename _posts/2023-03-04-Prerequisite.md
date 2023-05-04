---
title: Prerequisite
---

# Prerequisite
First thing first, all the instructions you will read here are specifically for Linux based systems (Ubuntu 22.04) 

## Operating system:
- It is recommended to use the latest version of Ubuntu from the official [website](https://ubuntu.com/download/desktop).
- LAMMPS can also be installed on Windows but, to get full control of its features upcoming instruction is about installing LAMMPS from **source code**

## Install Ubuntu.

- Go to the official Ubuntu [website](https://ubuntu.com/download) and download the latest version of Ubuntu that matches your system's architecture (32-bit or 64-bit).
- Create a bootable USB drive or DVD with the Ubuntu image you downloaded in the previous step. You can use a tool like [Rufus](https://rufus.ie/) to create a bootable USB drive or use the built-in tools on your operating system.
- Insert the bootable USB drive or DVD into your computer and restart the system.
- When the system is booting, you should see a screen with the option to boot from the USB drive or DVD. Choose the option that matches your installation media and press Enter.
- The Ubuntu installer will launch. Select your language and click "Install Ubuntu."
- On the next screen, select your keyboard layout and click "Continue."
- On the "Updates and Other Software" screen, you can choose whether to install updates and third-party software during the installation process. It is recommended to select both options. Click "Continue."
- On the "Installation Type" screen, you can choose to install Ubuntu alongside another operating system, erase the entire disk and install Ubuntu, or do a custom installation. Choose the option that suits your needs and click "Install Now."
- Follow the prompts to select your time zone, enter your username and password, and complete the installation.
- Once the installation is complete, remove the installation media and restart your computer.

## Installing LAMMPS from Source Code.

- Check out the [video](https://www.youtube.com/watch?v=Id3eVPDinDE&t=267s) for detailed instructions
- Open a terminal window in Ubuntu by pressing Ctrl+Alt+T.
- Install the required packages for building LAMMPS by running the following command:
  ```bash
  sudo apt-get update
  ```
  ```bash
  sudo apt-get install build-essential git cmake libfftw3-dev libjpeg-dev libpng-dev libtbb-dev libopenmpi-dev
  ```
  ```bash
  sudo apt-get install cmake
  ```
  ```bash
  sudo apt-get install git
  ```
- Clone the LAMMPS repository from GitHub using the following command:
  ```bash
  git clone -b stable https://github.com/lammps/lammps.git/lammps
  ```
- Change to the lammps directory using the following command:
  ```bash
  cd lammps
  ```
- Create a build directory and change to it using the following commands:
 ```bash
 mkdir build
 ```
 ```bash
 cd build
 ```
- Configure the LAMMPS build process using ccmake by running the following command:
  - cmake -D CMAKE\_INSTALL\_PREFIX=/home/user/lammps/lammps-install ../cmake/
- Now install ccmake gui by running the command:
  ```bash
  Sudo apt install cmake-curses-gui
  ```
  ```bash
  Ccmake ../cmake/
  ```
- Now whatever the feature you like to turn on just click enter then press c to configure then press g to generate
  - Example: BUILD\_MPI ON, BUILD\_OMP ON
- Compile LAMMPS using the following command:
  ```bash
  sudo make -jX
  ```
- Replace X with the number of CPU cores you want to use for the build process. For example, if you have a quad-core CPU, you can use make -j4.
- Install LAMMPS using the following command:
  ```bash
  sudo make install
  ```
  ```bash
  pwd
  ```
- Copy the directory in which you have installed lammps
  - Example /home/user/lammps/lammps-install 
- Congratulations! You have successfully installed LAMMPS from source code

Note: if you see any error during any steps above try to fix that error first by looking on the internet

- Now to access lammps from any where in your system do the following
  ```bash
  Sudo nano .bashrc
  ```
- At the end of the document add the following lines use opuput of pwd command (1)
  ```bash
  export path =$path: /home/user/lammps/lammps-install
  ```
  ```bash
  export OMP\_NUM\_THREADS= 1
  ```
- So every time you open terminal above path will be added.
- To Run LAMMPS type in terminal

```bash
lmp 
``` 
```bash
mpirun np -4 lmp in in.input 
```
-  you should see something like this
LAMMPS (23 Jun 2022 - Update 2)

using 1 OpenMP thread(s) per MPI task

## Installing VMD

- Download the VMD from official website [Link to download](https://www.ks.uiuc.edu/Development/Download/download.cgi?UserID=&AccessCode=&ArchiveID=1475)
- Go to the directory where you have downloaded the tar file and extract. Let's say we have downloaded this file in the Downloads directory.
  ```bash
  cd Downloads/
  ```
  ```bash
  tar xvzf vmd-1.9.3.bin.LINUXAMD64-CUDA102-OptiX650-OSPRay185.opengl.tar.gz
  ```
- It will create a new directory, namely, vmd-1.9.3.
- Move inside this new directory and install as shown below.
  ```bash
  cd vmd-1.9.3/
  ```
- Now, open the file configure and change the $install\_bin\_dir and $install\_library\_dir if you wish otherwise leave the default values as it is. Now paste the following command.
  ```bash
  ./configure
  ```
- Now, move inside the src directory to install.
  ```bash 
  cd src/
  ```
  ```bash
  sudo make install
  ```
- It will take a moment to finish the installation. Now, you can type vmd in the terminal and it should work. However, sometimes, it does not work that way. You have to add it to the path. For that, follow the steps given below.
  ```bash
  sudo nano ~/.bashrc
  ```
- Go to the end of the file and paste the following:
  - $ /path/to/vmd
- For example, $ /usr/local/bin/vmd
- Save and exit.
  ```bash
  source ~/.bashrc
  ```
- If it still doesn't open then add the following path instead of the above.
  - $ /usr/local/bin/vmd/vmd\_LINUXAMD64
  ```bash
  source ~/.bashrc
  ```
- Now, it should work properly
