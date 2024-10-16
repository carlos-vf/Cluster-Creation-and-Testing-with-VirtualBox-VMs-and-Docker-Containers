# Cluster-in-VirtualBox

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
### Machine clonation
First, lets clone the template machine so we can work in the real nodes of our cluster.
- Right click on _template_ machine in VirtualBox and then select _Clone_.
- Set a name for you clone.
<p align="center">
  <img src="https://github.com/user-attachments/assets/19d28453-aaf8-45d6-95bb-b722caa1bb33"  width="700">
</p>

- Repeat any number of times to create the worker nodes (for now, I will only create one worker named _node01_).


### Network adapters configuration
Once it is created, lets configure the two network adapters:
1. Adapter 1 (NAT): This connects the VM to the host’s network and allows internet access.
2. Adapter 2 (Internal network): This is used for communication between the VMs on a private internal network, where each VM is assigned a static IP address.
   
- Right click in the virtual machine (_master_/_node01_) and then **_Settings>Network_**. Check that the first adapter is attached to NAT.
<p align="center">
  <img src="https://github.com/user-attachments/assets/d7b8041d-f912-4587-87c1-5e6b8789d9b3"  width="700">
</p>

- Select _Adapter 2_, click on _Enable Network Adapter_ and attach it to _Internal Network_. Then select a name for the network that must be the same for all the nodes (e.g., _clustervimnet_).
<p align="center">
  <img src="https://github.com/user-attachments/assets/6fff35b0-a7af-4db3-b422-3b38bacff0b6"  width="700">
</p>

- Save the settings and repeat the configuration for the other nodes.



## Network Configuration for Master Node
The master node will act as the control point for the cluster, managing DNS (domain name system) and DHCP (dynamic host configuration protocol) services.

### Configure the network file
You need to configure the second network adapter (Adapter 2) to assign a static IP address. This IP address will allow the master node to communicate with the other worker nodes on the internal network.

After starting and logging into the master VM, find the network interfaces available on your VM using
```
ip link show
```

The output will be something similar to this:
<p align="center">
  <img src="https://github.com/user-attachments/assets/79386b9a-951c-4d61-99af-00e8fa3a56c0"  width="700">
</p>

where

- **enp0s3** is the first adapter (NAT).
- **enp0s8** is the second adapter (internal network).

We want to configure enp0s8 to have a static IP, like 192.168.0.1. To do this, lets edit the Netplan configuration file.
- Open the file with:
```
sudo vim /etc/netplan/50-cloud-init.yaml
```
- Add the necessary lines to make it look like this:
```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
     dhcp4: no
     addresses: [192.168.0.1/24]
  version: 2
```

Finally, apply the changes with
```
sudo netplan apply
```


### Configure hosts
To make it easier to identify machines in the cluster, lets change the hostname of the master node. Open the host file with:
```
sudo vim /etc/hostname
```
and change its content to _master_.

We also need to define the static IP addresses and hostnames for all nodes in the cluster in the /etc/hosts file. This will allow the master node to resolve the IP addresses of the worker nodes by name. Open the file with:
```
sudo vim /etc/hosts
```
and edit it until looks like this:
```shell
127.0.0.1 localhost
192.168.0.1 master

192.168.0.22 node01

# The following lines are deriable for IPv6 capable hosts
...
```
If you have more than one worker node, be sure to assign them an IP (192.168.0.23 node02, ...).



##  Setting Up Port Forwarding for SSH


















