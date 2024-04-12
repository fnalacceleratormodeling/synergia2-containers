# Build stage with Spack pre-installed and ready to be used
FROM spack/ubuntu-jammy:latest as builder

# Install OS packages needed to build the software
RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash perl wget tar gfortran curl \
 && rm -rf /var/lib/apt/lists/*

# Hack until spack fixes https://github.com/spack/spack/issues/27960
RUN sed -i 's/platform.release()/"5.4.100"/g' /opt/spack/var/spack/repos/builtin/packages/qt/package.py

# Install LLVM in builder
RUN apt-get -yqq update && apt-get -yqq install lsb-release software-properties-common
RUN wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | apt-key add -
RUN add-apt-repository "deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy-16 main"
RUN apt-get -yqq update
RUN apt-get -yqq install clang-16 libc++-16-dev libc++abi-16-dev libomp-16-dev

# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
RUN mkdir /opt/spack-environment \
&&  (echo spack: \
&&   echo '  compilers:' \
&&   echo '  - compiler:' \
&&   echo '      spec: clang@16.0.3' \
&&   echo '      paths:' \
&&   echo '        cc: /usr/bin/clang-16' \
&&   echo '        cxx: /usr/bin/clang++-16' \
&&   echo '        f77: /usr/bin/gfortran' \
&&   echo '        fc: /usr/bin/gfortran' \
&&   echo '      flags:' \
&&   echo '        cxxflags: -stdlib=libc++' \
&&   echo '      operating_system: ubuntu22.04' \
&&   echo '      modules: []' \
&&   echo '      environment: {}' \
&&   echo '      extra_rpaths: []' \
&&   echo '  packages:' \
&&   echo '    all:' \
&&   echo '      target:' \
&&   echo '      - x86_64_v2' \
&&   echo '      compiler:' \
&&   echo '      - clang' \
&&   echo '      providers:' \
&&   echo '        # Cannot use intel-oneapi-mpi with clang' \
&&   echo '        # for some reason, so use mpich for now.' \
&&   echo '        mpi:' \
&&   echo '        - mpich' \
&&   echo '        blas:' \
&&   echo '        - openblas' \
&&   echo '        lapack:' \
&&   echo '        - openblas' \
&&   echo '  specs:' \
&&   echo '  - mpich+fortran device=ch4 netmod=ofi' \
&&   echo '  - openblas~fortran' \
&&   echo '  - openpmd-api+python' \
&&   echo '  - petsc~fortran+debug' \
&&   echo '  - hdf5+hl+mpi~fortran' \
&&   echo '  - fftw+mpi+openmp' \
&&   echo '  - cmake' \
&&   echo '  - gsl' \
&&   echo '  - python+pythoncmd' \
&&   echo '  - py-pyparsing' \
&&   echo '  - py-numpy' \
&&   echo '  - py-mpi4py' \
&&   echo '  - py-h5py+mpi' \
&&   echo '  - py-pytest' \
&&   echo '  - ninja' \
&&   echo '  concretizer:' \
&&   echo '    duplicates:' \
&&   echo '      strategy: full' \
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
FROM ubuntu:22.04

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
 && apt-get -yqq install build-essential bash git tar gfortran curl wget \
 && rm -rf /var/lib/apt/lists/*

# Install LLVM in the final image
RUN apt-get -yqq update && apt-get -yqq install lsb-release software-properties-common
RUN wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | apt-key add -
RUN add-apt-repository "deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy-16 main"
RUN apt-get -yqq update
RUN apt-get -yqq install clang-16 libc++-16-dev libc++abi-16-dev libomp-16-dev

ENTRYPOINT [ "/entrypoint.sh" ]
CMD [ "/bin/bash" ]

