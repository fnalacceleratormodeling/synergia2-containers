# Build stage with Spack pre-installed and ready to be used
FROM spack/ubuntu-noble:develop as builder

# Install OS packages needed to build the software
RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash perl cmake gfortran wget tar curl \
 && rm -rf /var/lib/apt/lists/*

# Hack until spack fixes https://github.com/spack/spack/issues/27960
RUN sed -i 's/platform.release()/"5.4.100"/g' /opt/spack/var/spack/repos/builtin/packages/qt/package.py

# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
RUN mkdir -p /opt/spack-environment && \
set -o noclobber \
&&  (echo spack: \
&&   echo '  packages:' \
&&   echo '    all:' \
&&   echo '      target:' \
&&   echo '      - x86_64_v2' \
&&   echo '      providers:' \
&&   echo '        mpi:' \
&&   echo '        - mpich' \
&&   echo '        blas:' \
&&   echo '        - openblas' \
&&   echo '        lapack:' \
&&   echo '        - openblas' \
&&   echo '  specs:' \
&&   echo '  - hdf5+hl+mpi~fortran' \
&&   echo '  - mpich+fortran device=ch4 netmod=ofi' \
&&   echo '  - petsc~fortran+debug' \
&&   echo '  - openpmd-api+python' \
&&   echo '  - fftw+mpi+openmp' \
&&   echo '  - gsl' \
&&   echo '  - mold' \
&&   echo '  - python+pythoncmd' \
&&   echo '  - py-pyparsing' \
&&   echo '  - py-numpy' \
&&   echo '  - py-pybind11' \
&&   echo '  - py-mpi4py' \
&&   echo '  - py-h5py+mpi' \
&&   echo '  - py-pytest' \
&&   echo '  - ninja' \
&&   echo '  concretizer:' \
&&   echo '    unify: true' \
&&   echo '  view: /opt/views/view' \
&&   echo '  config:' \
&&   echo '    install_tree: /opt/software') > /opt/spack-environment/spack.yaml

# Find externals
RUN spack external find tar
RUN spack external find perl
RUN spack external find ncurses
RUN spack external find gettext

# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment && spack env activate . && spack install --fail-fast && spack gc -y

# Strip all the binaries
RUN find -L /opt/views/view/* -type f -exec readlink -f '{}' \; | \
    xargs file -i | \
    grep 'charset=binary' | \
    grep 'x-executable\|x-archive\|x-sharedlib' | \
    awk -F: '{print $1}' | xargs strip

# Modifications to the environment that are necessary to run
RUN cd /opt/spack-environment && \
    spack env activate --sh -d . > activate.sh


# Bare OS image to run the installed executables
FROM ubuntu:24.04

COPY --from=builder /opt/spack-environment /opt/spack-environment
COPY --from=builder /opt/software /opt/software

# paths.view is a symlink, so copy the parent to avoid dereferencing and duplicating it
COPY --from=builder /opt/views /opt/views

RUN { \
      echo '#!/bin/sh' \
      && echo '.' /opt/spack-environment/activate.sh \
      && echo 'exec "$@"'; \
    } > /entrypoint.sh \
&& chmod a+x /entrypoint.sh \
&& ln -s /opt/views/view /opt/view


RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash git cmake wget gfortran tar curl \
 && rm -rf /var/lib/apt/lists/*
ENTRYPOINT [ "/entrypoint.sh" ]
CMD [ "/bin/bash" ]

