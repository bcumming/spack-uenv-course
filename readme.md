Spack Course

# UENV, Spack and Best Practices


## Uenv are reproducible, non-composable images that provide complete software stacks

- software stacks can be programming environments, apps and tools
- Spack is used to build the software
    - only expose the features required for CSCS-style software stacks

## Uenv are from a recipe using the stackinator tool

Stackinator is like CMake: convert a recipe into a directory tree ready to hit "make"

[An example of the gromacs recipe](https://github.com/eth-cscs/alps-uenv/blob/main/recipes/gromacs/2024/gh200-mpi/compilers.yaml)

* `configure.yaml`
    - set the version of Spack - prefer fixed versions
* `compilers.yaml`
    - choose versions of compilers that other packages for the same target uarch use

The 
- create build path
- download Spack into build path
- create config
- generate the compiler and environment build paths
- generate the Makefiles build the software

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

2. pre-install hook:
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
    - builds a bootstrap gcc compiler
    - then builds a gcc compiler with that bootstrapped compiler
    - optionally "builds" nvhpc
5. environments
7. generate modules
    - optional: turn them off if you don't intend to provide them
        - only provide them if you intend to cultivate and water them, and support user requests.
8. post install hook
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
    2. 

TODO: JFrog repository address

The CI/CD pipeline and `uenv` CLI tool use oras to

- `oras pull`

### To access non-public namespaces (e.g. `build`), you have to configure JFrog tokens

Go to 

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

File system views are great: a single command configures the environment:

```bash
uenv view develop
```

*you need to take care to curate the environment*

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
