# Container Creation and Performance Testing with Docker

## Table of Contents
- [Goals](#goals)
- [Prerequisite Setup](#prerequisite-setup)
- [Docker Installation](#docker-installation)
- [Dockerfiles](#dockerfiles)
  - [Master Dockerfile](#master-dockerfile)
  - [Worker Dockerfile](#worker-dockerfile)
- [Docker Compose](#docker-compose)
   - [Master Container](#master-container)
   - [Worker Containers](#worker-containers)
   - [Network](#network)
   - [Volumes](#volumes)
- [Debugging](#debugging)
- [Performance Tests](#performance-tests)
  - [CPU Tests (HPCC)](#cpu-tests-hpcc)
  - [General System Tests](#general-system-tests)
    - [Stress-ng](#stress-ng)
    - [Sysbench](sysbench)
  - [Disk Tests (IOZone)](#disk-tests-iozone)
  - [Network Tests (iperf)](#network-tests-iperf)

    
## Goals
The goal is to set up and manage a set of ontainers using Docker. Three containers will be created:
- 1 master.
- 2 workers (where the tests will be performed).
Each container will have limited resources (2 GB of RAM and 2 CPUs) and they will share a file system.

After setting up the containers with their respective dockerfiles and docker compose, some performance tests will be carried out.
- CPU: HPC Challenge Benchmark for high-performance computation benchmarking
- General System: stress-ng and sysbench to evaluate CPU, memory and diskperformance.
- Disk: IOZone to test filesystem I/O.
- Network: iperf to measure network throughput and latency between VMs.


## Prerequisite Setup
- Windows Subsystem for Linux (WSL 2).


## Docker Installation
Before starting, let's make sure that the version of the subsystem in the machine is WSL 2 (necessary to run Docker). You can run in your Windows PowerShell the following command
```
wsl -l -v
```
which should output your WSL version and distribution.
```
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

If your WSL version is 1, update it by running:
```
wsl --set-version Ubuntu 2
```

Update your package lists and install dependencies:
```
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

And sdd Docker’s official GPG key:
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo tee /etc/apt/keyrings/docker.asc > /dev/null
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Now let's proceed to set up the repository and install docker engine anc CLI:
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

We can check if docker is correctly installed by running:
```
docker --version
```

Finally, let's activate the service and run a test container:
```
sudo service docker start
docker run hello-world
```

## Dockerfiles
Let's start by defining the images for our containers. We create two different dockerfiles, one for the *master*  container (`Dockerfile.master`) and the other one for the *worker* containers (`Dockerfile.workers`).

### Master Dockerfile
The purpose of `Dockerfile.master` is to set up an environment ready for launching tests on the worker nodes. It configures the OpenMPI package, downloads, and compiles the HPCC benchmark, ensuring the system has the final executable ready to use.

- ```
  FROM ubuntu:latest
  ```
  It pulls the latest version of the official Ubuntu image from Docker Hub. This serves as the foundation for the container, providing a minimal Ubuntu environment to build upon.
  
- ```
  ENV DEBIAN_FRONTEND=noninteractive
  ```
  It tells apt-get to run in non-interactive mode, meaning it won’t ask for user input (useful to avoid the container getting stuck in questions when installing packages).
  
- ```
  RUN apt-get update && apt-get install -y \
      vim \
      wget \
      gnupg \
      git \
      make \
      hwloc \
      iozone3 \
      iputils-ping \
      openmpi-bin \
      openmpi-common \
      libopenmpi-dev \
      openssh-client
  ```
  Installs most of the required packages for the container.

- ```yaml
  RUN wget -qO - https://repositories.intel.com/graphics/intel-graphics.key | tee /etc/apt/trusted.gpg.d/intel-graphics.asc
  RUN echo "deb https://apt.repos.intel.com/oneapi all main" | tee /etc/apt/sources.list.d/oneAPI.list
  RUN apt-key adv --fetch-keys "https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB"
  RUN apt-get update && apt-get install -y intel-mkl intel-oneapi-mkl intel-oneapi-mkl-devel
  ```
  Installs MKL and OpenMPI.
  
- ```
  RUN git clone https://github.com/icl-utk-edu/hpcc.git && \
      cd hpcc && \
      mv hpl/setup/Make.LinuxIntelIA64Itan2_eccMKL hpl/ && \
      sed -i 's/-fno-alias/-fno-strict-aliasing/' hpl/Make.LinuxIntelIA64Itan2_eccMKL && \
      sed -i 's/-mcpu=itanium2/-mtune=native/' hpl/Make.LinuxIntelIA64Itan2_eccMKL && \
      sed -i 's|-lmkl_i2p -lpthread -lguide|-lmkl_intel_lp64 -lmkl_intel_thread -lmkl_core -liomp5 -lpthread -lm|' hpl/Make.LinuxIntelIA64Itan2_eccMKL && \
      make arch=LinuxIntelIA64Itan2_eccMKL
  ```
  Donwloads and compiles HPCC Benchmark.

- ```
  ENV LD_LIBRARY_PATH=/opt/intel/oneapi/mkl/latest/lib/intel64:$LD_LIBRARY_PATH
  ENV OMPI_ALLOW_RUN_AS_ROOT=1
  ENV OMPI_ALLOW_RUN_AS_ROOT_CONFIRM=1
  ```
  Sets some environment variables for OpenMPI.

- ```
  RUN rm -rf /var/lib/apt/lists/*
  ```
  Removes the package lists stored (reducing image size and increasing security).


### Worker Dockerfile
The purpose of `Dockerfile.workers` is to set up an environment ready for executing the performance tests. It configures the OpenMPI package, and installs much others (`stress-ng`, `sysbench`, `iozone3` and `iperf3`).

- ```
  FROM ubuntu:latest
  ```
  It pulls the latest version of the official Ubuntu image from Docker Hub. This serves as the foundation for the container, providing a minimal Ubuntu environment to build upon.
  
- ```
  ENV DEBIAN_FRONTEND=noninteractive
  ```
  It tells apt-get to run in non-interactive mode, meaning it won’t ask for user input (useful to avoid the container getting stuck in questions when installing packages).
  
- ```
  RUN apt-get update && apt-get install -y \
      net-tools \
      vim \
      wget \
      gnupg \
      curl \
      hwloc \
      openmpi-bin \
      iputils-ping \
      stress-ng \
      sysbench \
      iozone3 \
      iperf3 \
      openssh-server
  ```
  Installs most of the required packages for the container.

- ```
  RUN wget -qO - https://repositories.intel.com/graphics/intel-graphics.key | tee /etc/apt/trusted.gpg.d/intel-graphics.asc
  RUN echo "deb https://apt.repos.intel.com/oneapi all main" | tee /etc/apt/sources.list.d/oneAPI.list
  RUN apt-key adv --fetch-keys "https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB"
  RUN apt-get update && apt-get install -y intel-mkl intel-oneapi-mkl intel-oneapi-mkl-devel
  ```
  Installs MKL and OpenMPI.

- ```
  ENV LD_LIBRARY_PATH=/opt/intel/oneapi/mkl/latest/lib/intel64:$LD_LIBRARY_PATH
  ```
  Sets some environment variables for OpenMPI.

- ```
  RUN mkdir -p /var/run/sshd \
      && echo "PermitRootLogin yes" >> /etc/ssh/sshd_config \
      && echo "PasswordAuthentication no" >> /etc/ssh/sshd_config \
      && echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config
  EXPOSE 22
  ```
  Configures SSH service and exposes port 22.
  
- ```
  RUN rm -rf /var/lib/apt/lists/*
  ```
  Removes the package lists stored (reducing image size and increasing security).

- ```
  CMD ["/usr/sbin/sshd", "-D"]
  ```
  Initiates SSH daemon in the foreground when the container runs.



## Docker Compose
Let's move to the docker compose file. This document defines the containers to be made as well as their properties: networks, resources, volumes...

### Master Container
```yaml
  master:
    build:
      context: .
      dockerfile: Dockerfile.master
    container_name: master
    privileged: true
    networks:
      - internal-network
    volumes:
      - /mnt/shared_volume:/mnt/shared
    command: tail -f /dev/null
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '2.0'
          memory: 2G
```

- `master:`: This defines the service named master in the Docker Compose configuration.
  - `build:`: 
    - `context: .`: Specifies the build context, which is the directory where Docker will look for the Dockerfile.
    - `dockerfile: Dockerfile.master`: Specifies the name of the Dockerfile to use for building the image.
  - `container_name: master`: This specifies the name of the container when it is created.
  - `privileged: true`: Runs the container in privileged mode (necessary for binding processes to processors).
  - `networks:`:
    - `- internal-network`: This sets the network(s) the container will connect to. Here, it connects to a custom network called internal-network.
  - `volumes:`
    - `- /mnt/shared_volume:/mnt/shared`: This mounts a directory or volume from the host to the container.
/mnt/shared_volume on the host is mapped to /mnt/shared inside the container.
  - `command: tail -f /dev/null`: The command field overrides the default command specified in the Dockerfile.
Makes the container run indefinitely without exiting.
  - `deploy:`: Deployment-related options.
    -`resources:`
      - `limits:`
        - `cpus: '2.0'`:  Specifies a limit on the CPU usage, setting it to a maximum of 2 CPU cores.
        - `memory: 2G`:  Limits the container's memory usage to 2GB.
      - `reservations:`
        - `cpus: '2.0'`:  Reserves 2 CPU cores for the container (at least 2 CPUs will be allocated for the container when it runs).
        - `memory: 2G`:  Reserves 2 GB of memory for the container (at least 2 GB of memory will be allocated for the container when it runs).


### Worker Containers
```yaml
worker1:
    build:
      context: .
      dockerfile: Dockerfile.workers
    container_name: worker1
    privileged: true
    networks:
      - internal-network
    volumes:
      - /mnt/shared_volume:/mnt/shared
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '2.0'
          memory: 2G
```

### Network
We also need to define the network:

```yaml
networks:
  internal-network:
    driver: bridge
```
- `networks:`: This is the section where Docker networks are defined for the services in the docker-compose.yml file. Networks allow containers to communicate with each other, and specifying them ensures that services are connected to the correct networks for internal or external communication.
  - `internal-network:`: This defines a network named internal-network (the name is arbitrary).
    - `driver: bridge`: This specifies the network driver to use (bridge driver is the default network driver in Docker). It creates a network that allows containers to communicate with each other on the same host. Containers connected to a bridge network can communicate with each other using their container names as hostnames.
   
### Volumes
We defined in file a volume to be mounted `mnt/shared_volume:/mnt/shared`. However, this volume doesn't exist in our system. In order to have a shared file system between the containers, we will create, format, and mount a virtual disk on them:

```
sudo mkdir -p /mnt/volumes
sudo fallocate -l 20G /mnt/volumes/shared_volume.img
sudo mkfs.ext4 /mnt/volumes/shared_volume.img
sudo mkdir -p /mnt/shared_volume
sudo mount -o loop /mnt/volumes/shared_volume.img /mnt/shared_volume
```

This way we created a 20 GB ext4 FS which will be mounted on our containers.



## Running the Containers

Once the dockerfiles are ready, it is time to build them:
```
docker build -t master-image -f Dockerfile.master .
docker build -t worker-image -f Dockerfile.workers .
```

The output should be similar to this one:
<p align="center">
  <img src="https://github.com/user-attachments/assets/b6bce2c2-edcf-4dc8-962b-6f266910fff9"  width="700">
</p>

Now we can proceed to execute the docker compose:
```
docker compose -f docker-compose.yml up -d
```
<p align="center">
  <img src="https://github.com/user-attachments/assets/e5d9c06a-0744-4f7d-9eb9-092be6f90a46"  width="500">
</p>

By running `docker exec` command, it is possible to open an interactive bash inside a specific container:
```
docker exec -it master sh
```

We will need to perform some extra steps to configured the passwordless SSH manually. First, open a bash in the worker containers and run the command `ifconfig` to get the IPs of the containers. In my case:
- worker1: 172.18.0.4
- worker2: 172.18.0.3

Now, we need to generate a key from the master container. Open it and run:
```
ssh-keygen -t rsa -b 2048 -f /root/.ssh/id_rsa -N ""
cat /root/.ssh/id_rsa.pub
```

Copy the output and open the workers. We will manually paste it in their corresponding files (substitute for you key):
```
echo "<public-key-content>" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

It is possible to test it now. If everything worked fine, you should be able to passwordless ssh your worker containers from the master. Important: test both connections before proceeding.
```
ssh root@172.18.0.4
```
```
ssh root@172.18.0.3
```


## Debugging

It is common to have all type of issues while building or executing the containers. In order to make troubleshooting easier, here are some commands that can be useful:

- ```
  docker ps -a
  ```
  This command is used to list all the containers on your system, including those that are currently running and those that have stopped. 

- ```
  docker logs master
  ```
  This command is used to view the logs of a running or stopped Docker container. The logs contain information about the container's execution, including output from running processes, errors, and other system messages generated by the container.
   
- ```
  docker compose down
  ```
  This command stops and removes all containers, networks, and volumes defined in your `docker-compose.yml` file.

- ```
  docker system prune -af
  ```
  This command removes all unused data, including containers, networks, images, and volumes.



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

Once the containers are running, we can find the `hpcc` executable in `/`. Let's move it to the file system so all the containers can acces it.
```
mv /hpcc /mnt/shared/
cd /mnt/shared/hpcc
```

We need now to configure the `_hpccinf.txt` where are located all the parameters for the HPL test. You can use a tooning tool (https://www.advancedclustering.com/act_kb/tune-hpl-dat-file/) to explore some optimal combinations depending on syour particular system or test some by hand.
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

Finally, run the MPI process in both worker nodes.
- `mpi run` launches the `hpcc` executable across multiple processes and machines.
- `-x LD_LIBRARY_PATH` exports the the environment variable to all MPI processes.
- `-np 4` specifies the number of MPI processes to start.
- `--host <ip node A>:<# processes>,<ip node B>:<# processes>` assigns processes to specific hosts.

To make sure MPI works properly, run a dummy command which returns the name of the nodes:
```
mpirun -np 4 --host 172.18.0.4:2,172.18.0.3:2 hostname
```
The ouput should be something similar to this:
<p align="center">
  <img src="https://github.com/user-attachments/assets/24554f2a-07ea-4342-bf56-1d1add4761a7"  width="700">
</p>

If it works fine, execute it with the `hpcc` file.
```
mpirun -x LD_LIBRARY_PATH -np 2 --host 172.18.0.3:1,172.18.0.2:1 hpcc
```

The output will be saved in the `hpccoutf.txt` file.


### General System Tests

#### Stress-ng

Each test will be running for 60 seconds. The main metrics we will focus on are:
| Metric | Description |
| ------------- | ------------- |
| stressor | The test being run |
| bogo ops | Total bogus operations performed |
| real time (s) | Total elapsed time for the test |
| usr time (s) | Time spent executing in user space |
| sys time (s) | Time spent in kernel space |
| bogo ops/s (real time) | Operations per second, based on elapsed time |
| bogo ops/s (usr+sys time) | Operations per second, based on actual CPU work done |
| CPU used per instance (%) | CPU utilization per stressor instance |
| RSS Max (KB) | Maximum resident set size |


- **CPU**
  In order to start two instances of *stress*, one in each worker:
  ```
  mpirun -x LD_LIBRARY_PATH -np 2 --host 172.18.0.4:1,172.18.0.3:1 stress-ng --cpu 2 --timeout 60s --metrics
  ```

- **Memory**
  
  Similarly, we will allocate 1GB of memory per node (2GB per node).
  ```
  mpirun -x LD_LIBRARY_PATH -np 2 --host 172.18.0.4:1,172.18.0.3:1 stress-ng --vm 2 --vm-bytes 1G --timeout 60s --metrics
  ```

- **Disk (I/O)**
  
  Let's run two processes to write and read from disk.
  ```
  mpirun -x LD_LIBRARY_PATH -np 2 --host 172.18.0.4:1,172.18.0.3:1 stress-ng --hdd 1 --timeout 60s --metrics
  ```


#### Sysbench
An alternative to `stress-ng` is `sysbench`, a benchmarking tool for evaluating system performance. 

To make things more readable, we will create a file indicating the hosts where the MPI commands mus be executed. This file must contain the IP address of the worker nodes together with their maximum number of slots (CPUs).
```
vim hosts.txt
```
```yaml
172.18.0.4 slots=2
172.18.0.3 slots=2
```

Moreover, we will introduce a new option to the `mpirun` command called `--bind-to core`, which ensures that each MPI process runs on a specific CPU core. This prevents the operating system from moving the process around between cores. Processor affinity can lead to an improvement in performance, specially in CPU intensive applications.

- **CPU**

  We will start by launching a process thread per CPU:
  ```
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench cpu --threads=2 run
  ```

- **Memory**
  
  Similarly, we will start some threads to measure RAM speed by reading/writing data.
  ```
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench memory --threads=2 run
  ```


- **Disk (I/O)**
  
  The option `--file-test-mode=<mode>` will be used to define four different tests.
  - `seqrd`: sequential reads.
  - `rndrd`: random reads.
  - `seqwr`: sequential writes.
  - `rndwr`: random writes.
  
  For all tests except `seqwr`, we need to prepare some test files before. For all of them we need to delete them afterwards.
  ```
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio --threads=2 --file-test-mode=seqrd prepare
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio --threads=2 --file-test-mode=seqrd run
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio cleanup
  ```
  ```
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio --threads=2 --file-test-mode=rndrd prepare
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio --threads=2 --file-test-mode=rndrd run
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio cleanup
  ```
  ```
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio --threads=2 --file-test-mode=seqwr run
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio cleanup
  ```
  ```
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio --threads=2 --file-test-mode=rndwr prepare
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio --threads=2 --file-test-mode=rndwr run
  mpirun -x LD_LIBRARY_PATH --bind-to core -np 2 --map-by ppr:1:core --hostfile hosts.txt sysbench fileio cleanup
  ```


### Disk Tests (IOZone)

IOZone is a popular benchmarking tool for file system performance, which can test various file operations like random read/write, sequential read/write, and more.

The options we will use for this command are:
- `-a`: Used to select full automatic mode.
- `-g`: Set maximum file size (in Kbytes) for auto mode. 
- `-i 0`: write/rewrite.
- `-i 1`: read/re-read.
- `-i 2`: random-read/write.
- `-i 3`: read-backwards.
- `-i 4`: re-write-record.
- `-i 5`: stride-read.
- `-i 6`: fwrite/re-fwrite.
- `-i 7`: fread/Re-fread.
- `-i 8`: random mix.
- `-i 9`: pwrite/Re-pwrite.
- `-i 10`: pwrite/Re-pwrite.
- `-i 11`: pwritev/Re-pwritev.
- `-i 12`: preadv/Repreadv.
- `-t`: Specify the number of threads to use for the test.
- `-+m <file>`: Specify the file containing worker node details (for cluster testing).

More information about what each operation means at https://www.iozone.org/docs/IOzone_msword_98.pdf.

In order to set the tests for the worker nodes and forcing them to work in the shared FS, we will create a configuration file with the following characteristics: the file contains one line for each client. The fields are space delimited. Field 1 is the client name. Field 2 is the working directory, on the client, where Iozone will run. Field 3 is the path to the executable Iozone on the client.

```
vim iozone_config
```
```yaml
172.18.0.4 /mnt/shared /usr/bin/iozone
172.18.0.3 /mnt/shared /usr/bin/iozone  
```
Since the process needs to comunicate with the workers to run the tests and the original configuration uses RSH (currently depracated), it is necessary to force the use of SSH:
```
export RSH=ssh
```

Now we test the disk performance. We can start a process in each worker node, each of them with two threads. It is also important to note that a writing test must be ran always before a reading test so IOZone can encounter the files to work with.

```
iozone -+m iozone_config -t2 -i0 -i1 -i2 -i3 -i4- -i5 -i6 -i7 -i8 -i9 -i10 -i11 -i12
```

Instead, it is possible to run the automatic mode (with only one thread pero node) which varies the record sizes from 4k to 16M and file sizes from 64k to 512M. 
```
iozone -+m iozone_config -a -i0 -i1 -i2 -i3 -i4- -i5 -i6 -i7 -i8 -i9 -i10 -i11 -i12 -g 1G
```


## Network Tests (iperf)

### Iperf
`iPerf3` is a tool for active measurements of the maximum achievable bandwidth on IP networks. It supports tuning of various parameters related to timing, buffers and protocols (TCP, UDP, SCTP with IPv4 and IPv6). 

We will run `iperf` in one container as a server and in another as a client. Let's suppose *worker1* and *worker2* respectively. To initialize the server, run on *worker1*:
```
iperf3 -s
```
which will listen on port 5201 by default (it is possible to modify the port with the `-p` option).

On *node03* run the client (replacing the IP with the one of *node02*):
```
iperf3 -c 172.18.0.4
```
<p align="center">
  <img src="https://github.com/user-attachments/assets/87eab153-d041-4c98-8e14-2b4c9ee8fbcd"  width="450">
</p>

This test measures the performance of the network using TCP comunication by default. In order to re-do the test for UDP:
```
iperf3 -c 172.18.0.4 -u
```



