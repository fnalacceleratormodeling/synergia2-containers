# Build stage with Spack pre-installed and ready to be used
FROM spack/ubuntu-jammy:latest as builder

# Install OS packages needed to build the software
RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash perl cmake gfortran wget tar \
 && rm -rf /var/lib/apt/lists/*

# Hack until spack fixes https://github.com/spack/spack/issues/27960
RUN sed -i 's/platform.release()/"5.4.100"/g' /opt/spack/var/spack/repos/builtin/packages/qt/package.py

# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
RUN mkdir /opt/spack-environment \
&&  (echo "spack:" \
&&   echo "  packages:" \
&&   echo "    all:" \
&&   echo "      target:" \
&&   echo "      - x86_64_v2" \
&&   echo "      providers:" \
&&   echo "        mpi:" \
&&   echo "        - mpich" \
&&   echo "        blas:" \
&&   echo "        - openblas" \
&&   echo "        lapack:" \
&&   echo "        - openblas" \
&&   echo "  specs:" \
&&   echo "  - hdf5@1.12.1+hl+mpi~fortran" \
&&   echo "  - mpich@develop~fortran device=ch4 netmod=ofi pmi=pmix" \
&&   echo "  - petsc@main~fortran+debug" \
&&   echo "  - fftw+mpi+openmp" \
&&   echo "  - gsl" \
&&   echo "  - python+pythoncmd" \
&&   echo "  - py-pyparsing" \
&&   echo "  - py-numpy" \
&&   echo "  - py-mpi4py" \
&&   echo "  - py-h5py+mpi" \
&&   echo "  - py-pytest" \
&&   echo "  - ninja" \
&&   echo "  concretizer:" \
&&   echo "    unify: true" \
&&   echo "  view: /opt/view" \
&&   echo "  config:" \
&&   echo "    install_tree: /opt/software") > /opt/spack-environment/spack.yaml

# Find externals
RUN spack external find tar
RUN spack external find perl
RUN spack external find ncurses
RUN spack external find gettext

# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment && spack env activate . && spack install --fail-fast && spack gc -y

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
COPY --from=builder /opt/._view /opt/._view
COPY --from=builder /opt/view /opt/view
COPY --from=builder /etc/profile.d/z10_spack_environment.sh /etc/profile.d/z10_spack_environment.sh

# Noninteractive install!
ENV DEBIAN_FRONTEND=noninteractive   \
    LANGUAGE=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8 \
    TZ=Etc/UTC

RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash git cmake wget gfortran tar \
 && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["/bin/bash", "--rcfile", "/etc/profile", "-l", "-c", "$*", "--" ]
CMD [ "/bin/bash" ]

