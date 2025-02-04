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
    - [HPCC Compilation](#hpcc-compilation)
    - [HPCC Parameters and Execution](#hpcc-parameters-and-execution)
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
      git \
      make \
      openmpi-bin \
      openmpi-common \
      libopenmpi-dev
  ```
  Installs most of the required packages for the container.

- ```yaml
  RUN wget -qO - https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | tee /etc/apt/trusted.gpg.d/intel-sw-products.asc
  RUN echo "deb https://apt.repos.intel.com/oneapi all main" > /etc/apt/sources.list.d/oneAPI.list
  RUN apt-get update && apt-get install -y intel-mkl intel-oneapi-mkl intel-oneapi-mkl-devel
  ```
  Installs MKL and OpenMPI.
  
- ```
  WORKDIR /mnt/shared
  RUN git clone https://github.com/icl-utk-edu/hpcc.git
  WORKDIR /mnt/shared/hpcc
  RUN mv hpl/setup/Make.LinuxIntelIA64Itan2_eccMKL hpl/
  
  RUN sed -i 's/-fno-alias/-fno-strict-aliasing/' hpl/Make.LinuxIntelIA64Itan2_eccMKL && \
      sed -i 's/-mcpu=itanium2/-mtune=native/' hpl/Make.LinuxIntelIA64Itan2_eccMKL && \
      sed -i 's|-lmkl_i2p -lpthread -lguide|-lmkl_intel_lp64 -lmkl_intel_thread -lmkl_core -liomp5 -lpthread -lm|' hpl/Make.LinuxIntelIA64Itan2_eccMKL
  
  RUN make arch=LinuxIntelIA64Itan2_eccMKL
  ```
  Donwloads and compiles HPCC Benchmark.

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
      software-properties-common \
      stress-ng \
      sysbench \
      iozone3 \
      iperf3
  ```
  Installs most of the required packages for the container.

- ```
  RUN wget -qO - https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | tee /etc/apt/trusted.gpg.d/intel-sw-products.asc
  RUN echo "deb https://apt.repos.intel.com/oneapi all main" > /etc/apt/sources.list.d/oneAPI.list
  RUN apt-get update && apt-get install -y intel-mkl intel-oneapi-mkl intel-oneapi-mkl-devel
  ```
  Installs MKL and OpenMPI.

- ```
  RUN rm -rf /var/lib/apt/lists/*
  ```
  Removes the package lists stored (reducing image size and increasing security).



## Docker Compose
Let's move to the docker compose file. This document defines the containers to be made as well as their properties: networks, resources, volumes...

### Master Container
```yaml
  master:
    build:
      context: .
      dockerfile: Dockerfile.master
    container_name: master
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

By running `docker exec` command, it is possible to open a bash insidea specified container:
```
docker exec -it master sh
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


