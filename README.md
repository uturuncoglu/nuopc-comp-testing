# nuopc-comp-testing

This action allows testing model component in an isolated environment outside of any Earth system modeling application. In this case, the model component is forced by data components provided by the Community Data Models for Earth Prediction Systems ([CDEPS](https://github.com/ESCOMP/CDEPS)) to create simple yet realistic test configuration. This approach allows to test the ESMF/NUOPC "cap" easily and test it along with the development automatically.

The action mainly includes following features:

- Installs component dependencies such as third-party libraries required to build model component using [Spack package manager] (https://github.com/spack/spack). The list of dependencies needs to include the [ESMF](https://github.com/esmf-org/esmf) library.

- Creates very simple executable that includes the active component as well as the data model using Earth System Model eXecutable layer ([ESMX](https://github.com/esmf-org/esmf/tree/develop/src/addon/ESMX)). In this case, CDEPS provides the data component capability. 

- Prepares run directory for testing specified configuration.
  - Downloads and stages input data and cache it is using [cache](https://github.com/actions/cache) action
  - Creates namelist files on-the-fly by leveraging set of Python script and improved version of [ParamGen](https://github.com/ESMCI/cime/tree/master/CIME/ParamGen) by processing YAML test configuration files

- Runs the configured testing applications

- Pushes specified model output to GitHub artifacts

- Creates baseline and store it as GitHub cache. Then, checks it against the existing baseline if it is created before. The action does not store all the data generated by the output. It just calculates the MD5 hashes of each file that is selected for the baseline generation (`baseline_files` argument) and store only those hashes in a simple ASCII file.

## Documentation

## What's New

### v1

* Added caching support for dependency installations, inputs and baseline
* Added support to use `Baseline Change` label for PRs to pass baseline checking against the existing one
* Both driver and component could have multiple `input` section in test definition `.yml` files 

## Usage

### Pre-requisites

Create a workflow `.yml` file in your repositories under `.github/workflows` directory. An [example workflow](#example-component-testing-workflow) is available below. For more information, reference the GitHub Help Documentation for [Creating a workflow file](https://help.github.com/en/articles/configuring-a-workflow#creating-a-workflow-file).

Create a top level driver `.yml` file in your repositories under `.github/workflows/tests` directory. This will point component specific `.yml` files as well as definition of driver specific input and namelist files. 

Create components `.yml` file in your repositories under `.github/workflows/tests/[NAME OF THE DRIVER YAML FILE]` directory. These files will include information about definition of required input file/s and namelist files for the specific model component.

### Inputs

* `app_install_dir` - An optional path of installation directory for test configuration. The default value is set to `${{ github.workspace }}/app`.
* `architecture` - An optional value for Spack target architecture. The default value is set to `x86_64_v4`.
* `artifacts_files` - A optional list of files, directories, and wildcard patterns to save specified files as artifacts. The default value is set to `None` and the action will not save any artifacts after action ends.
* `artifacts_retention_period` - An optional number to specify artifacts retention period. The default value is set to 2-days.
* `baseline_files` - A optional list of files, directories, and wildcard patterns to be used as a baseline. The default value is set to `None` and the action will not create baseline to check it in later runs.
* `cache_input_file_list` - A optional list of files, directories, and wildcard patterns to cache input files. The default value is set to `None`.
* `component_name` - A optional argument to specify component name. The default value is set to `${{ github.event.repository.name }}`.
* `component_build` - The list of shell commands to build the component. This is required since the GitHub Action do not know anything about the component build system and requirements.
* `component_module_name` - The Fortran module name of the component that will be tested. This argument is currently required but it would be a part of the user provided YAML files.
* `data_component_name` - The optional name of data component that will be used to force the component. The valid values are `datm` (default), `dice`, `dlnd`, `docn`, `drof` and `dwav`. It would be a part of the user provided YAML files in the future.
* `dependencies` - The list of dependencies that are used to build the application. Since Spack is used to install dependencies, the given packages need to be part of the Spack distribution. The list of packages can be seen in [here](https://packages.spack.io). The ESMF package (`esmf@8.4.0b15+parallelio`) needs to be added to the list. 
* `dependencies_install_dir` - An optional path of installation directory for dependencies. The default value is set to `~/.spack-ci`.
* `test_definition` - The top level YAML file that describes the test. This is required.

#### Environment Variables

* `GH_TOKEN` - This needs to be set to `${{ github.token }}`. It will allow to run `gh` command in the composite action to access list of available baseline caches and find the correct baseline to check it. 

### Outputs

In this version there has no output for top level action resides in the component repository.

### Example component testing workflow

#### Describing configuration of component testing through using YAML specification

The description of the component testing is defined via set of YAML files. The [Noah-MP land model](https://github.com/NOAA-EMC/noahmp) example will be used in the rest of the document to give brief introduction about the YAML files, their structures and building GitHub Action to test the component . 

##### GitHub Action for component testing (.github/workflows/datm_noahmp.yaml)

```yaml
name: test_datm_lnd

on:
  push:
    branches: [ develop ]
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled]
    branches: [ develop ]
  schedule:
    - cron: '0 0 * * MON'
    - cron: '0 0 * * FRI'
  workflow_dispatch:
  
jobs:
  latest-stable:
    runs-on: ${{ matrix.os }}

    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-22.04]
        test: [test_datm_lnd]
    
    env:
      # set token to access gh command
      GH_TOKEN: ${{ github.token }}
      # installation location for application
      APP_INSTALL_DIR: ${{ github.workspace }}/app 
      # installation location for dependencies
      DEP_INSTALL_DIR: ~/.spack-ci
      # option for retention period for artifacts, default is 90 days
      ARTIFACTS_RETENTION_PERIOD: 2

    steps:
      # test component
      - name: Test Component
        uses: uturuncoglu/nuopc-comp-testing@main 
        with:
          app_install_dir: ${{ env.APP_INSTALL_DIR }}
          artifacts_files: |
            ${{ env.APP_INSTALL_DIR }}/run/PET*
            ${{ env.APP_INSTALL_DIR }}/run/*.txt
            ${{ env.APP_INSTALL_DIR }}/run/*.log  
            ${{ env.APP_INSTALL_DIR }}/run/comp.test.lnd.out.2000-01-01-75600.*
          baseline_files: |
            ${{ env.APP_INSTALL_DIR }}/run/comp.test.*.nc
          cache_input_file_list: |
            ${{ env.APP_INSTALL_DIR }}/run/INPUT
            ${{ env.APP_INSTALL_DIR }}/run/fd_nems.yaml
          component_build: |
            export PATH=${{ env.DEP_INSTALL_DIR }}/view/bin:$PATH
            export ESMFMKFILE=${{ env.DEP_INSTALL_DIR }}/view/lib/esmf.mk
            export NetCDF_ROOT=${{ env.DEP_INSTALL_DIR }}/view
            export FC=gfortran
            cd ${{ env.APP_INSTALL_DIR }}/noahmp
            mkdir build
            cd build
            cmake -DCMAKE_INSTALL_PREFIX=${{ env.APP_INSTALL_DIR }} -DOPENMP=ON ../
            make
            make install
          component_module_name: lnd_comp_nuopc
          data_component_name: datm
          dependencies: |
            zlib@1.2.12
            fms@2022.04
            esmf@8.4.0b15+parallelio
            parallelio@2.5.8+pnetcdf
          dependencies_install_dir: ${{ env.DEP_INSTALL_DIR }}
          test_definition: ${{ env.APP_INSTALL_DIR }}/noahmp/.github/workflows/tests/${{ matrix.test }}.yaml
```

The GitHub Action will run when: (1) direct push to `develop` branch, (2) pull request to `develop` branch and, (3) scheduled on Monday and Friday (required to prevent auto-removing cache entries after 7-days and only runs for default branch) and (4) manually triggered.

The action includes single test called as `test_datm_lnd` but it is also possible to define series of test using different YAML file for test configuration.
 
##### Top-level YAML file for driver (.github/workflows/tests/test_datm_lnd.yaml)

```yaml
components:
  drv:
    runseq:
      dt: 
        values: 3600
      lnd-to-atm:
        values: remapMethod=bilinear:unmappedaction=ignore:zeroregion=select:srcTermProcessing=0:termOrder:srcseq
      atm-to-lnd: 
        values: remapMethod=bilinear:unmappedaction=ignore:zeroregion=select:srcTermProcessing=0:termOrder:srcseq
      atm:
      lnd:
    input:
      field_table:
        protocol: wget
        end_point: 'https://raw.githubusercontent.com'
        files:
          - /ufs-community/ufs-weather-model/develop/tests/parm/fd_nems.yaml
    config:
      nuopc:
        name: esmxRun.config
        content:
          ESMX_attributes:
            Verbosity:
              values: high
          ALLCOMP_attributes:
            case_name:
              values: comp.test
            stop_n: 
              values: 1
            stop_option: 
              values: ndays
            stop_tod: 
              values: 0
            stop_ymd: 
              values: -999
            restart_n: 
              values: 1
            restart_option: 
              values: never
            restart_ymd: 
              values: -999
          no_group:
            ESMX_component_list:
              values: COMPA COMPB
            startTime:
              values: '2000-01-01T00:00:00'
            stopTime:
              values: '2000-01-02T00:00:00'
            logKindFlag:
              values: ESMF_LOGKIND_MULTI
            globalResourceControl:
              values: .true.
            ESMX_log_flush:
              values: .true.
            ESMX_field_dictionary: 
              values: fd.yaml
    
  lnd: test_datm_lnd/lnd.yaml

  atm: test_datm_lnd/datm.yaml
```

The top level driver `.yml` file includes information about [NUOPC run sequence](http://earthsystemmodeling.org/docs/release/latest/NUOPC_refdoc/node4.html#SECTION000411300000000000000), input files such as [NUOPC field dictionary](http://earthsystemmodeling.org/docs/release/latest/NUOPC_refdoc/node3.html#SECTION00032000000000000000), ESMX driver specific namelist file and its content and pointer for component specific `.yml` files.

* `runseq` - This section defines the NUOPC run sequence to run the components and interaction among them. `dt` is used to define coupling interval (in seconds). The rest of the content of this section is used to define the run sequence. The unmappedaction=ignore:zeroregion=select:srcTermProcessing=0:termOrder:srcseq` part ensures the bit-to-bit reproducibility for specified interpolation type, which is defined as `bilinear` in this example.

* `input` - These sections define the set of inputs that are required for the driver. There could be multiple entries under this section. For example, `field_table` section basically downloads NUOPC field dictionary from UFS Weather Model repository using `wget` command. The [Python script](https://github.com/uturuncoglu/nuopc-comp-testing/blob/main/scripts/get_input.py) that is used to download data also supports other protocols such as `ftp`, `wget`, `s3` and `s3cli`. The optional `target_directory` argument also enable to specify target directory (both absolute and relative paths are supported) for the retrieved data. 

* `config` - This section mainly used to create namelist files. It supports multiple formats: (1) Standard Fortran namelist (`nml:`)  and, (2) ESMF config (`nuopc:`) format
   
   - `name:` This section basically defines the name of the namelist file and if any other `.yml` file (driver or component ones) has section with same file name, then the content are appended to the existing namelist file. With this approach, each component could append their component specific namelist options for example top level driver ESMF config file.
   
   - `content:` This section mainly includes the content of the namelist file and uses format defined by the ParamGen tool. It is also possible to overwrite the content of this section by defining environment variable with the same name. To that end, the same file can be used for different application by changing set of namelist variables on-the-fly. 
   
> **Note**
> It is also possible to add multiple namelist files in the same format by modifying namelist tag as `nuopc1:`, `nuopc2:` etc.

* `comp*` - This section defines the pointers for component model (incl. CDEPS) specific `.yml` files. The underlying infrastructure follow the file defined in here and read with the YAML parser to include the information defined in those files into internal Python dictionary that is used to parse information and perform the different operations. 

##### YAML files for model components

##### .github/workflows/test_datm_lnd/lnd.yaml

```yaml
---
input:
  fixed:
    protocol: s3
    end_point: noaa-ufs-regtests-pds
    files:
      - input-data-20221101/FV3_fix_tiled/C96/C96.maximum_snow_albedo.tile1.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.maximum_snow_albedo.tile2.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.maximum_snow_albedo.tile3.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.maximum_snow_albedo.tile4.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.maximum_snow_albedo.tile5.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.maximum_snow_albedo.tile6.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.slope_type.tile1.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.slope_type.tile2.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.slope_type.tile3.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.slope_type.tile4.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.slope_type.tile5.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.slope_type.tile6.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.soil_type.tile1.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.soil_type.tile2.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.soil_type.tile3.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.soil_type.tile4.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.soil_type.tile5.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.soil_type.tile6.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.substrate_temperature.tile1.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.substrate_temperature.tile2.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.substrate_temperature.tile3.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.substrate_temperature.tile4.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.substrate_temperature.tile5.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.substrate_temperature.tile6.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_greenness.tile1.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_greenness.tile2.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_greenness.tile3.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_greenness.tile4.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_greenness.tile5.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_greenness.tile6.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_type.tile1.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_type.tile2.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_type.tile3.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_type.tile4.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_type.tile5.nc
      - input-data-20221101/FV3_fix_tiled/C96/C96.vegetation_type.tile6.nc
      - input-data-20221101/FV3_input_data/INPUT/C96_grid.tile1.nc
      - input-data-20221101/FV3_input_data/INPUT/C96_grid.tile2.nc
      - input-data-20221101/FV3_input_data/INPUT/C96_grid.tile3.nc
      - input-data-20221101/FV3_input_data/INPUT/C96_grid.tile4.nc
      - input-data-20221101/FV3_input_data/INPUT/C96_grid.tile5.nc
      - input-data-20221101/FV3_input_data/INPUT/C96_grid.tile6.nc
      - input-data-20221101/FV3_input_data/INPUT/grid_spec.nc
      - input-data-20221101/FV3_input_data/INPUT/oro_data.tile1.nc
      - input-data-20221101/FV3_input_data/INPUT/oro_data.tile2.nc
      - input-data-20221101/FV3_input_data/INPUT/oro_data.tile3.nc
      - input-data-20221101/FV3_input_data/INPUT/oro_data.tile4.nc
      - input-data-20221101/FV3_input_data/INPUT/oro_data.tile5.nc
      - input-data-20221101/FV3_input_data/INPUT/oro_data.tile6.nc    
    target_directory: 'INPUT'
  ic:
    protocol: wget
    end_point: 'https://raw.githubusercontent.com'
    files:
      - /esmf-org/noahmp/develop/.github/workflows/data/C96.initial.tile1.nc
      - /esmf-org/noahmp/develop/.github/workflows/data/C96.initial.tile2.nc
      - /esmf-org/noahmp/develop/.github/workflows/data/C96.initial.tile3.nc
      - /esmf-org/noahmp/develop/.github/workflows/data/C96.initial.tile4.nc
      - /esmf-org/noahmp/develop/.github/workflows/data/C96.initial.tile5.nc
      - /esmf-org/noahmp/develop/.github/workflows/data/C96.initial.tile6.nc
    target_directory: 'INPUT'
config:
  nuopc:
    name: esmxRun.config
    content: 
      no_group:
        LND_model:
          values: noahmp
        LND_petlist: 
          values: 0-5
      LND_attributes:
        Verbosity:
          values: 0
        Diagnostic:
          values: 0
        mosaic_file:
          values: INPUT/grid_spec.nc
        input_dir:
          values: INPUT/
        ic_type:
          values: custom
        num_soil_levels:
          values: 4
        forcing_height:
          values: 10
        soil_level_thickness:
          values: 0.10:0.30:0.60:1.00
        soil_level_nodes:
          values: 0.05:0.25:0.70:1.50
        dynamic_vegetation_option:
          values: 4
        canopy_stomatal_resistance_option:
          values: 2
        soil_wetness_option:
          values: 1
        runoff_option:
          values: 1
        surface_exchange_option:
          values: 3
        supercooled_soilwater_option:
          values: 1
        frozen_soil_adjust_option:
          values: 1
        radiative_transfer_option:
          values: 3
        snow_albedo_option:
          values: 1
        precip_partition_option:
          values: 4
        soil_temp_lower_bdy_option:
          values: 2
        soil_temp_time_scheme_option:
          values: 3
        surface_evap_resistance_option:
          values: 1
        glacier_option:
          values: 1
        surface_thermal_roughness_option:
          values: 2
        output_freq:
          values: 10800
        has_export:
          values: .false.
  nml:
    name: input.nml
    content:
      fms_nml:
        clock_grain:
          values: "'ROUTINE'"
        clock_flags:
          values: "'NONE'"
        domains_stack_size:
          values: 5000000
        stack_size:
          values: 0
```

The land component specific `.yml` file includes component specific information such as required input and configuration files. As it can be seen, the file includes two main sections: (1) input and (2) configuration files related. In input file section, the static information for the land component is retrieved from NOAA UFS Weather Model provided [Amazon S3 bucket](https://registry.opendata.aws/noaa-ufs-regtests/). On the other hand, the initial condition data is stored in the component repository under `.github/workflows/data/` folder and retrieved via `wget` command. In this case, all the data files will be downloaded to `INPUT` directory (relative to `github.workspace }}/app/`), which is specified by the `target_directory:` entry.

Besides of input files, the YAML file also includes set of namelist options (in `config:` section). In this case, two namelist file will be created: (1) `esmxRun.config` and (2) `input.nml`. If the namelist file exists, then the configuration options will be appended to the existing file (`name:` is used to specify the namelist file name).

##### .github/workflows/test_datm_lnd/datm.yaml

```yaml
---
input:
  forcing:
    protocol: wget
    end_point: 'https://svn-ccsm-inputdata.cgd.ucar.edu'
    files:
      - /trunk/inputdata/atm/datm7/atm_forcing.datm7.GSWP3.0.5d.v1.c170516/Precip/clmforc.GSWP3.c2011.0.5x0.5.Prec.1999-12.nc
      - /trunk/inputdata/atm/datm7/atm_forcing.datm7.GSWP3.0.5d.v1.c170516/Precip/clmforc.GSWP3.c2011.0.5x0.5.Prec.2000-01.nc
      - /trunk/inputdata/atm/datm7/atm_forcing.datm7.GSWP3.0.5d.v1.c170516/Solar/clmforc.GSWP3.c2011.0.5x0.5.Solr.1999-12.nc
      - /trunk/inputdata/atm/datm7/atm_forcing.datm7.GSWP3.0.5d.v1.c170516/Solar/clmforc.GSWP3.c2011.0.5x0.5.Solr.2000-01.nc
      - /trunk/inputdata/atm/datm7/atm_forcing.datm7.GSWP3.0.5d.v1.c170516/TPHWL/clmforc.GSWP3.c2011.0.5x0.5.TPQWL.1999-12.nc
      - /trunk/inputdata/atm/datm7/atm_forcing.datm7.GSWP3.0.5d.v1.c170516/TPHWL/clmforc.GSWP3.c2011.0.5x0.5.TPQWL.2000-01.nc
      - /trunk/inputdata/atm/datm7/atm_forcing.datm7.GSWP3.0.5d.v1.c170516/clmforc.GSWP3.c2011.0.5x0.5.TPQWL.SCRIP.210520_ESMFmesh.nc
      - /trunk/inputdata/atm/datm7/topo_forcing/topodata_0.9x1.25_USGS_070110_stream_c151201.nc
      - /trunk/inputdata/atm/datm7/topo_forcing/topodata_0.9x1.SCRIP.210520_ESMFmesh.nc
      - /trunk/inputdata/share/meshes/fv1.9x2.5_141008_ESMFmesh.nc
    target_directory: 'INPUT'
config:
  nuopc1:
    name: esmxRun.config
    content: 
      no_group:
        ATM_model:
          values: datm
        ATM_petlist:
          values: 0-5
      ATM_attributes:
        Verbosity:
          values: 0
        Diagnostic:
          values: 0
        read_restart:
          values: .false.
        orb_eccen:
          values: 1.e36
        orb_iyear:
          values: 2000
        orb_iyear_align:
          values: 2000
        orb_mode: 
          values: fixed_year
        orb_mvelp: 
          values: 1.e36
        orb_obliq: 
          values: 1.e36
        ScalarFieldCount: 
          values: 3
        ScalarFieldIdxGridNX: 
          values: 1
        ScalarFieldIdxGridNY: 
          values: 2
        ScalarFieldIdxNextSwCday: 
          values: 3
        ScalarFieldName: 
          values: cpl_scalars
        case_name:
          values: comp.test
        stop_n:
          values: 1
        stop_option:
          values: ndays
        stop_tod:
          values: 0
        stop_ymd:
          values: -999
        restart_n:
          values: 1
        restart_option:
          values: never
        restart_ymd:
          values: -999
  nuopc2:
    name: datm.streams
    content:
      no_group:
        stream_info:
          values:
            - CLMGSWP3v1.Solar01
            - CLMGSWP3v1.Precip02
            - CLMGSWP3v1.TPQW03
            - topo.observed04
        taxmode:
          values: limit, limit, limit, cycle
        mapalgo:
          values: bilinear, bilinear, bilinear, bilinear
        tInterpAlgo:
          values: coszen, nearest, linear, lower
        readMode:
          values: single, single, single, single
        dtlimit:
          values: 1.5, 1.5, 1.5, 1.5
        stream_offset:
          values: 0, 0, 0, 0
        yearFirst: 
          values: 1999, 1999, 1999, 1
        yearLast:
          values: 2000, 2000, 2000, 1
        yearAlign:
          values: 1999, 1999, 1999, 1
        stream_vectors:
          values: null, null, null, null
        stream_mesh_file:
          values: |
            INPUT/clmforc.GSWP3.c2011.0.5x0.5.TPQWL.SCRIP.210520_ESMFmesh.nc,
            INPUT/clmforc.GSWP3.c2011.0.5x0.5.TPQWL.SCRIP.210520_ESMFmesh.nc,
            INPUT/clmforc.GSWP3.c2011.0.5x0.5.TPQWL.SCRIP.210520_ESMFmesh.nc,
            INPUT/topodata_0.9x1.SCRIP.210520_ESMFmesh.nc
        stream_lev_dimname:
          values: null, null, null, null
        stream_data_files:
          values: |
            "INPUT/clmforc.GSWP3.c2011.0.5x0.5.Solr.1999-12.nc" "INPUT/clmforc.GSWP3.c2011.0.5x0.5.Solr.2000-01.nc",
            "INPUT/clmforc.GSWP3.c2011.0.5x0.5.Prec.1999-12.nc" "INPUT/clmforc.GSWP3.c2011.0.5x0.5.Prec.2000-01.nc",
            "INPUT/clmforc.GSWP3.c2011.0.5x0.5.TPQWL.1999-12.nc" "INPUT/clmforc.GSWP3.c2011.0.5x0.5.TPQWL.2000-01.nc",
            "INPUT/topodata_0.9x1.25_USGS_070110_stream_c151201.nc"
        stream_data_variables:
          values: |
            "FSDS Faxa_swdn",
            "PRECTmms Faxa_precn",
            "TBOT Sa_tbot" "WIND Sa_wind" "QBOT Sa_shum" "PSRF Sa_pbot" "PSRF Sa_pslv" "FLDS Faxa_lwdn",
            "TOPO Sa_topo"
  nml:
    name: datm_in
    content:
      datm_nml:
        datamode: 
          values: '"CLMNCEP"'
        factorfn_data:
          values: '"null"'
        factorfn_mesh:
          values: '"null"'
        flds_co2:
          values: .false.
        flds_presaero:
          values: .false.
        flds_wiso:
          values: .false.
        iradsw:
          values: 1
        model_maskfile:
          values: '"INPUT/fv1.9x2.5_141008_ESMFmesh.nc"'
        model_meshfile:
          values: '"INPUT/fv1.9x2.5_141008_ESMFmesh.nc"'
        nx_global:
          values: 144
        ny_global:
          values: 96
        restfilm:
          values: '"null"'
        export_all:
          values: .true.
```

Like land component specific YAML file, CDEPS file also includes sections for input and namelist files. As it can be seen from the example YAML file, the configuration uses `datm` data component and `CLMNCEP` data mode, which provides GSWP3 forcing to land component.

> **Note**
> The YAML file defines two different namelist file in ESMF config format: `nuopc1` for `esmxRun.config` and `nuopc2` for `datm.streams`.
