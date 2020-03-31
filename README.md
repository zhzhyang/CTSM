# CTSM on JSC machines

Log on to JURECA or JUWELS. Follow the steps below to run a simple test case. 

  1. [Clone CTSM repository](#clone-ctsm-repository)
  2. [Configure environment](#configure-environment)
      * [Activate project](#activate-project)
      * [Create CTSM data directory](#create-ctsm-data-directory)
      * [Manage dependencies](#manage-dependencies)
      * [Create CIME config file](#create-cime-config-file)
  3. [Run test case](#run-test-case)
      * [Create case](#create-case)
      * [Setup](#setup)
      * [Build](#build)
      * [Submit](#submit)

All information in this README are based from these [sources](#references).

### Clone CTSM repository

Change to your preferred working directory. Execute these commands to download a working copy of CTSM.

```Shell
mkdir CTSM
git clone https://github.com/HPSCTerrSys/CTSM.git CTSM
cd CTSM
./manage_externals/checkout_externals
CTSMROOT=`pwd`
```

[checkout_externals][] automatically downloads the [CIME repository][], which contains the necessary scripts to run a case. 

### Configure environment

#### Activate project

```
jutil env activate -p <projectID>
```

Execute above command to initialize project-related environment variables. This simplifies configuration as it minimizes manual setting of environment variables during later steps. You may view the variables set by this command by running `jutil env dump`.

Note that you must specify a value for `projectID`. To determine which projects are available to you, run `jutil user projects -u $USER`.
    
#### Create CTSM data directory

Create a folder named `CESMDataRoot` under `$SCRATCH` file system as this folder would be huge. Run these commands:

```Shell
cd $SCRATCH/$USER
mkdir CESMDataRoot; cd CESMDataRoot
mkdir InputData CaseOutputs Archive BaselineTests
export CESMDATAROOT=`pwd`
```

This particular folder structure follows the definitions for JURECA and JUWELS in [config_machines.xml][].

#### Manage dependencies

Building CTSM requires at least:

* An MPI compiler and an MPI library
* netCDF Fortran and C libraries
* Perl 
* Python
* CMake

Dependency management is simplified through the [module system][]. CIME simplifies it further by allowing a default set of dependencies to be loaded on JURECA and JUWELS, which is already pre-defined in [config_machines.xml.][] Thus, you only have to specify which [Stage][] to use:

```Shell
export MODULESTAGE=2019a
```

All available Stages can be found by running `ls $STAGES`. However, Stages other than `2019a` and `2018b` are not guaranteed to work. 

#### Create CIME config file

Run

```Shell
mkdir ~/.cime
nano ~/.cime/config
```

and then paste the contents below to the text editor:

```ini
[main]
CIME_MODEL     = cesm
PROJECT        = 
CHARGE_ACCOUNT = 
MAIL_USER      = 
COMPILER       = intel
MPILIB         = psmpi
INPUT_DIR      = 
MAIL_TYPE      = 
```

Supply the missing fields:

* `PROJECT` Project ID specified during the [activate project](#activate-project) step.
* `CHARGE_ACCOUNT` Account name associated with the project ID. Run `printenv BUDGET_ACCOUNTS` to see which accounts are available.
* `INPUT_DIR` The resulting path when you run the command `echo $CESMDATAROOT/InputData`
* `MAIL_USER` Email address. Notifications on the job status are sent to this email.
* `MAIL_TYPE` Job notification type. Could be any of the following: **never**,**all**,**begin**,**fail**,**end**. Must be comma-separated if more one type is selected.

An example of a complete `~/.cime/config` file is shown below.

```ini
[main]
CIME_MODEL     = cesm
PROJECT        = cslts
CHARGE_ACCOUNT = slts
MAIL_USER      = my.name@fz-juelich.de
COMPILER       = intel
MPILIB         = psmpi
INPUT_DIR      = /p/scratch/cslts/user1/CESMDataRoot/InputData
MAIL_TYPE      = all
```

### Run test case 

[CIME] acts as an interface to CTSM. CIME is a set of Python scripts, config files, and other tools that manages creation, setup, build, and job submission of CTSM cases.

To begin, switch to CIME scripts directory.

```Shell
cd $CTSMROOT/cime/scripts
```

#### Create case

The quickest and simplest way to verify CTSM is to use an out-of-the-box dataset. Prepare a 1x1 Mexico test case with default Qian atmosphere data forcing by running:

```
./create_newcase --case testMXC --res 1x1_mexicocityMEX --compset I1PtClm50SpGs --run-unsupported
```

Then change to the newly created `testMXC` folder.

```Shell
cd testMXC
```

#### Setup

Namelists and PE layout are usually customized after creating a case. For simplicity we will just use the default values and then run

```
./case.setup
```

#### Build

Execute 

```
./case.build
```

This would take some time to finish. Build outputs are saved to `$CESMDATAROOT/CaseOutputs/testMXC/bld/`.

#### Submit

Before submitting the job you may review the job summary by executing `./preview_run`. Once you're ready, run

```
./case.submit
```

This script will automatically download the required inputs for Mexico test case if they are missing. Downloads are saved to `$CESMDATAROOT/InputData` and might take more than 30 minutes to complete. Once the downloads are complete, the job is submitted to the queue. 

You will receive job notifications via email if `MAIL_TYPE` is set to `all` on your [CIME config file](#create-cime-config-file). You may also run these commands to check the job status:

* `squeue -u $USER --start` Check whether the job has started or not.
* `tail CaseStatus` View recent CIME activity logs.

The job outputs are saved to `$CESMDATAROOT/CaseOutputs/testMXC/run/`.

### References

* [CIME Basic Usage](https://esmci.github.io/cime/versions/master/html/users_guide/index.html)
* [CLM5.0_on_JSC_Machines](https://icg4geo.icg.kfa-juelich.de/SoftwareTools/CLM5.0_on_JSC_Machines)
* [CLM5.0 User's Guide](https://escomp.github.io/ctsm-docs/doc/build/html/users_guide/index.html)
* [ESCOMP/CTSM Wiki pages](https://github.com/ESCOMP/CTSM/wiki)
* [JURECA](https://apps.fz-juelich.de/jsc/hps/jureca/index.html#) and [JUWELS](https://apps.fz-juelich.de/jsc/hps/juwels/index.html#) User Guides 

[checkout_externals]: manage_externals/checkout_externals/README.md
[CIME repository]: https://github.com/HPSCTerrSys/cime
[config_machines.xml]: https://github.com/HPSCTerrSys/cime/blob/master/config/cesm/machines/config_machines.xml#L1546
[module system]: https://www.fz-juelich.de/ias/jsc/EN/Expertise/Supercomputers/JURECA/Software/Software_node.html
[config_machines.xml.]: https://github.com/HPSCTerrSys/cime/blob/master/config/cesm/machines/config_machines.xml#L1563
[Stage]: https://www.fz-juelich.de/ias/jsc/EN/Expertise/Supercomputers/JURECA/Software/Software.html?nn=1803760#Stages
[CIME]: https://esmci.github.io/cime/versions/master/html/what_cime/index.html#what-is-cime
