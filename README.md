# synergia2-containers

This repository contains docker recipes for building containers that contain all dependencies for synergia2. These recipes are generated using [spack environments](https://spack.readthedocs.io/en/latest/environments.html) via [`spack containerize`](https://spack.readthedocs.io/en/latest/containers.html), with some minor modifications. GithubActions is used to build these containers for x86-haswell ISA and these containers can be pulled from the github container registry. For instructions on how to pull a particular image, visit the page associated with it [here](https://github.com/orgs/fnalacceleratormodeling/packages?repo_name=synergia2-containers).


These containers are used as test environments for testing synergia2 via GithubActions.
