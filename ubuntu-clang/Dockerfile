# Build stage with Spack pre-installed and ready to be used
FROM spack/ubuntu-jammy:latest as builder

# Install OS packages needed to build the software
RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash perl wget cmake tar gfortran \
 && rm -rf /var/lib/apt/lists/*

# Hack until spack fixes https://github.com/spack/spack/issues/27960
RUN sed -i 's/platform.release()/"5.4.100"/g' /opt/spack/var/spack/repos/builtin/packages/qt/package.py

# Install LLVM in builder
RUN apt-get -yqq update && apt-get -yqq install lsb-release wget software-properties-common
RUN wget https://apt.llvm.org/llvm.sh && \
    chmod +x llvm.sh && \
    ./llvm.sh 14 all

# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
RUN mkdir /opt/spack-environment \
&&  (echo "spack:" \
&&   echo "  compilers:" \
&&   echo "  - compiler:" \
&&   echo "      spec: clang@14.0.5" \
&&   echo "      paths:" \
&&   echo "        cc: /usr/bin/clang-14" \
&&   echo "        cxx: /usr/bin/clang++-14" \
&&   echo "        f77: /usr/bin/gfortran-11" \
&&   echo "        fc: /usr/bin/gfortran-11" \
&&   echo "      flags:" \
&&   echo "        cxxflags: -stdlib=libc++" \
&&   echo "      operating_system: ubuntu22.04" \
&&   echo "      target: x86_64" \
&&   echo "      modules: []" \
&&   echo "      environment: {}" \
&&   echo "      extra_rpaths:" \
&&   echo "      - /lib/gcc/x86_64-linux-gnu/11/libgfortran.so" \
&&   echo "  packages:" \
&&   echo "    all:" \
&&   echo "      target:" \
&&   echo "      - x86_64_v2" \
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
&&   echo "  - python+pythoncmd" \
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
RUN spack external find tar
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
FROM ubuntu:22.04

COPY --from=builder /opt/spack-environment /opt/spack-environment
COPY --from=builder /opt/software /opt/software
COPY --from=builder /opt/view /opt/view
COPY --from=builder /etc/profile.d/z10_spack_environment.sh /etc/profile.d/z10_spack_environment.sh

# Install LLVM in final container
RUN apt-get -yqq update && apt-get -yqq install lsb-release wget software-properties-common
RUN wget https://apt.llvm.org/llvm.sh && \
    chmod +x llvm.sh && \
    ./llvm.sh 14 all

RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash git cmake tar wget gfortran \
 && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["/bin/bash", "--rcfile", "/etc/profile", "-l"]

