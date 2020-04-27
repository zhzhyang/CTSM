# CTSM on JSC machines

Log on to JURECA or JUWELS. Follow the steps below to run a simple test case. 

  1. [Clone CTSM repository](#clone-ctsm-repository)
  2. [Configure environment](#configure-environment)
      * [Activate compute project](#activate-compute-project)
      * [Create CTSM data directory](#create-ctsm-data-directory)
      * [Manage dependencies](#manage-dependencies)
      * [Create CIME config file](#create-cime-config-file)
  3. [Run single point case](#run-single-point-case)
      * [Create case](#create-case-1)
      * [Setup](#setup-1)
      * [Build](#build-1)
      * [Submit](#submit-1)
  4. [Run regional case (Optional)](#run-a-regional-case)
      * [Create case](#create-case-2)
      * [Setup](#setup-2)
      * [Build](#build-2)
      * [Submit](#submit-2)

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

#### Activate compute project

```
jutil env activate -p <projectID>
```

Execute above command to initialize project-related environment variables. This simplifies configuration as it minimizes manual setting of environment variables during later steps. You may view the variables set by this command by running `jutil env dump`.

Note that you must specify a value for `projectID`. To determine which projects are available to you, run `jutil user projects -u $USER`.
    
#### Create CTSM data directory

CTSM processes a huge stream of data files. Inputs are read from files in [netCDF][] format (.nc), which is a standardized format for representing scientific data. Outputs take the form of binaries, structured data files, logs, and various other formats. Altogether, CTSM consumes a number of files (*a.k.a* [inodes][]) and storage space that significantly grows larger with the complexity of a test case. The CTSM data directory should be prepared according to these expectations.

The [$SCRATCH file system][], a large data storage with high I/O bandwidth, is best suited for this purpose. This file system is also accessible from most compute projects. Start by creating your own personal folder under `$SCRATCH`. You may skip this step if you already have one.

```Shell
mkdir $SCRATCH/$USER
```

Switch to your `$SCRATCH` personal folder by running `cd $SCRATCH/$USER` or `cd $SCRATCH/<your-folder>`. Then start creating the directories:

```Shell
mkdir CESMDataRoot; cd CESMDataRoot
mkdir InputData CaseOutputs Archive BaselineTests
export CESMDATAROOT=`pwd`
```

Verify that the folders created match the folder structure below (run `tree` or `ls`).

```
.
└── CESMDataRoot/
    ├── CaseOutputs/
    ├── Archive/
    ├── BaselineTests/
    └── InputData/
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
INPUT_DIR      = /p/scratch/cslts/shared_data/rlmod_CLM_UCAR/inputdata
MAIL_TYPE      = 
```

Supply the missing fields:

* `PROJECT` Project ID specified during the [activate project](#activate-project) step.
* `CHARGE_ACCOUNT` Account name associated with the project ID. Run `printenv BUDGET_ACCOUNTS` to see which accounts are available.
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
INPUT_DIR      = /p/scratch/cslts/shared_data/rlmod_CLM_UCAR/inputdata
MAIL_TYPE      = all
```


### Run single point case

The simplest way to verify CTSM is to run a [built-in single point case][] with predefined input files. We will run the 1x1 Mexico test case with default Qian atmospheric data forcing.

To begin, switch to CIME scripts directory.

```Shell
cd $CTSMROOT/cime/scripts
```

#### Create case

Create a new case by running

```Shell
./create_newcase --case testMXC --res 1x1_mexicocityMEX --compset I1PtClm50SpGs --run-unsupported
cd testMXC
```
This command creates a folder named`testMXC` containing CIME environment files (`env_*.xml`) and symlinks to the CIME scripts. Make sure your working directory is set to `testMXC` in order to execute the next steps. 

#### Setup

This step prepares the machine-specific configuration for the case. Run

```
./case.setup
```

#### Build

Execute 

```
./case.build
```

This will take 5-10 minutes to finish. Build outputs are saved to `$CESMDATAROOT/CaseOutputs/testMXC/bld/`.

#### Submit

Before submitting the job you may review the job summary by executing `./preview_run`. Once you're ready, run

```
./case.submit
```

This script will automatically download the required inputs for Mexico test case if they are missing. Downloads will be skipped if `INPUT_DIR = /p/scratch/cslts/shared_data/rlmod_CLM_UCAR/inputdata` in your [CIME config file](#create-cime-config-file). Once the required inputs have been verified, the job is submitted to the queue. 

You will receive job notifications via email if `MAIL_TYPE` is set to `all` on your [CIME config file](#create-cime-config-file). You may also run these commands to check the job status:

* `squeue -u $USER --start` Check whether the job has started or not.
* `tail CaseStatus` View recent CIME activity logs. The run is successful if you see the lines `[yyyy-mm-dd hh:mm:ss]: model execution success` and `[yyyy-mm-dd hh:mm:ss]: case.run success`.
* `sacct` Checks the status of the submitted job. A job is successful if its ExitCode = 0 and its status is `COMPLETED`.

The job outputs are saved to `$CESMDATAROOT/CaseOutputs/testMXC/run/`.

### Run a regional case

In this section we will run a case with user-supplied files. The land model domain is set in North-Rhine Westphalia, Germany with a grid resolution of 300 latitudes by 300 longitudes. The atmospheric forcing data is based from the [COSMO-REA6][] dataset.

To begin, switch back to the CIME scripts directory.

```Shell
cd $CTSMROOT/cime/scripts
```

#### Create case

`./create_newcase` allows the user to specify an input directory other than the [CIME config file](#create-cime-config-file) default via the `--input-dir` option. We will use this option to access the NRW input files created by other users.

Run these commands:

```
NRW_INPUTDATA=/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x
DOMAIN_FILE=domain.lnd.300x300_NRW_300x300_NRW.190619.nc
./create_newcase --case NRW_300x300 --res CLM_USRDAT --compset I2000Clm50BgcCruGs --input-dir $NRW_INPUTDATA --run-unsupported
cd NRW_300x300
```

The succeeding steps assume your working directory is set to `NRW_300x300`. 

#### Setup

Here we introduce the [xmlchange] script which is a usefool tool for modifying the CIME environment variables saved in `env_*.xml` files. These environment variables affect the behavior of CIME case control scripts (`case.setup`, `case.build`, `case.submit`). To print the value of a CIME environment variable, use the [xmlquery][] script.

Execute the following commands to set up the NRW case.

```
./xmlchange CLM_BLDNML_OPTS="-bgc bgc -crop" 
./xmlchange ATM_DOMAIN_PATH=$NRW_INPUTDATA/clm,LND_DOMAIN_PATH=$NRW_INPUTDATA/clm
./xmlchange ATM_DOMAIN_FILE=$DOMAIN_FILE,LND_DOMAIN_FILE=$DOMAIN_FILE
./xmlchange NTASKS=96
./case.setup
```

#### Build

##### 1. Modify user namelists

Edit the CLM namelist file `user_nl_clm` using your preferred text editor (`vim`, `emacs`, `nano`, etc.). Copy the contents below to your `user_nl_clm` file (you  may skip the header descriptions prefixed with `!`).

```
!----------------------------------------------------------------------------------
! Users should add all user specific namelist changes below in the form of 
! namelist_var = new_namelist_value 
!
! EXCEPTIONS: 
! Set use_cndv           by the compset you use and the CLM_BLDNML_OPTS -dynamic_vegetation setting
! Set use_vichydro       by the compset you use and the CLM_BLDNML_OPTS -vichydro           setting
! Set use_cn             by the compset you use and CLM_BLDNML_OPTS -bgc  setting
! Set use_crop           by the compset you use and CLM_BLDNML_OPTS -crop setting
! Set spinup_state       by the CLM_BLDNML_OPTS -bgc_spinup      setting
! Set co2_ppmv           with CCSM_CO2_PPMV                      option
! Set dtime              with L_NCPL                             option
! Set fatmlndfrc         with LND_DOMAIN_PATH/LND_DOMAIN_FILE    options
! Set finidat            with RUN_REFCASE/RUN_REFDATE/RUN_REFTOD options for hybrid or branch cases
!                        (includes $inst_string for multi-ensemble cases)
!                        or with CLM_FORCE_COLDSTART to do a cold start
!                        or set it with an explicit filename here.
! Set maxpatch_glcmec    with GLC_NEC                            option
! Set glc_do_dynglacier  with GLC_TWO_WAY_COUPLING               env variable
!----------------------------------------------------------------------------------
finidat = '/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/FSpinup_300x300_NRW.clm2.r.2222-01-01-00000.nc'
fsnowaging ='/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/snicar_drdt_bst_fit_60_c070416.nc'
fsnowoptics = '/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/snicar_optics_5bnd_c090915.nc'
fsurdat = '/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/surfdata_300x300_NRW_hist_78pfts_CMIP6_simyr2000_c190619.nc'
paramfile = '/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/clm5_params.c171117.nc'
stream_fldfilename_ndep = '/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/fndep_clm_hist_b.e21.BWHIST.f09_g17.CMIP6-historical-WACCM.ensmean_1849-2015_monthly_0.9x1.25_c180926.nc'
stream_fldfilename_popdens = '/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/clmforc.Li_2017_HYDEv3.2_CMIP6_hdm_0.5x0.5_AVHRR_simyr1850-2016_c180202.nc'
stream_fldfilename_urbantv = '/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/CLM50_tbuildmax_Oleson_2016_0.9x1.25_simyr1849-2106_c160923.nc'
stream_fldfilename_lightng = '/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/clmforc.Li_2012_climo1995-2011.T62.lnfm_Total_c140423.nc'
stream_fldfilename_ch4finundated = '/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/finundated_inversiondata_0.9x1.25_c170706.nc'
megan_factors_file = '/p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm/megan21_emis_factors_78pft_c20161108.nc'
hist_mfilt = 365,
hist_nhtfrq = -24,
co2_ppmv = 367.0
urban_traffic = .false.
urban_hac = 'ON_WASTEHEAT'
hist_fincl1 =  'TLAI','TSOI','TOTSOMC','TOTVEGC','TOTECOSYSC'
hist_empty_htapes = .true.
use_init_interp = .true.
```

Similarly, edit the data atmosphere namelist file `user_nl_datm` with the contents copied from below.

```
!------------------------------------------------------------------------
! Users should ONLY USE user_nl_datm to change namelists variables
! Users should add all user specific namelist changes below in the form of
! namelist_var = new_namelist_value
! Note that any namelist variable from shr_strdata_nml and datm_nml can
! be modified below using the above syntax
! User preview_namelists to view (not modify) the output namelist in the
! directory $CASEROOT/CaseDocs
! To modify the contents of a stream txt file, first use preview_namelists
! to obtain the contents of the stream txt files in CaseDocs, and then
! place a copy of the  modified stream txt file in $CASEROOT with the string
! user_ prepended.
!------------------------------------------------------------------------
dtlimit = 5.1, 5.1, 5.1, 1.5, 1.5
mapalgo = "nn", "nn", "nn", "bilinear", "bilinear"
tintalgo = "nearest", "nearest", "linear", "linear", "lower"
streams = "datm.streams.txt.CLMCRUNCEPv7.Solar 2017 2017 2017",
        "datm.streams.txt.CLMCRUNCEPv7.Precip 2017 2017 2017",
        "datm.streams.txt.CLMCRUNCEPv7.TPQW 2017 2017 2017"
```


##### 2. Modify data atmosphere stream files

The atmospheric forcing data are specified through [DATM stream files][]. Generate the default stream files by running:

```
./preview_namelists
cp CaseDocs/datm.streams.txt.CLMCRUNCEPv7.Precip user_datm.streams.txt.CLMCRUNCEPv7.Precip
cp CaseDocs/datm.streams.txt.CLMCRUNCEPv7.Solar user_datm.streams.txt.CLMCRUNCEPv7.Solar
cp CaseDocs/datm.streams.txt.CLMCRUNCEPv7.TPQW user_datm.streams.txt.CLMCRUNCEPv7.TPQW
```

The default CRUNCEPv7 forcing data must be substituted with COSMO-REA6 forcing data. Edit the `user_datm.streams.txt.CLMCRUNCEPv7.*` files with the contents for each file specified below. 

* `user_datm.streams.txt.CLMCRUNCEPv7.Solar`
```
<?xml version="1.0"?>
<file id="stream" version="1.0">
    <dataSource>
    GENERIC
    </dataSource>
    <domainInfo>
    <variableNames>
            time    time
            xc      lon
            yc      lat
            area    area
            mask    mask
    </variableNames>
    <filePath>
        /p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm
    </filePath>
    <fileNames>
        domain.lnd.300x300_NRW_300x300_NRW.190619.nc
    </fileNames>
    </domainInfo>
    <fieldInfo>
    <variableNames>
            FSDS swdn
    </variableNames>
    <filePath>
        /p/project/cjicg41/jicg4180/COSMOREA6/forcings
    </filePath>
    <fileNames>
        2017-01.nc
        2017-02.nc
        2017-03.nc
        2017-04.nc
        2017-05.nc
        2017-06.nc
        2017-07.nc
        2017-08.nc
        2017-09.nc
        2017-10.nc
        2017-11.nc
        2017-12.nc
    </fileNames>
    <offset>
        0
    </offset>
    </fieldInfo>
</file>
```

* `user_datm.streams.txt.CLMCRUNCEPv7.Precip`
```
<?xml version="1.0"?>
<file id="stream" version="1.0">
    <dataSource>
    GENERIC
    </dataSource>
    <domainInfo>
    <variableNames>
            time    time
            xc      lon
            yc      lat
            area    area
            mask    mask
    </variableNames>
    <filePath>
        /p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm
    </filePath>
    <fileNames>
        domain.lnd.300x300_NRW_300x300_NRW.190619.nc
    </fileNames>
    </domainInfo>
    <fieldInfo>
    <variableNames>
            PRECTmms precn
    </variableNames>
    <filePath>
        /p/project/cjicg41/jicg4180/COSMOREA6/forcings
    </filePath>
    <fileNames>
        2017-01.nc
        2017-02.nc
        2017-03.nc
        2017-04.nc
        2017-05.nc
        2017-06.nc
        2017-07.nc
        2017-08.nc
        2017-09.nc
        2017-10.nc
        2017-11.nc
        2017-12.nc
    </fileNames>
    <offset>
        0
    </offset>
    </fieldInfo>
</file>
```

* `user_datm.streams.txt.CLMCRUNCEPv7.TPQW`
```
<?xml version="1.0"?>
<file id="stream" version="1.0">
    <dataSource>
    GENERIC
    </dataSource>
    <domainInfo>
    <variableNames>
            time    time
            xc      lon
            yc      lat
            area    area
            mask    mask
    </variableNames>
    <filePath>
        /p/scratch/cslts/shared_data/rcmod_TSMP-ref_SLTS/TestCases/nrw_5x/clm
    </filePath>
    <fileNames>
        domain.lnd.300x300_NRW_300x300_NRW.190619.nc
    </fileNames>
    </domainInfo>
    <fieldInfo>
    <variableNames>
        TBOT     tbot
            WIND     wind
            QBOT     shum
            PSRF     pbot
    </variableNames>
    <filePath>
        /p/project/cjicg41/jicg4180/COSMOREA6/forcings
    </filePath>
    <fileNames>
        2017-01.nc
        2017-02.nc
        2017-03.nc
        2017-04.nc
        2017-05.nc
        2017-06.nc
        2017-07.nc
        2017-08.nc
        2017-09.nc
        2017-10.nc
        2017-11.nc
        2017-12.nc
    </fileNames>
    <offset>
        0
    </offset>
    </fieldInfo>
</file>
```

Lastly, run these scripts to check for syntax errors and validity of input files. If these scripts return errors, ensure that your `user_datm.streams.txt.CLMCRUNCEPv7.*` files match exactly the contents of stream files specified above.

```
./preview_namelists
./check_input_data
```

##### 3. Execute build script

```
./case.build
```

`./case.build` will proceed with the build if there are no errors on the previous steps. 

#### Submit

Run the model with simulation time span set to the whole year of 2017:

```
./xmlchange STOP_OPTION=date,RUN_STARTDATE=2017-01-01,STOP_DATE=20171231,STOP_N=-1
./case.submit
```

Run `sacct` or `tailf CaseStatus` to track the job status (see [Submit](#submit-1) section for the 1x1 Mexico case). 

When the model execution is complete, you may change the simulation time span and then run `./case.submit` again. The simulation time span should be within 2017 since the atmospheric forcing data is supplied only for this year. Below are some examples:

* 12 hours: `./xmlchange STOP_OPTION=nhours,STOP_N=12,RUN_STARTDATE=2017-01-01,STOP_DATE=-1` 
* 30 days: `./xmlchange STOP_OPTION=ndays,STOP_N=30,RUN_STARTDATE=2017-01-01,STOP_DATE=-1` 
* 6 months: `./xmlchange STOP_OPTION=nmonths,STOP_N=6,RUN_STARTDATE=2017-01-01,STOP_DATE=-1` 

### References

* [CIME Basic Usage](https://esmci.github.io/cime/versions/master/html/users_guide/index.html)
* [CLM5.0_on_JSC_Machines](https://icg4geo.icg.kfa-juelich.de/SoftwareTools/CLM5.0_on_JSC_Machines)
* [CLM5.0 User's Guide](https://escomp.github.io/ctsm-docs/versions/master/html/index.html)
* [ESCOMP/CTSM Wiki pages](https://github.com/ESCOMP/CTSM/wiki)
* [JURECA](https://apps.fz-juelich.de/jsc/hps/jureca/index.html#) and [JUWELS](https://apps.fz-juelich.de/jsc/hps/juwels/index.html#) User Guides 

[checkout_externals]: manage_externals/checkout_externals/README.md
[netCDF]: https://www.unidata.ucar.edu/software/netcdf/docs/faq.html
[inodes]: https://unix.stackexchange.com/a/117094
[$SCRATCH file system]: https://www.fz-juelich.de/SharedDocs/FAQs/IAS/JSC/EN/JUST/FAQ_00_File_systems.html
[CIME repository]: https://github.com/HPSCTerrSys/cime
[config_machines.xml]: https://github.com/HPSCTerrSys/cime/blob/master/config/cesm/machines/config_machines.xml#L1546
[module system]: https://www.fz-juelich.de/ias/jsc/EN/Expertise/Supercomputers/JURECA/Software/Software_node.html
[config_machines.xml.]: https://github.com/HPSCTerrSys/cime/blob/master/config/cesm/machines/config_machines.xml#L1563
[Stage]: https://www.fz-juelich.de/ias/jsc/EN/Expertise/Supercomputers/JURECA/Software/Software.html?nn=1803760#Stages
[CIME]: https://esmci.github.io/cime/versions/master/html/what_cime/index.html#what-is-cime
[built-in single point case]: https://escomp.github.io/ctsm-docs/versions/master/html/users_guide/running-single-points/running-single-point-configurations.html
[COSMO-REA6]: https://reanalysis.meteo.uni-bonn.de/?COSMO-REA6
[DATM stream files]: http://www.cesm.ucar.edu/models/cesm1.2//data8/doc/c72.html
[xmlchange]: https://esmci.github.io/cime/versions/master/html/Tools_user/xmlchange.html
[xmlquery]: https://esmci.github.io/cime/versions/master/html/Tools_user/xmlquery.html
