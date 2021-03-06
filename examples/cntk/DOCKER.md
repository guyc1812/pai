
# CNTK on PAI docker env

## Contents

1. [Basic environment](#basic-environment)
2. [Advanced environment](#advanced-environment)

## Basic environment

First of all, PAI runs all jobs in Docker container.

[Install Docker-CE](https://docs.docker.com/install/linux/docker-ce/ubuntu/) if you haven't. Register an account at public Docker registry [Docker Hub](https://hub.docker.com/) if you do not have a private Docker registry.

You can also jump to [CNTK examples](#cntk-examples) using [pre-built images](https://hub.docker.com/r/openpai/pai.example.cntk/) on Docker Hub.

We need to build a CNTK image with GPU support to run CNTK workload on PAI, this can be done in three steps:

1. Build a base Docker image for PAI. We prepared a [base Dockerfile](../../job-tutorial/Dockerfiles/cuda8.0-cudnn6/Dockerfile.build.base) which can be built directly.

    ```bash
    $ cd ../job-tutorial/Dockerfiles/cuda8.0-cudnn6
    $ sudo docker build -f Dockerfile.build.base \
    >                   -t pai.build.base:hadoop2.7.2-cuda8.0-cudnn6-devel-ubuntu16.04 .
    $ cd -
    ```

2. Build a openmpi Docker image. We prepared a [mpi Dockerfile](../../job-tutorial/Dockerfiles/cuda8.0-cudnn6/Dockerfile.build.mpi) which can be built based on the base image.

    ```bash
    $ cd ../job-tutorial/Dockerfiles/cuda8.0-cudnn6
    $ sudo docker build -f Dockerfile.build.mpi \
    >                   -t pai.build.mpi:openmpi1.10.4-hadoop2.7.2-cuda8.0-cudnn6-devel-ubuntu16.04 .
    $ cd -
    ```

3. Prepare CNTK envoriment in a [Dockerfile](./Dockerfile.example.cntk) using the base image.

    Write a CNTK Dockerfile and save it to `Dockerfile.example.cntk`:

    ```dockerfile
    FROM pai.build.mpi:openmpi1.10.4-hadoop2.7.2-cuda8.0-cudnn6-devel-ubuntu16.04

    ENV CNTK_VERSION=2.0.beta11.0

    RUN apt-get -y update && \
        apt-get -y install git \
            fuse \
            golang \
            libjasper1 \
            libjpeg8 \
            libpng12-0 \
            libgfortran3 && \
        apt-get clean && \
        rm -rf /var/lib/apt/lists/*

    WORKDIR /

    # Install hdfs-mount
    RUN git clone --recursive https://github.com/Microsoft/hdfs-mount.git && \
        cd hdfs-mount && \
        make -j $(nproc) && \
        make test && \
        cp hdfs-mount /bin && \
        cd .. && \
        rm -rf hdfs-mount

    # Install Anaconda
    RUN ANACONDA_PREFIX="/root/anaconda3" && \
        ANACONDA_VERSION="3-4.1.1" && \
        ANACONDA_SHA256="4f5c95feb0e7efeadd3d348dcef117d7787c799f24b0429e45017008f3534e55" && \
        wget -q https://repo.continuum.io/archive/Anaconda${ANACONDA_VERSION}-Linux-x86_64.sh && \
        echo "$ANACONDA_SHA256 Anaconda${ANACONDA_VERSION}-Linux-x86_64.sh" | sha256sum --check --strict - && \
        chmod a+x Anaconda${ANACONDA_VERSION}-Linux-x86_64.sh && \
        ./Anaconda${ANACONDA_VERSION}-Linux-x86_64.sh -b -p ${ANACONDA_PREFIX} && \
        rm -rf Anaconda${ANACONDA_VERSION}-Linux-x86_64.sh && \
        $ANACONDA_PREFIX/bin/conda clean --all --yes

    ENV PATH=/root/anaconda3/bin:/usr/local/mpi/bin:$PATH \
        LD_LIBRARY_PATH=/root/anaconda3/lib:/usr/local/mpi/lib:$LD_LIBRARY_PATH

    # Get CNTK Binary Distribution
    RUN CNTK_VERSION_DASHED=$(echo $CNTK_VERSION | tr . -) && \
        CNTK_SHA256="2e60909020a0f553431dc7f7818401cc1bb2c99eef307d65bb552c497993593a" && \
        wget -q https://cntk.ai/BinaryDrop/CNTK-${CNTK_VERSION_DASHED}-Linux-64bit-GPU.tar.gz && \
        echo "$CNTK_SHA256 CNTK-${CNTK_VERSION_DASHED}-Linux-64bit-GPU.tar.gz" | sha256sum --check --strict - && \
        tar -xzf CNTK-${CNTK_VERSION_DASHED}-Linux-64bit-GPU.tar.gz && \
        rm -f CNTK-${CNTK_VERSION_DASHED}-Linux-64bit-GPU.tar.gz && \
        wget -q https://raw.githubusercontent.com/Microsoft/CNTK-docker/master/ubuntu-14.04/version_2/${CNTK_VERSION}/gpu/runtime/install-cntk-docker.sh \
            -O /cntk/Scripts/install/linux/install-cntk-docker.sh && \
        /bin/bash /cntk/Scripts/install/linux/install-cntk-docker.sh && \
        /root/anaconda3/bin/conda clean --all --yes && \
        rm -rf /cntk/cntk/python

    ENV PATH=/cntk/cntk/bin:$PATH \
        LD_LIBRARY_PATH=/cntk/cntk/lib:/cntk/cntk/dependencies/lib:$LD_LIBRARY_PATH

    WORKDIR /root
    ```

    Build the Docker image from `Dockerfile.example.cntk`:

    ```bash
    $ sudo docker build -f Dockerfile.example.cntk -t pai.example.cntk .
    ```

    Push the Docker image to a Docker registry:

    ```bash
    $ sudo docker tag pai.example.cntk USER/pai.example.cntk
    $ sudo docker push USER/pai.example.cntk
    ```
    *Note: Replace USER with the Docker Hub username you registered, you will be required to login before pushing Docker image.*


## Advanced environment

You can skip this section if you do not need to prepare other dependencies.

You can customize runtime CNTK environment in [Dockerfile.example.cntk](./Dockerfile.example.cntk), for example, adding other dependeces in Dockerfile.
