# CESM/CIME configuration for SISC HPC

CESM/CIME is used directly from the source code. Researches will clone some release or development version of CESM, grab additional sources for CIME in external repos and use that ensemble to create, build and run simulations (*aka* cases). Therefore, it is not possible to seamlesly adapt the software to our clusters. Users have to manually add the configuration settings and any other modification needed to use CESM in our clusters.

## User Documentation

User documentation can be found at https://hpc.vub.be/documentation/software.html#how-can-i-use-cesm

## Machine files

XML files configuring the system environment
* config_machines.xml
    * regex to identify the system machine
    * file structure: location of input data, case folders and case output
    * configuration of the module system
    * list of modules to be loaded by CESM
* config_compiler.xml
    * compiler settings
    * library paths
    * filesystem settings
* config_batch.xml:
    * description of the queue
    * syntax of the request for resources

### Hydra

* Using `--machine hydra` is optional as long the user is in a compute node or a login node in Hydra. CESM uses `NODENAME_REGEX` in `config_machines.xml` to identify the host machine.
* The only module needed is `CESM-deps`. It has to be loaded at all times, from cloning of the sources to case submission.
* There is a single configuration for the compiler that is agnostic of the CPU architecture.
* CESM is not capable of detecting and automatically adding the required libraries to its build process. The current specification of `SLIBS` contains just what we found so far to be required.
* By design, CESM sets a specific queue with `-q queue_name`, otherwise it fails to even create the case. In Hydra we have the queue `submission` for this purpose.
* Limit maximum number of nodes to 10 to ensure that the scale of CESM jobs stay within reasonable limits for Hydra.

### Breniac

* There are two clusters defined in Breniac, `breniac` and `breniac-skl`
    * `breniac` is the default one, uses AVX2 and the resulting binaries will works in all nodes in Breniac. It will be picked by default in the login nodes or if `--machine breniac` is specified
    * `breniac-skl` is optimized for the Skylake nodes in Breniac and uses AVX512. It will only be picked if the host system is a Skylake node or if `--machine breniac-skl` is specified
* The only module needed is `CESM-deps`. It has to be loaded at all times, from cloning of the sources to case submission.
* By design, CESM sets a specific queue with `-q queue_name`, otherwise it fails to even create the case. In Breniac we can use the queue `qdef` as it will derive the job to `q1h`, `q24h` or `q72h` depending on the walltime requested.

## File structure

All paths are below `$VSC_SCRATCH`. These folders contain everything from the executables, to input and output data. Moreover, CESM can generate rather big cases reaching several hundreds of GB.

```
$VSC_SCRATCH
 └─── cesm/inputdata
 └─── cime
       └─── cases
       └─── output
       └─── source_code (optional)
```

## Job scripts

The common workflow with CESM consists in creating and building the case interactively in the login node of the cluster. Then the case is submitted to the queue and CESM will automatically set the resources, walltime and queue of each job. This can cause problems in a heterogeneus environment as not all nodes might provide the same hardware features as the login nodes.

The example job script in `scripts/case.job` solves this problem by executing all steps in the compute nodes of the cluster. In this way, the compilation can be optimized to the host machine, simplifying the configuration, and the user does not have to worry about where the case is build and where it is executed.

## Easyconfigs

* CESM-deps loads all dependencies to build and run CESM cases
    * `CESM-deps-2-intel-2018a.eb` is used in Breniac
    * `CESM-deps-2-foss-2019a.eb` is used in Hydra

* CESM-tools loads software commonly used to analyse the results of the simulations
    * `CESM-tools-2-foss-2019a.eb` is available in Hydra

Our easyconfigs of CESM-deps are based on those available in [EasyBuild](https://github.com/easybuilders/easybuild-easyconfigs/tree/master/easybuild/easyconfigs/c/CESM-deps). However, the CESM-deps module in Hydra also contains the configuration files and scripts from this repository, which are located in the installation directory (`$EBROOTCESMMINDEPS`). Hence, our users have direct access to these files once `CESM-deps-2-foss-2019a.eb` is loaded. The usage instructions of our CESM-deps modules also provide a minimum set of instructions to create cases in Hydra with this configuration files.

## CPRNC

There is small tool called [cprnc](https://github.com/ESMCI/cime/tree/master/tools/cprnc) that needs to be compiled and placed in `CCSM_CPRNC`, a path defined in `config_machines.xml`.

These are the steps to compile this tool

1. Load the CESM-deps/2-foss-2019a module
2. Prepare the source code tree of CESM as usual (as explained in our documentation)
3. Change to source folder: `cd $VSC_SCRACTH/cime/cesm-x.y.z/cime/tools/cprnc/`
4. Configure: `CIMEROOT=../.. ../configure --macros-format=Makefile`
5. Build:

```
$ CIMEROOT=../.. source ./.env_mach_specific.sh &&
  make FFLAGS="$FFLAGS -I${EBROOTNETCDFMINFORTRAN}/include"
  LDFLAGS="$LDFLAGS -I${EBROOTNETCDFMINFORTRAN}/lib"
```

The binary installed in `CCSM_CPRNC` will be used in all nodes in the cluster. Therefore it has to be build with the minimum CPU optimizations
* the binary for Hydra was built in an IvyBridge CPU (available upon request)
* the binary for Breniac was built in a Broadwell CPU (available upon request)

## Validation of the CESM installation

We have carried out a scientific validation of the CESM installation in Hydra and Breniac to verify the reliability of the results obtained with it. The procedure is described in http://www.cesm.ucar.edu/models/cesm2/python-tools/.

1. Create case with the `ensemble.py` tool provided in the CESM source code
2. Execute that case in the cluster (setup + build + submit)
3. Use the web tool to compare the resulting `.nc` files with the reference data

An installation of CESM v2.1.1 in Breniac passed all tests
* UF-CAM-ECT test: validated by Steven
* POP-ECT: validated by Alex

An installation of CESM v2.1.1 in Hydra passed all tests
* UF-CAM-ECT test: validated by Alex
* POP-ECT: validated by Alex

Results of the validations can be found in the `validation` folder.
