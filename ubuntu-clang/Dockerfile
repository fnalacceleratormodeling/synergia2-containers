FROM ubuntu:20.04 as bootstrap

ENV SPACK_ROOT=/opt/spack \
    CURRENTLY_BUILDING_DOCKER_IMAGE=1 \
    container=docker

ENV DEBIAN_FRONTEND=noninteractive   \
    LANGUAGE=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8

RUN apt-get -yqq update \
 && apt-get -yqq install --no-install-recommends \
        build-essential \
        ca-certificates \
        curl \
        file \
        g++ \
        gcc \
        gfortran \
        git \
        gnupg2 \
        iproute2 \
        locales \
        lua-posix \
        make \
        python3 \
        python3-pip \
        python3-setuptools \
        unzip \
 && locale-gen en_US.UTF-8 \
 && pip3 install boto3 \
 && rm -rf /var/lib/apt/lists/*

RUN mkdir $SPACK_ROOT && cd $SPACK_ROOT && \
    git clone https://github.com/spack/spack.git . && git checkout develop  && \
    mkdir -p $SPACK_ROOT/opt/spack

RUN ln -s $SPACK_ROOT/share/spack/docker/entrypoint.bash \
          /usr/local/bin/docker-shell \
 && ln -s $SPACK_ROOT/share/spack/docker/entrypoint.bash \
          /usr/local/bin/interactive-shell \
 && ln -s $SPACK_ROOT/share/spack/docker/entrypoint.bash \
          /usr/local/bin/spack-env

RUN mkdir -p /root/.spack \
 && cp $SPACK_ROOT/share/spack/docker/modules.yaml \
        /root/.spack/modules.yaml \
 && rm -rf /root/*.* /run/nologin $SPACK_ROOT/.git

# [WORKAROUND]
# https://superuser.com/questions/1241548/
#     xubuntu-16-04-ttyname-failed-inappropriate-ioctl-for-device#1253889
RUN [ -f ~/.profile ]                                               \
 && sed -i 's/mesg n/( tty -s \\&\\& mesg n || true )/g' ~/.profile \
 || true


WORKDIR /root
SHELL ["docker-shell"]

# Hack until spack fixes https://github.com/spack/spack/issues/27960
RUN sed -i 's/platform.release()/"5.4.100"/g' /opt/spack/var/spack/repos/builtin/packages/qt/package.py

# Creates the package cache
RUN spack spec hdf5+mpi

ENTRYPOINT ["/bin/bash", "/opt/spack/share/spack/docker/entrypoint.bash"]
CMD ["interactive-shell"]

# Build stage with Spack pre-installed and ready to be used
FROM bootstrap as builder

# Install OS packages needed to build the software
RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash perl cmake wget gcc-10 g++-10 gfortran-10 \
 && rm -rf /var/lib/apt/lists/*

# Get latest CMake, since we need 3.19 default is 3.16
RUN wget -qO- "https://cmake.org/files/v3.22/cmake-3.22.2-linux-x86_64.tar.gz" | tar --strip-components=1 -xz -C /usr/local

RUN wget https://apt.kitware.com/kitware-archive.sh && \
    chmod +x kitware-archive.sh && \
    ./kitware-archive.sh

# Install LLVM in builder
RUN apt-get -yqq update && apt-get -yqq install lsb-release wget software-properties-common
RUN wget https://apt.llvm.org/llvm.sh && \
    chmod +x llvm.sh && \
    ./llvm.sh 13 all

# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
RUN mkdir /opt/spack-environment \
&&  (echo "spack:" \
&&   echo "  compilers:" \
&&   echo "  - compiler:" \
&&   echo "      spec: clang@13.0.1" \
&&   echo "      paths:" \
&&   echo "        cc: /usr/bin/clang-13" \
&&   echo "        cxx: /usr/bin/clang++-13" \
&&   echo "        f77: /usr/bin/gfortran-10" \
&&   echo "        fc: /usr/bin/gfortran-10" \
&&   echo "      flags:" \
&&   echo "        cxxflags: -stdlib=libc++" \
&&   echo "      operating_system: ubuntu20.04" \
&&   echo "      target: x86_64" \
&&   echo "      modules: []" \
&&   echo "      environment: {}" \
&&   echo "      extra_rpaths:" \
&&   echo "      - /lib/gcc/x86_64-linux-gnu/10/libgfortran.so" \
&&   echo "  packages:" \
&&   echo "    all:" \
&&   echo "      target:" \
&&   echo "      - haswell" \
&&   echo "      compiler:" \
&&   echo "      - clang" \
&&   echo "      providers:" \
&&   echo "        mpi:" \
&&   echo "        - mpich" \
&&   echo "        blas:" \
&&   echo "        - openblas" \
&&   echo "        lapack:" \
&&   echo "        - openblas" \
&&   echo "  specs:" \
&&   echo "  - mpich@develop~fortran device=ch4 netmod=ofi" \
&&   echo "  - hdf5@1.12.1+hl+mpi" \
&&   echo "  - fftw+mpi+openmp" \
&&   echo "  - gsl" \
&&   echo "  - python" \
&&   echo "  - py-numpy" \
&&   echo "  - py-mpi4py" \
&&   echo "  - py-h5py+mpi" \
&&   echo "  - py-pytest" \
&&   echo "  - ninja" \
&&   echo "  concretization: together" \
&&   echo "  view: /opt/view" \
&&   echo "  config:" \
&&   echo "    install_tree: /opt/software") > /opt/spack-environment/spack.yaml

# Find externals
RUN spack external find perl
RUN spack external find ncurses
RUN spack external find gettext
RUN spack external find cmake

# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment && \
    spack env activate . && \
    spack install --fail-fast && \
    spack gc -y

# Strip all the binaries
RUN find -L /opt/view/* -type f -exec readlink -f '{}' \; | \
    xargs file -i | \
    grep 'charset=binary' | \
    grep 'x-executable\|x-archive\|x-sharedlib' | \
    awk -F: '{print $1}' | xargs strip -s

# Modifications to the environment that are necessary to run
RUN cd /opt/spack-environment && \
    spack env activate --sh -d . >> /etc/profile.d/z10_spack_environment.sh

# Bare OS image to run the installed executables
FROM ubuntu:20.04

COPY --from=builder /opt/spack-environment /opt/spack-environment
COPY --from=builder /opt/software /opt/software
COPY --from=builder /opt/view /opt/view
COPY --from=builder /etc/profile.d/z10_spack_environment.sh /etc/profile.d/z10_spack_environment.sh

# Noninteractive install!
ENV DEBIAN_FRONTEND=noninteractive   \
    LANGUAGE=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8 \
    TZ=Etc/UTC

RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash cmake git wget gcc-10 g++-10 gfortran-10 \
 && rm -rf /var/lib/apt/lists/*
   
# Install LLVM in final container
RUN apt-get -yqq update && apt-get -yqq install lsb-release wget software-properties-common
RUN wget https://apt.llvm.org/llvm.sh && \
    chmod +x llvm.sh && \
    ./llvm.sh 13 all

# Get latest CMake, since we need 3.19 default is 3.16
RUN wget -qO- "https://cmake.org/files/v3.22/cmake-3.22.2-linux-x86_64.tar.gz" | tar --strip-components=1 -xz -C /usr/local
  
ENTRYPOINT ["/bin/bash", "--rcfile", "/etc/profile", "-l"]

