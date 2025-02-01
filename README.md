# Cluster Creation and Testing in VirtualBox

## Table of Contents
- [Goals](#goals)
- [Prerequisite Setup](#prerequisite-setup)
- [Template VM Creation](#template-vm-creation)
  - [Machine Creation](#machine-creation)
  - [Package Installation](#package-installation)
- [Nodes Creation & Network Adapters Configuration](#nodes-creation--network-adapters-configuration)
  - [Machine Clonation](#machine-clonation)
  - [Network Adapters Configuration](#network-adapters-configuration)
- [Master Node Configuration](#master-node-configuration)
  - [SSH Configuration](#ssh-configuration)
    - [Port Forwarding Rule](#port-forwarding-rule)
    - [SSH Service](#ssh-service)
    - [Passwordless SSH](#passwordless-ssh)
  - [Network Configuration](#network-configuration)
  - [Hostnames Configuration](#hostnames-configuration)
  - [DHCP & DNS](#dhcp--dns)
  - [Port Forwarding and NAT in the Internal Network](#port-forwarding-and-nat-in-the-internal-network)
  - [Distributed File System](#distributed-file-system)
- [Worker Nodes Configuration](#worker-nodes-configuration)
  - [Hostname Configuration](#hostname-configuration)
  - [Network Configuration](nNetwork-configuration)
  - [SSH Configuration](#ssh-configuration)
  - [Testing DHCP, DNS & NAT](#testing-dhcp-dns--nat)
  - [File System Configuration (mounting point)](#file-system-configuration-mounting-point)
  - [File System Configuration (automatic mount)](#file-system-configuration-automatic-mount)
- [More Workers](#more-workers)
- [Performance Tests](#performance-tests)
  - [CPU Tests (HPCC)](#cpu-tests-hpcc)
    - [HPCC Compilation](#hpcc-compilation)
    - [HPCC Parameters and Execution](#hpcc-parameters-and-execution)



## Goals
The goal is to create a cluster of virtual machines connected by a virtual switch and testing their performance.

- Our machines will be named *master* (or *node01*), *node02*, *node03*, ..., *nodeXX*.
- Each machine will have 2 CPUs, 2GB of RAM and 20 GB hard disk.
- The first machine will act as a master node, being the DNS, DHCP and gateway of the cluster.
- The other machines will act as workers nodes.

<p align="center">
  <img src="https://github.com/user-attachments/assets/9bddf35b-147c-49d5-9778-f1a68295bcca" width="500">
</p>

## Prerequisite Setup
- VirtualBox (Windows/Linux/Mac).
- Download the Ubuntu 24.10 image from Ubuntu's website (https://ubuntu.com/download/server).


## Template VM Creation
**Objective**: Create a template VM that can be cloned to deploy the cluster.

### Machine Creation
- In VirtualBox click _**Machine>New**_.
- Configure name and operating system of the machine (you must select the Ubuntu image previously downloaded).
<p align="center">
  <img src="https://github.com/user-attachments/assets/567237c9-17f9-4249-8205-f9194e751087" width="700">
</p>

- Configure username and password.
<p align="center">
  <img src="https://github.com/user-attachments/assets/7d0c1aa1-8ae0-4a26-9f25-bc80fa56971d" width="700">
</p>

- Configure hardware (let's select 2048 MB of RAM memory and 2 CPU).
<p align="center">
  <img src="https://github.com/user-attachments/assets/3a058592-66fe-427f-b9f1-e1f9ae4f1252"  width="700">

- Configure hard disk (select 20 GB).
<p align="center">
  <img src="https://github.com/user-attachments/assets/a5e93cd6-b9ce-4bb7-84a6-9aff836f855d"  width="700">
</p>

After clicking _Finish_, the machine will start. Lets wait ultil it is fully built so we can test it.


### Package Installation
Run these commands to update apt package manager:
```
sudo apt update
sudo apt upgrade
```

Then, install some aditional packages running
```
sudo apt install net-tools gcc make openssh-server
```

After setting up the template VM, shut it down:
```
sudo shutdown -h now
```

The template machine should be ready now for cloning.


## Nodes Creation & Network Adapters Configuration
### Machine Clonation
First, let's clone the template machine so we can work in the real nodes of our cluster.
- Right click on _template_ machine in VirtualBox and then select _Clone_.
- Set a name for you clone.
<p align="center">
  <img src="https://github.com/user-attachments/assets/3306c8e0-11b8-43c0-a5f8-c65eb4f07a71"  width="700">
</p>

- Repeat for both *master* and *node02* (once the worker is configured, we will clone it again).


### Network Adapters Configuration
In order to configure the two network adapters:
- Adapter 1 (NAT): This connects the VM to the host’s network and allows internet access.
- Adapter 2 (Internal network): This is used for communication between the VMs on a private internal network, where each VM is assigned a dynamic IP address.
   
- Right click in the virtual machine *master* and then **_Settings>Network_**. Check that the first adapter is attached to NAT.
<p align="center">
  <img src="https://github.com/user-attachments/assets/44323437-3db4-43a1-b284-a3af2c167c04"  width="700">
</p>

- Select _Adapter 2_, click on _Enable Network Adapter_ and attach it to _Internal Network_. Then select a name for the network that must be the same for all the nodes (e.g., _clusternet_).
<p align="center">
  <img src="https://github.com/user-attachments/assets/a0172a05-eb97-4a3f-878a-215a9056eae4"  width="700">
</p>

- For *node02* we will only activate the second adapter with the internal network. Worker nodes won't be able to connect directly to the internet.
<p align="center">
  <img src="https://github.com/user-attachments/assets/347bf75e-def2-4b5a-8360-d36052b8c050"  width="700">
</p>
<p align="center">
  <img src="https://github.com/user-attachments/assets/ea8eaa2d-18b9-437e-bd0c-b216f1464ce8"  width="700">
</p>



## Master Node Configuration

### SSH Configuration

#### Port Forwarding Rule
To connect to the VMs (e.g., the master node) from your host machine, VirtualBox requires port forwarding rules to map ports from the guest VMs to your host machine. In this case, you’ll set up a rule to forward SSH traffic from port 2222 on the host to port 22 on the master node.

- Right click on the _master_ machine, then **_Settings>Network>Port Forwarding>Add_**.
- Fill the gaps following the table.

| Namer  | Protocol | Host IP | Host Port | Guest IP | Gest Port |
| ------------- | ------------- | ------------- | ------------- | ------------- | ------------- |
| ssh  | TCP  | 127.0.0.1 | 2222 | | 22 |

- Apply the changes and start the VM.


#### SSH Service
Lets check SSH is enabled now by running the command:
```
sudo systemctl status ssh
```
<p align="center">
  <img src="https://github.com/user-attachments/assets/4a3d19e1-217c-434d-9096-d6dda7913f5c"  width="700">
</p>

To ensure that the service starts on boot:
```
sudo systemctl enable ssh
```


#### Passwordless SSH
To avoid entering a password each time you connect via SSH, you can configure SSH key-based authentication.

First, on your host machine, generate an SSH key pair (if you don’t have one). Note: the commands below run on the **Windows PowerShell**.
```
ssh-keygen
```

Now, copy the generated public key (mine is _id_ed25519.pub_ but yours can be different) from your host to the master node (your VM must be running).
```
cd .ssh
scp -P 2222 id_ed25519.pub admin@127.0.0.1:~
```

If your host machine shows a HUGE warning telling you that you can be suffering a man-in-the-middle attack, don't worry, you are that man. When you connect to an SSH server for the first time, the server’s host key is stored in the known_hosts file on your client machine. If the host key of the server changes (which can happen, for example, if you rebuild the VM), SSH assumes this could be a security issue. You need to remove the outdated or incorrect host key from your known_hosts file and run the _scp_ command again.
```
ssh-keygen -R [127.0.0.1]:2222
```

At this point, you should be been able to connect you VM by SSH. Try it by running:
```
ssh -p 2222 admin@127.0.0.1
```

Now that the public key has been copied into your home directory, lets add it to the `authorized_keys` file. Run these commands on you VM:
```
cat id_ed25519.pub >> .ssh/authorized_keys
rm id_ed25519.pub
```

If everything is set up correctly, you will be able to log in without needing to enter a password again.



### Network Configuration
The master node will act as the control point for the cluster, managing DNS (domain name system) and DHCP (dynamic host configuration protocol) services.

You need to configure the second network adapter (Adapter 2) to assign a static IP address (only for this node). This IP address will allow the *master* node to communicate with the other worker nodes on the internal network.

After starting and logging into the *master* VM, find the network interfaces available on your VM using
```
ip link show
```

The output will be something similar to this:
<p align="center">
  <img src="https://github.com/user-attachments/assets/b14bff71-050d-4d04-af1b-9702d85c0a95"  width="700">
</p>

where

- **enp0s3** is the first adapter (NAT).
- **enp0s8** is the second adapter (internal network).

We want to configure enp0s8 to have a static IP, like 192.168.0.1 (usually the gateway takes the first available address of the network). To do this, lets edit the Netplan configuration file.
- Open the file with:
```
sudo vim /etc/netplan/50-cloud-init.yaml
```
- Add the necessary lines to make it look like this:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
     dhcp4: false
     addresses: [192.168.0.1/28]
```

Finally, apply the changes with
```
sudo netplan apply
```


### Hostnames Configuration
To make it easier to identify machines in the cluster, lets change the hostname of the *master* node. Open the host file with:
```
sudo vim /etc/hostname
```
and change its content to _master_.

We also need to define the static IP address and hostname for the master node in `/etc/hosts` file. Open the file with:
```
sudo vim /etc/hosts
```
and edit it until looks like this:
```yaml
127.0.0.1 localhost
192.168.0.1 master

# The following lines are deriable for IPv6 capable hosts
...
```

Reboot you machine to check the new changes.


### DHCP & DNS
The goal of this section is to set up the master node to dynamically assign IP addresses and manage DNS resolution for the worker nodes.

First, install `dnsmasq`, which is a lightweight DHCP and DNS server. It will serve the internal network, assigning dynamic IPs to the nodes and resolving hostnames.
```
sudo apt install dnsmasq -y
```

Then, we need to configure the `/etc/dnsmasq.conf` file. The configuration given in the tutorial is minimal, but it’s sufficient for basic DHCP and DNS operations.
```
sudo vim /etc/dnsmasq.conf
```

Here are the key settings:

- **port=53**: This configures dnsmasq to listen on port 53, the standard port for DNS.
- **interface=enp0s8**: This defines the network interface on which `dnsmasq` listens. Ensure that the internal network interface (in my case, enp0s8) is specified.
- **listen-address**: Specifies the IP addresses `dnsmasq` will listen on. These include loopback (127.0.0.1) and the IP of the master node on the internal network (192.168.0.1).
- **dhcp-range**: This defines the IP range that dnsmasq will assign to the worker nodes, with a lease time of 12 hours.
- **dhcp-option=option:dns-server,192.168.0.1**: Specifies the DNS server for the DHCP clients, which is the master node itself.

Add the following lines of code to the file:
```yaml
port=53
bogus-priv
strict-order
interface=enp0s8
listen-address=::1,127.0.0.1,192.168.0.1
bind-interfaces
log-queries
log-dhcp
dhcp-range=192.168.0.2,192.168.0.14,255.255.255.240,12h
dhcp-option=option:dns-server,192.168.0.1
dhcp-option=3
```

To ensure proper DNS resolution, create a symbolic link to the system `resolv.conf` file, which uses `systemd-resolved`.
```
sudo ln -fs /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

We now need to ensure that the master node has the correct network configuration for both interfaces. Edit the network file to make it similar to this:
```
sudo vim /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses: [192.168.0.1/28]
      nameservers:
        addresses: [192.168.0.1]
```
```
sudo netplan apply
```

If we want to control how `dnsmasq` interacts with `resolvconf`, we need to uncommend a couple of lines in the `/etc/default/dnsmasq` file. Lets open the file:
```
sudo vim /etc/default/dnsmasq
```

And uncomment the following lines:
```yaml
IGNORE_RESOLVCONF=yes
DNSMASQ_EXCEPT="lo"
```

Apply the changes by restarting the services.
```
sudo systemctl restart dnsmasq systemd-resolved
```

After restarting, check that dnsmasq is running without issues. The service should be enabled and active.
```
sudo systemctl status dnsmasq
```
<p align="center">
  <img src="https://github.com/user-attachments/assets/41ff66a5-21e0-4626-b88c-9aa154d853ea"  width="700">
</p>


### Port Forwarding and NAT in the Internal Network
In order to connect to the internet from the worker nodes using the master node as a gateway, we need to configure port forwarding in our master node.

Lets create a new file and write the following line:
```
sudo vim /etc/sysctl.d/99-ipforward.conf
```
```yaml
net.ipv4.ip_forward=1
```

Apply changes inmmediately
```
sudo sysctl --system
```

Verify that IP forwarding is enabled by running the following command. If the ouput is "1", everything is fine by now.
```
cat /proc/sys/net/ipv4/ip_forward
```

We need now to ensure that the IP tables are configured to allow NAT (Network Address Translation) between network interfaces.

`iptables-persistent` is a package that automatically saves your current iptables rules and loads them when the system boots. Lets install it (press _yes_ all the times it is needed):
```
sudo apt install iptables-persistent
```

Now it's time to set up a NAT rule for the outgoing interface (enp0s3) and save it:
```
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo netfilter-persistent save
```



### Distributed File System
To build a cluster, you need a shared filesystem that all nodes can access.

Lets begin installing the NFS kernel server package.
```
sudo apt install nfs-kernel-server
```

Then create a shared directory that all nodes will access.
```
sudo mkdir /shared
sudo chmod 777 /shared
```

Open the NFS exports file and specify the directory and network permissions.
- **rw**: Grants read and write permissions.
- **sync**: Ensures changes are written to disk immediately.
- **no_root_squash**: Allows root users on clients to access files as root.
- **no_subtree_check**: Disables subtree checking for better performance.
```
sudo vim /etc/exports
```
```yaml
/shared/ 192.168.0.0/255.255.255.240(rw,sync,no_root_squash,no_subtree_check)
```
This allows all machines on the internal network (192.168.0.0/28) to access the shared directory with read/write permissions.

Enable and start the NFS server.
```
sudo systemctl enable nfs-kernel-server
sudo systemctl restart nfs-kernel-server
```

We can now create a test file in the new folder
```
touch /shared/hello_world.txt
```


## Worker Nodes Configuration

We can move now to the *node02* setup. 

### Hostname Configuration
Let's change the hostname of the worker node. Open the host file with:
```
sudo vim /etc/hostname
```
and change its content to *node02*.


### Network Configuration
First we need to modify the network file.
```
sudo vim /etc/netplan/50-cloud-init.yaml
```
```yaml
network:
  ethernets:
    enp0s8:
      dhcp4: true
      dhcp-identifier: mac
      nameservers:
        addresses: [192.168.0.1]
      routes:
        - to: 0.0.0.0/0
          via: 192.168.0.1
```

For the adapter enp0s8:

- **dhcp4: true**: The interface will use DHCP to get an IP address. This IP will be in the range defined by the master node's DHCP server (set up using `dnsmasq`).

-**dhcp-identifier: mac**: This ensures that the DHCP server assigns an IP address based on the MAC address, allowing for stable IP assignments.

- **nameservers.addresses: [192.168.0.1]**: The DNS server for the internal network will be the master node at 192.168.0.1. This ensures that the DNS queries for the cluster are handled by the master node.

Apply the configuration:
```
sudo netplan apply
```

Finally, set the DNS server.
```
sudo ln -fs /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

We should now reboot the machine to make sure the changes work properly.


### SSH Configuration

Activate the service in the *worker* node by running:
```
sudo systemctl start ssh
```

It is important to note that we won't be able to connect from our host machine, but from the *master* node. Now we can generate a key from the *master* and copy it in the *worker*. change *admin* for your node's user and the *192.168.0.2* for its IP.
```
ssh-keygen -t rsa -b 4096
ssh-copy-id admin@192.168.0.2
```

Now we should be able to connect.
```
ssh admin@192.168.0.2
```


### Testing DHCP, DNS & NAT
In order to verify that the nodes gets an IP from the DHCP, lets run the command:
```
ip address
```

There you should be able to see the IP address assigned to the interface enp0s8. We are sure now that the DHCP server is working properly.

In order to test the DNS server, you can run from your worker node the command
```
host master
```
If it resolves the name and returns the IP of the node, then the DNS is working as it should, too.

Moreover, you should also be able to connect to the internet from your workers. You can try to connect _google.com_:
```
ping google.com
```
If it works, that means that your master node is able to return the packages from the destination to you (NAT and Port Forwarding are working properly).



### File System Configuration (mounting point)
Lets install the NFS client in *node02*.
```
sudo apt install nfs-common
```

Create a mount point (where the shared folder from the master will be mounted).
```
sudo mkdir /shared
```

Mount the shared directory from the master node to this folder.
```
sudo mount 192.168.0.1:/shared /shared
```

To check if everything went fine, just show the content of the folder. The file `ciao_mondo.txt` should appear.
```
ls /shared
```

NOTE: This type of mounting only works while the machine is running. After rebooting the mount point will disappear.


### File System Configuration (automatic mount)
Install AutoFS to automatically mount the shared directory on boot, install the AutoFS package.
```
sudo apt -y install autofs
```

Edit the `auto.master` configuration file to include the mount points adding the following line:
```
sudo vim /etc/auto.master
```
```
/- /etc/auto.mount
```

Create the AutoFS configuration file (`auto.mount`) to define the NFS mount. Add the following configuration:
```
sudo vim /etc/auto.mount
```
```yaml
/shared -fstype=nfs,rw  192.168.0.1:/shared
```

Finally, restart AutoFS service to apply changes.
```
sudo systemctl restart autofs
```
Now the `shared` folder shoud remain mounted even after rebooting your node.



### More Workers

In order to create more worker nodes, we only need to clone our already configured node. For now, we will only create one more worker node: *node03*. After bootstrapping we should change the hostname to the new node name by running:
```
sudo vim /etc/hostname
```

Once the machine is rebooted, it will be able to get an IP from the DHCP, connect to the internet or other nodes using the DNS and access the shared file system.

Note: you need to re-configure in each node the SSH keys in order to use the service.




## Performance Tests

### CPU Tests (HPCC)

HPCC (High-Performance Computing Challenge) is a benchmark suite designed to evaluate the performance of processors, memory subsystems, and interconnects in high-performance computing (HPC) systems (https://hpcchallenge.org/hpcc/index.html). 

HPCC consists of multiple tests, including:

- HPL (Linpack): Measures floating-point performance by solving dense linear systems.
- DGEMM: Measures dense matrix-matrix multiplication performance.
- PTRANS: Tests parallel matrix transpose performance (network bandwidth).
- RandomAccess: Evaluates memory bandwidth by performing random updates in a large array.
- FFT: Measures floating-point performance for Fast Fourier Transforms.
- STREAM: Assesses sustainable memory bandwidth.

#### MKL & OpenMPI Installation
In order to work with HPCC, it is needed first to install Intel MKL (Math Kernel Library) and OpenMPI in our system. This installation must be performed in ALL NODES.

To download Intel’s GPG key:
```
wget -qO - https://repositories.intel.com/graphics/intel-graphics.key | sudo tee /etc/apt/trusted.gpg.d/intel-graphics.asc
```

Add Intel OneAPI Repository:
```
echo "deb https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/oneAPI.list
```

Ensure that Intel’s packages are correctly signed by fetching another public GPG key:
```
sudo apt-key adv --fetch-keys "https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB"
sudo apt update
```

Let's install now the Math Kernel Library.
- `intel-mkl`: Installs the Intel MKL runtime (optimized mathematical functions for HPC).
- `intel-oneapi-mkl`: Installs the full OneAPI version of MKL.
- `intel-oneapi-mkl-devel`: Includes developer files (headers, libraries) for compiling software with MKL.
```
sudo apt install intel-mkl intel-oneapi-mkl intel-oneapi-mkl-devel
```
Note: During the installation, press *Ok* and *No* in the huge purple screens.

Load Intel OneAPI environment variables:
```
source /opt/intel/oneapi/setvars.sh
```

And finally install OpenMPI:
- `openmpi-bin`: Installs OpenMPI binaries (like mpirun, mpiexec).
- `openmpi-common`: Installs OpenMPI configuration files.
- `libopenmpi-dev`: Installs development files (for compiling MPI applications).
```
sudo apt install openmpi-bin openmpi-common libopenmpi-dev
```


#### HPCC Compilation

It is time to download and compile the HPCC benchmark (we can work now in the *master* node). Given that all nodes need to have access to the executable, let's move to the distributed FS.
```
cd /shared/
```

We will need to clone the HPCC repository into out machine.
```
git clone https://github.com/icl-utk-edu/hpcc.git
cd hpcc/
```

In the `hpl/setup` folder you can find different makefiles for different processor architectures. Most of them are a bit outdated, so we will choose the closest one to our architecture and make the pertinent changes.
```
ls hpl/setup
```
<p align="center">
  <img src="https://github.com/user-attachments/assets/3b1913b4-8c04-4b0b-9161-abc32bf0cab6"  width="700">
</p>

Even though `Make.LinuxIntelIA64Itan2_eccMKL` was originally designed for Intel Itanium 2 processors, it should work for Intel Core processors adjusting a couple of parameters. First, the file must be moved one directory above:
```
mv /shared/hpcc/hpl/setup/Make.LinuxIntelIA64Itan2_eccMKL /shared/hpcc/hpl
```

Let's do the make now. It is important to set the architecture parameter to the same architecture name of the file:
```
make arch=LinuxIntelIA64Itan2_eccMKL
```

Some errors at compilation time might appear. Here are the ones I had to deal with and their possible solutions. However, each makefile executed in different systems can produce different errors. Some debugging must be probably needed to generate the final binary we are looking for.

- `gcc: error: unrecognized command-line option ‘-fno-alias’`
Open the file and edit the CCFLAGS parameter. Change `-fno-alias` for `-fno-strict-aliasing`
```
vim hpl/Make.LinuxIntelIA64Itan2_eccMKL
```
```yaml
CCFLAGS      = $(HPL_DEFS) -O3 -fno-strict-aliasing -Wall -mcpu=itanium2
```

- `cc1: error: bad value ‘itanium2’ for ‘-mtune=’ switch`
Open the file and edit the CCFLAGS parameter. Change `-mcpu=itanium2` for `-mtune=native` to automatically detect the architecture of your CPU.
```
vim hpl/Make.LinuxIntelIA64Itan2_eccMKL
```
```yaml
CCFLAGS      = $(HPL_DEFS) -O3 -fno-strict-aliasing -Wall -mtune=native
```

- `/libhpl.a  -lmkl_i2p -lpthread -lguide  -lm
  /usr/bin/ld: cannot find -lmkl_i2p: No such file or directory
  /usr/bin/ld: cannot find -lguide: No such file or directory
  collect2: error: ld returned 1 exit status`. Open the file and edit the LAlib parameter (which selects the Linear Algebra modules). Names might vary between MKL versions so let's set the new ones (the order is important since it will be the order in which the object files are linked).
```
vim hpl/Make.LinuxIntelIA64Itan2_eccMKL
```
```yaml
LAlib        = -lmkl_intel_lp64 -lmkl_intel_thread -lmkl_core -liomp5 -lpthread -lm
```


- `MPI_Address` and `MPI_Type_struct` errors. If you are using the HPCC version from the webpage instead of the one in Github, then you may encounter some more errors like these (given that it is a previous version). This happens because of a difference in how MPI functions are named in different versions. To update the names:
```
find . -type f -name "*.c" -o -name "*.h" | xargs -I {} cp {} {}.bak
find . -type f \( -name "*.c" -o -name "*.h" \) -exec sed -i 's/\bMPI_Address\b/MPI_Get_address/g' {} +
find . -type f \( -name "*.c" -o -name "*.h" \) -exec sed -i 's/\bMPI_Type_struct\b/MPI_Type_create_struct/g' {} +
```

Once the compilation is completed, we should have a new executable file `hpcc`.
<p align="center">
  <img src="https://github.com/user-attachments/assets/3fb6d105-da11-40bf-9eaf-24a934adb2c2"  width="700">
</p>


#### HPCC Parameters and Execution

Firstly we need to configure `_hpccinf.txt` where are located all the parameters for the HPL test. You can use a tooning tool (https://www.advancedclustering.com/act_kb/tune-hpl-dat-file/) to explore some optimal combinations depending on syour particular system or test some by hand.
```
mv _hpccinf.txt hpccinf.txt
vim hpccinf.txt
```

The ones I will use are the following ones:
```yaml
HPLinpack benchmark input file
Innovative Computing Laboratory, University of Tennessee
HPL.out      output file name (if any)
6            device out (6=stdout,7=stderr,file)
1            # of problems sizes (N)
8192         Ns
1            # of NBs
64           NBs
0            PMAP process mapping (0=Row-,1=Column-major)
1            # of process grids (P x Q)
1            Ps
1            Qs
16.0         threshold
1            # of panel fact
2            PFACTs (0=left, 1=Crout, 2=Right)
1            # of recursive stopping criterium
4            NBMINs (>= 1)
1            # of panels in recursion
2            NDIVs
1            # of recursive panel fact.
1            RFACTs (0=left, 1=Crout, 2=Right)
1            # of broadcast
1            BCASTs (0=1rg,1=1rM,2=2rg,3=2rM,4=Lng,5=LnM)
1            # of lookahead depth
1            DEPTHs (>=0)
2            SWAP (0=bin-exch,1=long,2=mix)
64           swapping threshold
0            L1 in (0=transposed,1=no-transposed) form
0            U  in (0=transposed,1=no-transposed) form
1            Equilibration (0=no,1=yes)
8            memory alignment in double (> 0)
##### This line (no. 32) is ignored (it serves as a separator). ######
0                               Number of additional problem sizes for PTRANS
1200 10000 30000                values of N
0                               number of additional blocking sizes for PTRANS
40 9 8 13 13 20 16 32 64        values of NB
```



After configuring the input file, let's set up the enviroment variables to be used for MPI:
```
export LD_LIBRARY_PATH=/opt/intel/oneapi/mkl/latest/lib/intel64:$LD_LIBRARY_PATH
echo 'export LD_LIBRARY_PATH=/opt/intel/oneapi/mkl/latest/lib/intel64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

Finally, run the MPI process in both worker nodes.
- `mpi run` launches the `hpcc` executable across multiple processes and machines.
- `-x LD_LIBRARY_PATH` exports the the environment variable to all MPI processes.
- `-np 4` specifies the number of MPI processes to start.
- `--host <ip node A>:<# processes>,<ip node B>:<# processes>` assigns processes to specific hosts.

To make sure MPI works properly, run a dummy command which returns the name of the nodes:
```
mpirun -np 4 --host 192.168.0.3:2,192.168.0.6:2 hostname
```
The ouput should be something similar to this:
<p align="center">
  <img src="https://github.com/user-attachments/assets/24554f2a-07ea-4342-bf56-1d1add4761a7"  width="700">
</p>

If it works fine, execute it with the `hpcc` file.
```
mpirun -x LD_LIBRARY_PATH -np 2 --host 192.168.0.10:1,192.168.0.4:1 hpcc
```

To check if it is working as expected, you can ran some commands like
```
mpstat -P ALL 1
```
or
```
htop
```
which show you the CPU ussage by each core or process.
<p align="center">
  <img src="https://github.com/user-attachments/assets/5d9e027d-58ff-4a14-b300-9e3f3a0624fe"  width="900">
</p>

It is important to note that we are launching the tests in two cores (one pero node) instead of four. The HPCC benchmark performs intensive computing over the CPU, testing it on a 100% usage. Deploying the tests in all the cores would cause kernel starvation (all CPU resources are occupied by user-space processes, leaving no time for the operating system kernel to perform critical background tasks). This can trigger a soft lockup warning (checked by the watchdog timer), freezing the system, making other processes crash and ending in an error. 

If your nodes have more than two CPUs, then just leaving one for the OS should be enough.

Finally, once all tests finsih, the summary of the results are written in an output file called `hpccoutf.txt` (there is an example in the repository).
