Spack Course

# UENV, Spack and Best Practices


## Uenv are reproducible, non-composable images that provide complete software stacks

- software stacks can be programming environments, apps and tools
- Spack is used to build the software
    - only expose the features required for CSCS-style software stacks

## Uenv are built in stages from a recipe

Stackinator is like CMake: convert a recipe into a directory tree ready to hit "make"
- create build path
- download Spack into build path
- create config
- generate the compiler and environment build paths
- generate the Makefiles that coordinate 

The following stages are performed when building an image:

1 bootstrap
2 pre install hook
3 configure build cache
4 build compilers
    - bootstrap
    - gcc
    - (optional) nvhpc
5 generate-config
    - this stage generates the full `config` path
6 build environments
    - build each environment in parallel
7 generate modules
8 post install hook
9 generate squashfs image

Why do we use Makefiles?
- Because Harmen likes Makefiles
    - if Harmen accepts credit for all the good things he gave us, he has to take responsibility
- Because building a uenv requires a sequence of 
Of interest to you are

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
    - 
7. generate modules
    - optional: turn them off if you don't intend to provide them
        - only provide them if you intend to cultivate and water them, and support user requests.
8. post install hook
    - like pre-install hooks, but run just after everything has been installed and configured.
    - use case: provide an custom activation script:
        - https://github.com/eth-cscs/alps-uenv/blob/main/recipes/linaro-forge/23.1.2/post-install
    - use case: add some extra meta-data specific to your workflow:
        - https://github.com/eth-cscs/alps-uenv/blob/main/recipes/wcp/icon/v1/a100/post-install

## How UENV are managed

### system/uarch/name/version/tag


Which versions should I support? Only the ones that you have to deploy on.
- don't lose sleep over Mi250x

### The description of clusters and mapping of recipes to clusters is in config.yaml

### Uenv and their meta data are stored in JFrog DockerHub registries.

## Guide To Creating UENV

### Know your target audience: provide what they need

only need the tool - just provide the tool.


## Choosing a compiler

For bootstrap - pick the same one that other uenv use on the same uarch

## Configuring environments


## Documentation

### adding documentation in the recipe repo keeps it up to date

### documentation can be updated independently of recipe changes

### If you expect users to use the uenv a specific way - document it

Example of documenting two different ways to interact with a uenv
https://eth-cscs.github.io/alps-uenv/uenv-linaro-forge/#usage-notes

Example of documenting how to build:
https://eth-cscs.github.io/alps-uenv/uenv-qe/#using-modules

### provide tips, tricks and workarounds

Environment variables, slurm parameters and known issues can be documented clearly

https://eth-cscs.github.io/alps-uenv/uenv-gromacs/#how-to-run
