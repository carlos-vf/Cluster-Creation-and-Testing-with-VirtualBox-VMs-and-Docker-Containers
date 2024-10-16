# Cluster-in-VirtualVox

## Goals
In this tutorial, we are going to create a set of four Linux virtual machines.

- Each machine is capable of connecting to the internet and able to connect with each other privately as well as can be reached from the host machine.
- Our machines will be named master, node01, node02,..., node0X.
- The first machine will act as a master node and will have 1vCPUs, 2GB of RAM and 25 GB hard disk.
- The other machines will act as workers and they will have 1vCPUs, 1GB of RAM and 10 GB hard disk.
- We will assign our machines static IP address in the internal network: 192.168.0.1, 192.168.0.22, 192.168.0.23, .... 192.168.0.XX.

<p align="center">
  <img src="https://github.com/user-attachments/assets/9bddf35b-147c-49d5-9778-f1a68295bcca" width="500">
</p>

## Prerequisite Setup
- VirtualBox must be installed on your system (Windows/Linux/Mac).
- Download the Ubuntu 24.10 image from Ubuntu's website (https://ubuntu.com/download/server).


## Template VM Creation
**Objective**: Create a template VM that can be cloned to deploy the cluster.

### Creation
- In VirtualBox click _**Machine>New**_.
- Configure name and operating system of the machine (you must select the Ubuntu image previously downloaded).
<p align="center">
  <img src="https://github.com/user-attachments/assets/406e5466-695b-4b8a-a7ef-5107f98867e9" width="700">
</p>

- Configure username and password.
<p align="center">
  <img src="https://github.com/user-attachments/assets/4578060f-7f05-44e9-b80a-3d4831a121e3" width="700">
</p>

- Configure hardware (lets select 1000 MB of RAM memory and 1 CPU).
<p align="center">
  <img src="https://github.com/user-attachments/assets/d87a6cf4-a861-45d0-8dbb-b4b1e357c675"  width="700">

- Configure hard disk (select 25 GB if it is not already the default option in your machine).
<p align="center">
  <img src="https://github.com/user-attachments/assets/6345406a-ef18-4c57-acda-7dcba1c6d5b5"  width="700">
</p>

After clicking _Finish_, the machine will start. Lets wait ultil it is fully built so we can test it.


### Package installation
Run these commands to update apt package manager:
```
sudo apt update
sudo apt upgrade
```

Then, install some aditional packages running
```
sudo apt install net-tools gcc make
```

After setting up the template VM, shut it down:
```
sudo shutdown -h now
```

The template machine should be ready now for cloning.


## Mater/Nodes creation & Network adapters configuration
Once it is created, lets configure the two network adapters: one for internet access (NAT) and one for internal communication between VMs.



