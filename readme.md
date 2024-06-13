Spack Course

# UENV, Spack and Best Practices


## Uenv are reproducible, non-composable images that provide complete software stacks

Software stacks can be programming environments, apps and tools

Spack is used to build the software
- only expose the features required for "CSCS-style" software stacks

## Uenv are from a recipe using the stackinator tool

Stackinator is like CMake: convert a recipe into a directory tree ready to hit "make"

Stackinator takes the following inputs:
- a recipe:
    - for example: [the gromacs recipe](https://github.com/eth-cscs/alps-uenv/blob/main/recipes/gromacs/2024/gh200-mpi/compilers.yaml)
- a cluster configuration, e.g.
    - for example: [the todi definition](https://github.com/eth-cscs/alps-cluster-config/tree/master/todi)

It takes these inputs and:

- create build path
- downloads Spack into build path
- create spack config
- generate the compiler and environment build paths
- generate the Makefiles build the software
- create the "store" path that will be mounted at `/user-environment` when building, and will be the final squashfs image.

## uenv are built using make, which builds the environments in the correct order

The following stages are performed when building an image:

1. bootstrap
2. pre install hook
3. configure build cache
4. build compilers
    - bootstrap
    - gcc
    - (optional) nvhpc
5. generate-config
    - this stage generates the full `config` path
6. build environments
    - build each environment in parallel
7. generate modules
8. post install hook
9. generate squashfs image

Why do we use Makefiles?
- Because Harmen likes Makefiles
- Because we are not building the whole stack as a single Spack environment
    - Build multiple environments that depend on one another
    - And steps like updating build caches, creating modules, etc
- Because it allows us to use a make server to concurrently run parallel package builds.

Some steps of interest:

2. **pre-install hook:**
    - runs an arbitrary script
        - you can use it to make modifications to the uenv
    - use case: copy source code tar balls for restricted access codes (e.g. NAMD and VASP)
        - https://github.com/eth-cscs/alps-uenv/blob/main/recipes/namd/3.0b6/gh200/pre-install
        - use the JFrog artifactory build exactly for this purpose
    - use case: patch Spack (e.g. the arch.json file)
3. configure build caches:
    - use them!
    - if you use the CI/CD pipeline, you get the benefit of a shared cache
    - unfortunately buildcaches are not always compatible between Spack versions.
4. compilers:
    - builds a bootstrap gcc compiler (yes, we need it)
    - then builds a gcc compiler with that bootstrapped compiler
    - optionally "builds" nvhpc
5. environments
    - more on these later
7. generate modules
    - optional: turn them off if you don't intend to provide them
        - only provide them if you intend to cultivate and water them, and support user requests.
8. **post install hook**
    - like pre-install hooks, but run just after everything has been installed and configured.
    - use case: provide an custom activation script:
        - https://github.com/eth-cscs/alps-uenv/blob/main/recipes/linaro-forge/23.1.2/post-install
    - use case: add some extra meta-data specific to your workflow:
        - https://github.com/eth-cscs/alps-uenv/blob/main/recipes/wcp/icon/v1/a100/post-install

## UENV deployed on Alps (and AlpsM/AlpsE) are managed in a GitHub repository

https://github.com/eth-cscs/alps-uenv

The repository contains:
- recipes for all of the uenv
- CI/CD configuration (GitLab yaml and python scripts)
- scripts that run each stage
    - building the uenv
    - running reframe tests

### On Alps uenv are labelled: system/uarch/name/version:tag

Documented on Confluence: https://confluence.cscs.ch/display/VCUE/UENV

### The description of clusters and mapping of recipes to clusters is in config.yaml

[`alps-uenv/config.yaml`](https://github.com/eth-cscs/alps-uenv/blob/07995366d34f7845163fe418938258dfd4b41a36/config.yaml)

When you add a new recipe, this file will need to be updated before building 

```yaml
  cp2k:
    "2023":
      recipes:
        # zen2 and zen3 targets use the same recipe
        zen2: 2023/mc
        zen3: 2023/mc
        a100: 2023/a100
      deploy:
        eiger: [zen2]
      develop: False
    "2024.1":
      recipes:
        gh200: 2024.1/gh200
      deploy:
        santis: [gh200]
        todi: [gh200]
      develop: False
```

### Uenv and their meta data are stored in a JFrog registry.

The registry are OCI container registries.
- there are two namespaces:
    1. `build`: the CI/CD pipeline pushes images here
        - the unique id of the pipeline is used as the image tag `system/uarch/name/version:buildid`
        - `uenv image find --build` will show all images built for the vCluster it is called on (the `CLUSTER_NAME`) env. var is used.
    2. `deploy`: images are moved here to make them available to users
        - a more meaningful tag like `v3` is applied
        - `uenv image deploy --tags=v3 `
        - `uenv image find` will list all images available on a vCluster

The CI/CD pipeline and `uenv` CLI tool use oras to interact with the registry:
- pipelines push and pull
- pipelines attach meta data, and query it
    - this allows us to query meta data without downloading the whole image
- uenv copies an image to deploy it
- uenv pulls an image to download it locally

### To access non-public namespaces (e.g. `build`), you have to configure JFrog tokens

To get access to deploy (and VASP etc, if you have access):
1. log into the CSCS VPN
2. then go to https://jfrog.svc.cscs.ch
3. click "log in" in the top rhs corner
4. click "set me up" in the drop down menu in top RHS corner
5. generate a Docker token and copy it
6. log into the target system and use the copy of oras bundled with uenv

```
cat TOKEN | /opt/cscs/uenv/libexec/uenv-oras login jfrog.svc.cscs.ch --username bcumming --password-stdin
```

## There are best-practices for creating a uenv recipe

Follow the guide: https://eth-cscs.github.io/alps-uenv/pkg-application-tutorial/

## Write documentation for uenv

The documentation is on the eth-cscs github.io:

https://eth-cscs.github.io/alps-uenv/

Which is generated from the markdown using (Material for MKDocs](https://squidfunk.github.io/mkdocs-material/)
- :shushing_face: ... much easier to write and structure than confluence.
- use the features

### adding documentation in the recipe repo keeps it up to date

Use the KB article to give a quick overview of information that doesn't go out of date:

Document the versions, environment variables, etc in the docs.
- it is expected that documentation will be updated

### documentation can be updated independently of recipe changes

Discover a bug and the fix is to set an environment variable: push the information to the documentation page

### If you expect users to use the uenv a specific way - document it

Example of documenting two different ways to interact with a uenv
https://eth-cscs.github.io/alps-uenv/uenv-linaro-forge/#usage-notes

Example of documenting how to build:
https://eth-cscs.github.io/alps-uenv/uenv-qe/#using-modules

### provide tips, best practices and workarounds

Environment variables, slurm parameters and known issues can be documented clearly

https://eth-cscs.github.io/alps-uenv/uenv-gromacs/#how-to-run

## pro tips

### File system views are great, when they work.

File system views are great! a single command configures the environment:

```bash
uenv view develop
```

**take care to curate the environment**

By default, when you create a file system view for an environment, Spack will link every package in the environment into the view
- root specs and all of their dependencies.

This causes myriad problems when a view sets `LD_LIBRARY_PATH`:
- some software on the system, e.g. `git`, `more` and ... drumroll... libfabric will use `LD_LIBRARY_PATH` to find dependencies like:

```
        /usr/bin/ld: /usr/lib64/libssh.so.4: undefined reference to `EVP_KDF_CTX_new_id@OPENSSL_1_1_1d'
        /usr/bin/ld: /usr/lib64/libssh.so.4: undefined reference to `EVP_KDF_ctrl@OPENSSL_1_1_1d'
        /usr/bin/ld: /usr/lib64/libssh.so.4: undefined reference to `EVP_KDF_CTX_free@OPENSSL_1_1_1d'
        /usr/bin/ld: /usr/lib64/libssh.so.4: undefined reference to `EVP_KDF_derive@OPENSSL_1_1_1d'
```

- `libssh.so` expects a symbol in `libcrypto.so` - but gets the version built in the uenv which does not have the same symbols as `/user/lib64/libcrypto.so`

TODO: we will start using the security libraries provided by the system by default.

#### Use `link:roots` and include _every_ spec that you need

Only packages in the list of specs will be linked:
```yaml
  views:
    develop:
      link: roots
```

You will need to add every single spec that is required by your view.

#### Only add specs that you need

To avoid collisions.

> :warning: **Ben's first law of providing software**
>
> If users find it, they will use it.

TODO: write some reframe tests that try to trigger known issues
- e.g. `more` vs. `ncurses`

### It is a pain to specify which compiler to use when using more than one toolchain

Frequently both the nvhpc and gcc toolchains have to be used to build an environment that requires the nvhpc Fortran compiler for OpenACC
- e.g. ICON, Quantumespresso, and VASP

In the current version of stackinator we are at the mercy of the concretizer:
```yaml
  compiler:
      - toolchain: gcc
        spec: gcc
      - toolchain: llvm
        spec: nvhpc
  mpi:
      spec: cray-mpich@8.1.29%nvhpc
      gpu: cuda
  specs:
  - boost%gcc ~mpi
  - python@3.10%gcc
  - cmake%gcc
  - cuda@12.3%gcc
  - hdf5%gcc
  - hwloc%gcc
  - netcdf-c%gcc
  - netcdf-cxx4%gcc
  - netcdf-fortran%nvhpc
  - numactl%gcc
  - osu-micro-benchmarks@5.9%nvhpc
  # everything needed for nccl on SS11
  - aws-ofi-nccl@master%gcc
  - nccl%gcc
  - nccl-tests%gcc
  # The following are required to stop spack from using nvhpc to build
  # basic dependencies, some of which don't compile with nvc etc.
  # Explicitly excluded as modules.
  - autoconf%gcc
  - automake%gcc
  - ca-certificates-mozilla%gcc
  - diffutils%gcc
  - gnuconfig%gcc
  - libiconv%gcc
  - libxcrypt%gcc
  - libxml2%gcc
  - m4%gcc
  - ncurses%gcc
  - openssl%gcc
  - perl%gcc
  - xz%gcc
  - zlib%gcc
  - zstd%gcc
  - c-blosc%gcc
  - libaec%gcc
  - jasper%gcc
  - patchelf%gcc
  - gmake%gcc
```

There is a PR open in Stackinator that will enforce that the first compiler will be the default, so that it becomes:

```yaml
  compiler:
      - toolchain: gcc
        spec: gcc
      - toolchain: llvm
        spec: nvhpc
  mpi:
      spec: cray-mpich@8.1.29%nvhpc
      gpu: cuda
  specs:
  - boost ~mpi
  - python@3.10
  - cmake
  - cuda@12.3
  - hdf5
  - hwloc
  - netcdf-c
  - netcdf-cxx4
  - netcdf-fortran%nvhpc
  - numactl
  - osu-micro-benchmarks@5.9%nvhpc
  # everything needed for nccl on SS11
  - aws-ofi-nccl@master
  - nccl
  - nccl-tests
```

* by default you want to build everything with `gcc` insofar as it is possible
    - believe me, you really do
* so set

*NOTE*: Always annotate the `cray-mpich` spec with a compiler in multi 
* if you don't target Fortran, you can probably get away with cray-mpich%gcc
    - C/C++ have an ABI
* Fortran modules generated by different toolchains are incompatible
    - the `.mod` files provided must match the 

TODOs
- walk through a PR
- testing
