# Cluster Creation and Testing with VirtualBox VMs and Docker Containers

The following project aims to create a cluster of machines departing from two different approaches: Virtual machines and Docker containers. By implementing both, it is possible to compare the performance and efficiency of virtualized versus containerized clustering. [Cluster Creation and Performance Testing in VirtualBox](Cluster%20Creation%20and%20Performance%20Testing%20in%20VirtualBox.md). 

The first approach for creating a cluster is through **Virtual Machines (VMs) with Oracle VirtualBox** [^1]. In this setup, multiple VMs are interconnected via a virtual switch, simulating a distributed computing environment. [Container Creation and Performance Testing with Docker](Container%20Creation%20and%20Performance%20Testing%20with%20Docker.md).  

The cluster consists of a *master* node (or *node01*) and some *worker* nodes (*node02*, *node03*, ..., *nodeXX*). The *master* serves as DNS server (resolving hostnames within the cluster), DHCP server (assigning IP addresses dynamically to worker nodes), and gateway (managing communication between nodes and external networks).  

The second approach is a containerized cluster using **Docker containers** [^2]. It can be set up using Dockerfiles and Docker Compose, ensuring portability and ease of deployment. This setup also consists of a *master* container and a set of *worker* containers.  

The *master* container is mainly used for defining and starting the tasks to be executed by the worker containers (*worker1*, *worker2*, ..., *workerX*).  

For each virtual machine and container, the resources will be the same (in order to make fair comparisons):  

- 2 CPUs.  
- 2 GB of RAM.  
- 20 GB of hard disk storage.  

After their creation, the performance of both systems will be evaluated by carrying out the following tests:  

- `HPC Challenge Benchmark`, which tests computational efficiency.  
- `stress-ng` and `sysbench`, which evaluate CPU, memory, and disk performance.  
- `IOZone`, which analyzes file system I/O speed.  
- `iperf`, which measures network throughput and latency.  

[^1]: [Oracle VirtualBox](https://www.virtualbox.org/)  
[^2]: [Docker](https://www.docker.com/)  
