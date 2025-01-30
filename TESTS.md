CPU - HPC Challenge Benchmark

    sudo apt update
    sudo apt install openmpi-bin openmpi-common libopenmpi-dev intel-oneapi-mkl intel-oneapi-mkl-devel

    If error with intel-oneapi-mkl intel-oneapi-mkl-devel:
        wget -qO - https://repositories.intel.com/graphics/intel-graphics.key | sudo tee /etc/apt/trusted.gpg.d/intel-graphics.asc
        
        sudo apt-key adv --fetch-keys "https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB"
        
        sudo apt update
        
        sudo apt install intel-mkl intel-oneapi-mkl intel-oneapi-mkl-devel



    wget https://hpcchallenge.org/projectsfiles/hpcc/download/hpcc-1.5.0.tar.gz
    tar -xvzf filename.tar.gz

    cd hpcc-1.5.0/hpl/setup
    ls
    mv Make.LinuxIntelIA64Itan2_eccKML ..

    make arch=LinuxIntelIA64Itan2_eccKML

        gcc error: unrecognized command-line option '-fno-alias'; did you mean '-fno-libs='?
        -fno-strict-aliasing

        cc1: error: bad value 'itanium2' for '-mtune' switch
        -cpu=itanium2    ->     -mtune=native

        # Backup files before modifying
        find . -type f -name "*.c" -o -name "*.h" | xargs -I {} cp {} {}.bak

        # Replace MPI_Address with MPI_Get_address
        find . -type f \( -name "*.c" -o -name "*.h" \) -exec sed -i 's/\bMPI_Address\b/MPI_Get_address/g' {} +

        # Replace MPI_Type_struct with MPI_Type_create_struct
        find . -type f \( -name "*.c" -o -name "*.h" \) -exec sed -i 's/\bMPI_Type_struct\b/MPI_Type_create_struct/g' {} +



General System - stress-ng/sysbench

Dik I/O - IOZone

Network - iperf/netcat