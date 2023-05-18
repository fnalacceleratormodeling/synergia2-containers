FROM nvidia/cuda:11.8.0-devel-ubuntu22.04 as bootstrap

ENV SPACK_ROOT=/opt/spack \
    CURRENTLY_BUILDING_DOCKER_IMAGE=1 \
    container=docker

ENV DEBIAN_FRONTEND=noninteractive   \
    LANGUAGE=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8

RUN apt-get -yqq update \
 && apt-get -yqq upgrade \
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
        make \
        python3 \
        python3-pip \
        python3-setuptools \
        unzip \
 && locale-gen en_US.UTF-8 \
 && pip3 install boto3 \
 && rm -rf /var/lib/apt/lists/*

RUN mkdir $SPACK_ROOT && cd $SPACK_ROOT && \
    git clone https://github.com/spack/spack.git . && git fetch origin develop:container_branch && git checkout container_branch  && \
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

# Creates the package cache
RUN spack bootstrap now && spack spec hdf5+mpi

ENTRYPOINT ["/bin/bash", "/opt/spack/share/spack/docker/entrypoint.bash"]
CMD ["interactive-shell"]

# Build stage with Spack pre-installed and ready to be used
FROM bootstrap as builder

# Install OS packages needed to build the software
RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash perl cmake gfortran wget tar \
 && rm -rf /var/lib/apt/lists/*

# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
RUN mkdir /opt/spack-environment \
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
&&   echo '  - hdf5@1.12.2+hl+mpi~fortran' \
&&   echo '  - mpich@develop~fortran device=ch4 netmod=ofi' \
&&   echo '  - petsc@main~fortran+debug' \
&&   echo '  - fftw+mpi+openmp' \
&&   echo '  - gsl' \
&&   echo '  - python+pythoncmd' \
&&   echo '  - py-pyparsing' \
&&   echo '  - py-numpy' \
&&   echo '  - py-mpi4py' \
&&   echo '  - py-h5py+mpi' \
&&   echo '  - py-pytest' \
&&   echo '  - ninja' \
&&   echo '  concretizer:' \
&&   echo '    unify: true' \
&&   echo '  view: /opt/view' \
&&   echo '  config:' \
&&   echo '    install_tree: /opt/software') > /opt/spack-environment/spack.yaml

# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment && spack env activate . && spack install --fail-fast && spack gc -y

# Strip all the binaries
RUN find -L /opt/view/* -type f -exec readlink -f '{}' \; | \
    xargs file -i | \
    grep 'charset=binary' | \
    grep 'x-executable\|x-archive\|x-sharedlib' | \
    awk -F: '{print $1}' | xargs strip

# Modifications to the environment that are necessary to run
RUN cd /opt/spack-environment && \
    spack env activate --sh -d . >> /etc/profile.d/z10_spack_environment.sh


# Bare OS image to run the installed executables
FROM nvidia/cuda:11.8.0-base-ubuntu22.04

COPY --from=builder /opt/spack-environment /opt/spack-environment
COPY --from=builder /opt/software /opt/software
COPY --from=builder /opt/._view /opt/._view
COPY --from=builder /opt/view /opt/view
COPY --from=builder /etc/profile.d/z10_spack_environment.sh /etc/profile.d/z10_spack_environment.sh


RUN apt-get -yqq update && apt-get -yqq upgrade \
 && apt-get -yqq install build-essential bash git cmake wget gfortran tar \
 && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["/bin/bash", "--rcfile", "/etc/profile", "-l", "-c", "$*", "--" ]
CMD [ "/bin/bash" ]

