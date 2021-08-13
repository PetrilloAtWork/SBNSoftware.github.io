---
title:  ICARUS guide to writing FHiCL configuration files
author: Gianluca Petrillo (petrillo@slac.stanford.edu)
toc:    true
---

[toc]

This page goes through the typical sections of a job configuration file
in FHiCL format, and explains the recommended practices for ICARUS.

Two main chapters are included: writing a configuration from scratch
and customising an existing one. The latter is the recommended path.
Additional chapters describe the location where FHiCL configuration files
are expected to be stored, and the current (naming) conventions.


Job configuration from scratch
===============================

The order of sections in a job configuration is recommended to be:
* header (documentation)
* FHiCL inclusion (`#include`)
* prologues (`BEGIN_PROLOG`, `END_PROLOG`)
* process name (`process_name`)
* services (`services`)
* input specification (`source`)
* `physics` section:
    * module configuration (`producers`, `filters`, `analyzers`)
    * selection of modules (`trigger_paths`, `end_paths`)
* output (`outputs`)
* parameter overriding

One section of this chapter is devoted to each of these components.

Authors are encouraged to use graphical marks to highlight different sections,
including comment fences and blank lines. For example:
```
########################################################################
###  Services
########################################################################
services: {
  # ...
}
services.LArG4Parameters.ParticleKineticEnergyCut: 0.0005


########################################################################
###  Modules
########################################################################
physics: {
  producers: {
  
    # ----------------------------------------------------------------
    daqPMT: {
      # ...
    }
    
    # ----------------------------------------------------------------
    daqTPC: {
      # ...
    }    
    # ----------------------------------------------------------------
    
  } # producers
  
  # ...
  
} # physics


########################################################################
```
Indentation via spaces is recommended and, as always, mixing tabulations 
and spaces in indentation is evil and forbidden.



Header: documentation
----------------------

The FHiCL file should start with an extensive documentation section including
a short description of the purpose, the author contact information.
The type of input required should also be documented,
and the content of the output as well.
In addition, special settings should be noted.

An example ([`fcl/detsim/commissioning/triggersim_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/detsim/commissioning/triggersim_icarus.fcl)):
```
#
# File:    triggersim_icarus.fcl
# Purpose: Runs a chain to simulate ICARUS trigger primitives.
# Author:  Gianluca Petrillo (petrillo@slac.stanford.edu)
# Date:    May 17, 2021
# Version: 1.0
#
# This is a top-level configuration that can be run directly.
# 
# This job simulates the full detector in two configurations: with tiling
# windows (no trigger window overlap) and with sliding windows
# (half-overlapping).
# For each configuration, an overall trigger is simulated as well as only
# cryostat 0 ("east module") and only cryostat 1 ("west module").
# Also, for each configuration, several trigger patterns are simulated including
# single window requirements (M1 to M6), main and opposite window (M3O3 to M6O6)
# and total of main and opposite window (M2S4, M4S8, M5S10, M6S12).
#
[...]
#
# Required inputs
# ----------------
#
#  * optical detector readout: `opdaq`
#
#
# Changes
# --------
# 
# 20210517 (petrillo@slac.stanford.edu) [v1.0]
# :   original version based on `triggersim_eastmodule_icarus.fcl` v1.0
#
```
Lots of words, but they may help users a lot.



FHiCL inclusion (`#include`)
-----------------------------

The suggested order of inclusion is:
1. service configuration files
2. common module configuration files (input, output)
3. algorithm and module configuration files

Another piece of example from [`fcl/detsim/commissioning/triggersim_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/detsim/commissioning/triggersim_icarus.fcl):
```
#include "services_common_icarus.fcl"
#include "rootoutput_icarus.fcl"
#include "trigger_icarus.fcl"
```


Prologues (`BEGIN_PROLOG`, `END_PROLOG`)
-----------------------------------------

All the settings that are not in a standard _art_ table
(`process_name`, `services`, `source`, `physics`, `outputs`)
should be enclosed in a prologue section.



Process name (`process_name`)
------------------------------

The process name will appear as a part of each data product in the output
(usually omitted) and it is also used in ICARUS as part of the output file name.
It is recommended that it be kept as short as reasonably possible, ideally
in the 3 to 7 characters range.
Example:
```
process_name: Trigger
```
(or just `Trig`, which is as clear).


Services
---------

The list of services should be tailored to the task; on the other end,
predefined service bundles are not only convenient to use,
but also allow for easier maintenance. For example, when the new services
`GeometryConfigurationWriter` became mandatory for `Geometry`,
it was just matter of updating the `icarus_geometry_services` table.

One approach is to start from a service specific to the job being performed
(for example, `LArG4Parameters` for a GEANT4 job) and check 
[which bundles include it](#service-configuration-bundles);
then, to choose among these bundles the most fitting one
as a starting point, and finally to tailor it by
[adding the missing services and pruning the superfluous ones](#customising-a-table).
This is admittedly a tedious task, altough it may be easily compensated
by the reduction in runtime for the jobs.

The following paragraphs describe the available configurations
for each single _art_ service of interest to ICARUS,
and the service bundles that are expected to be used as a base
for new job configurations.
Most services have actually only one configuration (ICARUS "standard")
and a few have others which are of more limited use.
Each service and bundle also reports in which bundles it is included
directly or, in parentheses, indirectly.
The indirect inclusion is omitted for very basic services which end up
included in (almost) every bundle.
Some bundles also report "suggested" use cases.

> This section contains many details and the nature of this documentation
> makes it very prone to fall out of date: consider cross-checking
> the information with the current version of the code.

### Standard configurations for the most common services


#### _art_ and low level services

`scheduler`
:   basic _art_ settings; optional.

    _Bundles:_ always included via `icarus_art_services`; no individual preset.


`TimeTracker` and `MemoryTracker`
:   keep track of used memory; disable in jobs with large amount of events
    to save memory and storage space
    (output files may be huge, esp. `MemoryTracker`'s).
    
    _Bundles:_ always included via `icarus_art_services`; no individual preset.


`DuplicateEventTracker`
:   currently from `icaruscode`, immediately halts when a duplicate event is
    detected. Disable it if the input sample is known to include duplicate
    event ID and there is absolutely no way to filter them out.
    
    _Bundles:_ always included via `icarus_art_services`; no individual preset.


`message` ([_[`icarusalg`]_ `fcl/services/messages_icarus.fcl`](https://github.com/SBNSoftware/icarusalg/blob/develop/fcl/services/messages_icarus.fcl))
:   several definitions available; the most common:
     * `icarus_message_services`: "whatever" option
     * `icarus_message_services_interactive`: for interactive jobs (output on screen)
     * `icarus_message_services_interactive_debug`: adds debugging messages on file
     
    _Bundles:_ all.


`TFileService`
:   _Bundles:_ always included via `icarus_minimum_services`.


`RandomNumberGenerator`
:   _art_ random engine manager, used in LArSoft with `NuRandomService`.

    _Bundles:_ `icarus_random_services`; no individual preset.


`NuRandomService` ([_[`nurandom`]_ `nurandom/RandomUtils/Providers/seedservice.fcl`](https://cdcvs.fnal.gov/redmine/projects/nurandom/repository/revisions/master/entry/nurandom/RandomUtils/Providers/seedservice.fcl))
:   several configurations available; the most relevant:
    * `random_NuRandomService` (safest, but to reproduce results a "master seed" is required)
    * `per_event_NuRandomService` (most recommended, except for jobs running specific modules, esp. event generators like `GENIEGen` and `CORSIKAGen`)
    
    This service should be included directly in a configuration only as
    an override; for first time inclusion, the random engine services
    bundle (`icarus_random_services`) should be used instead.
    
    _Bundles:_ `icarus_random_services`.


* random engine services ([_[`icaruscode`]_ `fcl/services/services_common_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_common_icarus.fcl))
:   random number generator management:
    ```@table::icarus_random_services  # from `services_common_icarus.fcl` ```;
    include only in generation and simulation jobs
    (in most jobs, a different `NuRandomService` configuration is recommended).
    * `RandomNumberGenerator`
    * `NuRandomService` (`random_NuRandomService`)
    
    _Bundles:_ `icarus_wirecalibration_minimum_services`, `icarus_common_services`
    (`icarus_gen_services`, `icarus_simulation_basic_services`,
    `icarus_detsim_dark_services`, `icarus_detsim_services`).


`FileCatalogMetadata` ([_[`icaruscode`]_ `fcl/services/sam_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/sam_icarus.fcl))
:   although presets are provided as placeholders, when its settings are important
    (i.e. in production) specific configuration should be designed:
    * `art_file_catalog_mc`
    * `art_file_catalog_data`


#### Common LArSoft services

geometry services ([_[`icarusalg`]_`icarusalg/Geometry/geometry_icarus.fcl`](https://github.com/SBNSoftware/icarusalg/blob/develop/icarusalg/Geometry/geometry_icarus.fcl))
:   geometry services come in a bundle
    (```@table::icarus_geometry_services  # from `geometry_icarus.fcl` ```).
    For more information see the
    [ICARUS geometry wiki page](../Detector_geometry.md).
    Main available configurations:
    * `icarus_geometry_services`: standard configuration (recommended);
    * `icarus_nooverburden_geometry_services`: configuration without concrete overburden (for commissioning and first analyses);
    * `icarus_overburden_geometry_services`: configuration with concrete overburden (for SBN and full dataset analyses).
    
    _Bundles:_ all except _art_-only ones (`icarus_art_services`).


`LArPropertiesService` ([_[`icarusalg`]_ `fcl/services/larproperties_icarus.fcl`](https://github.com/SBNSoftware/icarusalg/blob/develop/fcl/services/larproperties_icarus.fcl))
:   (`icarus_properties`)
    
    _Bundles:_ all except `icarus_art_services`.


`DetectorClocksService` ([_[`icarusalg`]_ `fcl/services/detectorclocks_icarus.fcl`](https://github.com/SBNSoftware/icarusalg/blob/develop/fcl/services/detectorclocks_icarus.fcl))
:   more than one configuration available:
    * `icarus_detectorclocks` is the standard one;
    * `icarus_detectorclocks_data_notiming` would fit older data runs.
    
    _Bundles:_ all except `icarus_art_services`.


`DetectorPropertiesService` ([_[`icarusalg`]_ `fcl/services/detectorproperties_icarus.fcl`](https://github.com/SBNSoftware/icarusalg/blob/develop/fcl/services/detectorproperties_icarus.fcl))
:   (`icarus_detproperties`)
    
    _Bundles:_ all except `icarus_art_services`.

`DetPedestalService` ([_[`icaruscode`]_ `fcl/services/calibrationservices_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/calibrationservices_icarus.fcl))
:   currently a placeholder with no content (`icarus_detpedestalservice`)
    
    _Bundles:_ included via `icarus_calibration_services`.


`ChannelStatusService` ([_[`icaruscode`]_ `fcl/services/calibrationservices_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/calibrationservices_icarus.fcl))
:   currently a placeholder with no content (`icarus_channelstatusservice`)
    
    _Bundles:_ always included via `icarus_calibration_services`.


`icarus_calibration_services` ([_[`icaruscode`]_ `fcl/services/calibrationservices_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/calibrationservices_icarus.fcl))
:   currently includes:
    * `DetPedestalService`
    * `ChannelStatusService`
    
    _Bundles:_ `icarus_wirecalibration_minimum_services`
      (`icarus_wirecalibration_services`, `icarus_detsim_dark_services`,
      `icarus_detsim_services`).


`MagneticField` ([_[`lardata`]_ `lardata/Utilities/magfield_larsoft.fcl`](https://github.com/LArSoft/lardata/blob/develop/lardata/Utilities/magfield_larsoft.fcl))
:   Configuration (`no_mag_larsoft`) is from LArSoft;
    `LArG4` module requires this to tell that there is no magnetic field.
    
    Bundles: `icarus_g4_dark_services`.


`SpaceChargeService` ([_[`icaruscode`]_ `icaruscode/fcl/services/simulationservices_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/correctionservices_icarus.fcl))
:   space charge configuration may change content and location with
    [`icaruscode` issue #236](https://github.com/SBNSoftware/icaruscode/issues/236)
    * `icarus_spacecharge`: standard setting and no space charge simulaton
    
    _Bundles:_ `icarus_simulation_basic_services`
    (`icarus_g4_dark_services`, `icarus_simulation_dark_services`, 
    `icarus_g4_services`, `icarus_simulation_services`).

`SignalShapingICARUSService` ([_[`icaruscode`]_ `icaruscode/TPC/Utilities/signalservices_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/TPC/Utilities/signalservices_icarus.fcl))
:   settings for TPC digitisation 
    
    _Bundles:_ `icarus_wirecalibration_minimum_services`
      (`icarus_wirecalibration_services`, `icarus_detsim_dark_services`,
      `icarus_detsim_services`).


`LArG4Parameters` ([_[`icaruscode`]_ `icaruscode/fcl/services/simulationservices_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/simulationservices_icarus.fcl))
:   standard settings for `LArG4` (`icarus_largeantparameters`);
    it is usually overridden key by key when needed.
    
    _Bundles:_ `icarus_simulation_basic_services`
    (`icarus_g4_dark_services`, `icarus_simulation_dark_services`,
    `icarus_g4_services`, `icarus_simulation_services`).


`LArVoxelCalculator` ([_[`icaruscode`]_ `fcl/services/simulationservices_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/simulationservices_icarus.fcl))
:   specific to `LArG4` (`icarus_larvoxelcalculator`)
    
    _Bundles:_ `icarus_simulation_basic_services`
    (`icarus_g4_dark_services`, `icarus_simulation_dark_services`, 
    `icarus_g4_services`, `icarus_simulation_services`).


`PhotonVisibilityService` ([_[`icaruscode`]_ `fcl/services/photpropservices_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/photpropservices_icarus.fcl))
:   * `icarus_photonvisibilityservice`: standard configuration
    * many legacy configurations are also available
    
    _Bundles:_ `icarus_g4_services`, `icarus_simulation_services`.


`BackTrackerService` ([_[`larsim`]_ `larsim/MCCheater/backtrackerservice.fcl`](https://github.com/LArSoft/larsim/blob/develop/larsim/MCCheater/backtrackerservice.fcl))
:   configuration (`standard_backtrackerservice`) is from LArSoft
    
    _Bundles:_ `icarus_backtracking_services`.


`ParticleInventoryService` ([_[`larsim`]_ `larsim/MCCheater/particleinventoryservice.fcl`](https://github.com/LArSoft/larsim/blob/develop/larsim/MCCheater/particleinventoryservice.fcl))
:   configuration (`standard_particleinventoryservice`) is from LArSoft
    
    _Bundles:_ `icarus_backtracking_services`.


`PhotonBackTrackerService` ([_[`larsim`]_ `larsim/MCCheater/photonbacktrackerservice.fcl`](https://github.com/LArSoft/larsim/blob/develop/larsim/MCCheater/photonbacktrackerservice.fcl))
:   configuration (`standard_photonbacktrackerservice`) is from LArSoft
    
    _Bundles:_ `icarus_backtracking_services`.


`icarus_backtracking_services` ([_[`icaruscode`]_ `icaruscode/fcl/services/simulationservices_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/simulationservices_icarus.fcl))
:   "backtracking" bundle, may be overkill since photon backtracking is
    less often needed:
    * `BackTrackerService` (`standard_backtrackerservice`)
    * `PhotonBackTrackerService` (`standard_photonbacktrackerservice`)
    * `ParticleInventoryService` (`standard_particleinventoryservice`)
    
    _Bundles:_ .


### Service configuration bundles

Each of these incremental configuration tables is suitable to be the baseline
for a job configuration.


`icarus_basic_services` ([_[`icarusalg`]_ `fcl/services/icarus_basic_services.fcl`](https://github.com/SBNSoftware/icarusalg/blob/develop/fcl/services/services_basic_icarus.fcl))
:   ICARUS/LArSoft essential services and no _art_ utility service:
    * `message` (`icarus_message_services_interactive`)
    * geometry
    * `LArPropertiesService`
    * `DetectorClocksService`
    * `DetectorPropertiesService`
    
    This bundle ends up being included pretty much everywhere.
     
    _Bundles:_ `icarus_minimum_services`
    (`icarus_common_services`, `icarus_wirecalibration_minimum_services`).
    
    _Suggested for:_ data dumpers requiring some knowledge of detector configuration.


`icarus_art_services` ([_[`icaruscode`]_ `fcl/services/services_common_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_common_icarus.fcl))
:   collection of _art_ and other low-level services that is _usually_ convenient to have;
    it ends up included pretty much everywhere.
    It includes:
    * `scheduler`
    * `message` (like `icarus_basic_services`)
    * `TimeTracker`
    * `MemoryTracker`
    * `DuplicateEventTracker`
    
    _Bundles:_ `icarus_minimum_services`
      (`icarus_common_services`, `icarus_wirecalibration_minimum_services`).
    
    _Suggested for:_ simple data dumpers and non-LArSoft data file management.


`icarus_minimum_services` ([_[`icaruscode`]_ `fcl/services/services_common_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_common_icarus.fcl))
:   includes `icarus_art_services` _and_ `icarus_basic_services`.
    Except for special cases (and for _gallery_ jobs)
    this is the minimal service configuration needed for ICARUS physics jobs.
    It includes:
    * `TFileService` (with [standard output file name](#output-file-names))
    * `FileCatalogMetadata` (`art_file_catalog_mc`)
    * via `icarus_art_services`:
        * `scheduler`
        * `TimeTracker`, `MemoryTracker`
        * `DuplicateEventTracker`
    * via `icarus_basic_services`:
        * `message` (`icarus_message_services_interactive`)
        * geometry
        * `LArPropertiesService`
        * `DetectorClocksService`
        * `DetectorPropertiesService`
    
    _Suggested for:_ high level reconstruction
    (if calibration is not needed).
    
    _Bundles:_ `icarus_common_services` (and derived), 
    `icarus_wirecalibration_minimum_services`
    (`icarus_wirecalibration_services`, `icarus_detsim_dark_services`,
    `icarus_detsim_services`).


`icarus_common_services` ([_[`icaruscode`]_ `fcl/services/services_common_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_common_icarus.fcl))
:   minimal services, plus random generation management
    * via `icarus_minimum_services`:
        * via `icarus_art_services`:
            * `scheduler`
            * `TimeTracker`, `MemoryTracker`
            * `DuplicateEventTracker`
        * via `icarus_basic_services`:
            * `message` (`icarus_message_services_interactive`)
            * geometry
            * `LArPropertiesService`
            * `DetectorClocksService`
            * `DetectorPropertiesService`
    * via `icarus_random_services`:
        * `RandomNumberGenerator`
        * `NuRandomService` (`random_NuRandomService`)

    _Suggested for:_ simulation not involving TPC signal processing;
    fallback in case of doubt.
    
    _Bundles:_ `icarus_gen_services`, `icarus_prod_services`,
    `icarus_simulation_basic_services`
    (`icarus_g4_dark_services`, `icarus_simulation_dark_services`,
    `icarus_g4_services`, `icarus_g4_services`,
    `icarus_simulation_services`).


`icarus_wirecalibration_minimum_services` ([_[`icaruscode`]_ `fcl/services/services_common_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_common_icarus.fcl))
:   minimal services plus signal shaping and calibration services
    * `SignalShapingICARUSService`
    * via `icarus_calibration_services`:
        * `DetPedestalService`
        * `ChannelStatusService`
    * via `icarus_minimum_services`:
        * via `icarus_art_services`:
            * `scheduler`
            * `TimeTracker`, `MemoryTracker`
            * `DuplicateEventTracker`
        * via `icarus_basic_services`:
            * `message` (`icarus_message_services_interactive`)
            * geometry
            * `LArPropertiesService`
            * `DetectorClocksService`
            * `DetectorPropertiesService`

    _Bundles:_ `icarus_wirecalibration_services`
    (`icarus_detsim_dark_services`, `icarus_detsim_services`).
    
    _Suggested for:_ TPC signal processing.


`icarus_wirecalibration_services` ([_[`icaruscode`]_ `fcl/services/services_common_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_common_icarus.fcl))
:   wire calibration "minimal" services, plus random generator management
    * via `icarus_wirecalibration_minimum_services`:
        * `SignalShapingICARUSService`
        * via `icarus_calibration_services`:
            * `DetPedestalService`
            * `ChannelStatusService`
        * via `icarus_minimum_services` (via `icarus_art_services`):
            * `scheduler`
            * `TimeTracker`, `MemoryTracker`
            * `DuplicateEventTracker`
        * via `icarus_minimum_services` (via `icarus_basic_services`):
            * `message` (`icarus_message_services_interactive`)
            * geometry
            * `LArPropertiesService`
            * `DetectorClocksService`
            * `DetectorPropertiesService`
    * random engine services (`icarus_random_services`)
    
    _Bundles:_ `icarus_detsim_dark_services` (`icarus_detsim_services`).
    
    _Suggested for:_ complete TPC digitisation simulation (`DetSim`).


`icarus_prod_services` ([_[`icaruscode`]_ `fcl/services/services_common_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_common_icarus.fcl))
:   adds metadata services:
    * via `icarus_common_services` (via `icarus_minimum_services`):
        * via `icarus_art_services`:
            * `scheduler`
            * `TimeTracker`, `MemoryTracker`
            * `DuplicateEventTracker`
        * via `icarus_basic_services`:
            * `message` (`icarus_message_services_interactive`)
            * geometry
            * `LArPropertiesService`
            * `DetectorClocksService`
            * `DetectorPropertiesService`
    * via `icarus_common_services` (via `icarus_random_services`):
        * `RandomNumberGenerator`
        * `NuRandomService` (`random_NuRandomService`)
    * `FileCatalogMetadata` (`art_file_catalog_mc`)
    
    _Suggested for:_ in principle, production jobs. In practice, try to avoid it
    (production has their own wrappers, and they are better than this).


`icarus_gen_services` ([_[`icaruscode`]_ `fcl/services/services_icarus_simulation.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_icarus_simulation.fcl))
:   event generation preset
    * via `icarus_common_services` (via `icarus_minimum_services`):
        * via `icarus_art_services`:
            * `scheduler`
            * `TimeTracker`, `MemoryTracker`
            * `DuplicateEventTracker`
        * via `icarus_basic_services`:
            * `message` (`icarus_message_services_interactive`)
            * geometry
            * `LArPropertiesService`
            * `DetectorClocksService`
            * `DetectorPropertiesService`
    
    _Suggested for:_ event generators


`icarus_simulation_basic_services` ([_[`icaruscode`]_ `fcl/services/services_icarus_simulation.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_icarus_simulation.fcl))
:   event simulation component:
    * via `icarus_common_services` (via `icarus_minimum_services`):
        * via `icarus_art_services`:
            * `scheduler`
            * `TimeTracker`, `MemoryTracker`
            * `DuplicateEventTracker`
        * via `icarus_basic_services`:
            * `message` (`icarus_message_services_interactive`)
            * geometry
            * `LArPropertiesService`
            * `DetectorClocksService`
            * `DetectorPropertiesService`
    * `LArG4Parameters`
    * `LArVoxelCalculator`
    * `SpaceChargeService`
    
    _Suggested for:_ nothing.
    
    _Bundles:_ `icarus_g4_dark_services`
      (`icarus_g4_services`, `icarus_g4_services`),
      `icarus_simulation_dark_services` (`icarus_simulation_services`).


`icarus_g4_dark_services` ([_[`icaruscode`]_ `fcl/services/services_icarus_simulation.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_icarus_simulation.fcl))
:   `LArG4` services without no light simulation.
    * via `icarus_simulation_basic_services`:
        * via `icarus_common_services` (via `icarus_minimum_services`):
            * via `icarus_art_services`:
                * `scheduler`
                * `TimeTracker`, `MemoryTracker`
                * `DuplicateEventTracker`
            * via `icarus_basic_services`:
                * `message` (`icarus_message_services_interactive`)
                * geometry
                * `LArPropertiesService`
                * `DetectorClocksService`
                * `DetectorPropertiesService`
        * `LArG4Parameters`
        * `LArVoxelCalculator`
        * `SpaceChargeService`
    * `MagneticField` (no magnetic field)
    
    _Bundles:_ `icarus_g4_services`.
    
    _Suggested for:_ `LArG4` simulation when scintillation light
    is not propagated.


`icarus_detsim_dark_services` ([_[`icaruscode`]_ `fcl/services/services_icarus_simulation.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_icarus_simulation.fcl))
:   * via `icarus_wirecalibration_services`
      (via `icarus_wirecalibration_minimum_services`):
        * `SignalShapingICARUSService`
        * via `icarus_calibration_services`:
            * `DetPedestalService`
            * `ChannelStatusService`
        * via `icarus_minimum_services` (via `icarus_art_services`):
            * `scheduler`
            * `TimeTracker`, `MemoryTracker`
            * `DuplicateEventTracker`
        * via `icarus_minimum_services` (via `icarus_basic_services`):
            * `message` (`icarus_message_services_interactive`)
            * geometry
            * `LArPropertiesService`
            * `DetectorClocksService`
            * `DetectorPropertiesService`
    * via `icarus_wirecalibration_services`:
        * random engine services (`icarus_random_services`)
    
    _Bundles:_ `icarus_detsim_dark_services`.
    
    _Suggested for:_ TPC digitisation.


`icarus_simulation_dark_services` ([_[`icaruscode`]_ `fcl/services/services_icarus_simulation.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_icarus_simulation.fcl))
:   * via `icarus_simulation_basic_services`:
        * via `icarus_common_services` (via `icarus_minimum_services`):
            * via `icarus_art_services`:
                * `scheduler`
                * `TimeTracker`, `MemoryTracker`
                * `DuplicateEventTracker`
            * via `icarus_basic_services`:
                * `message` (`icarus_message_services_interactive`)
                * geometry
                * `LArPropertiesService`
                * `DetectorClocksService`
                * `DetectorPropertiesService`
        * `LArG4Parameters`
        * `LArVoxelCalculator`
        * `SpaceChargeService`
    
    _Bundles:_ `icarus_simulation_services`.


`icarus_g4_services` ([_[`icaruscode`]_ `fcl/services/services_icarus_simulation.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_icarus_simulation.fcl))
:   * via `icarus_g4_dark_services`
      (via `icarus_simulation_basic_services`):
        * via `icarus_common_services` (via `icarus_minimum_services`):
            * via `icarus_art_services`:
                * `scheduler`
                * `TimeTracker`, `MemoryTracker`
                * `DuplicateEventTracker`
            * via `icarus_basic_services`:
                * `message` (`icarus_message_services_interactive`)
                * geometry
                * `LArPropertiesService`
                * `DetectorClocksService`
                * `DetectorPropertiesService`
        * `LArG4Parameters`
        * `LArVoxelCalculator`
        * `SpaceChargeService`
    * via `icarus_g4_dark_services`
        * `MagneticField` (no magnetic field)
    * `PhotonVisibilityService`
    
    _Suggested for:_ complete GEANT4 simulation.


`icarus_detsim_services` ([_[`icaruscode`]_ `fcl/services/services_icarus_simulation.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_icarus_simulation.fcl))
:   complete digitisation service bundle, may include PMT calibration in future
    * via `icarus_detsim_dark_services`
      (via `icarus_wirecalibration_services`, 
      via `icarus_wirecalibration_minimum_services`):
        * `SignalShapingICARUSService`
        * via `icarus_calibration_services`:
            * `DetPedestalService`
            * `ChannelStatusService`
        * via `icarus_minimum_services` (via `icarus_art_services`):
            * `scheduler`
            * `TimeTracker`, `MemoryTracker`
            * `DuplicateEventTracker`
        * via `icarus_minimum_services` (via `icarus_basic_services`):
            * `message` (`icarus_message_services_interactive`)
            * geometry
            * `LArPropertiesService`
            * `DetectorClocksService`
            * `DetectorPropertiesService`
    * via `icarus_detsim_dark_services`
      (via `icarus_wirecalibration_services`):
        * random engine services (`icarus_random_services`)
    
    _Suggested for:_ full detector digitisation including calibration.


`icarus_simulation_services` ([_[`icaruscode`]_ `fcl/services/services_icarus_simulation.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/services/services_icarus_simulation.fcl))
:   * via `icarus_simulation_dark_services`
      (via `icarus_simulation_basic_services`):
        * via `icarus_common_services` (via `icarus_minimum_services`):
            * via `icarus_art_services`:
                * `scheduler`
                * `TimeTracker`, `MemoryTracker`
                * `DuplicateEventTracker`
            * via `icarus_basic_services`:
                * `message` (`icarus_message_services_interactive`)
                * geometry
                * `LArPropertiesService`
                * `DetectorClocksService`
                * `DetectorPropertiesService`
        * `LArG4Parameters`
        * `LArVoxelCalculator`
        * `SpaceChargeService`
    * `PhotonVisibilityService`



Input specification (`source`)
-------------------------------

If the job takes an existing file as input, it is recommended that
_no `source` be specified_: `lar` will make up one with `RootInput`
if an input file is specified on command line.

If the job has no input, instead, it is recommended that the input configuration
from [`fcl/configurations/emptyevent_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/configurations/emptyevent_icarus.fcl)
be used:
```
source: @local::emptyevent_icarus  # from `emptyevent_icarus.fcl`
```
The key feature of it is currently the timestamp plugin that enables the use
of `perEvent` random policy of `NuRandomService`.



Module configuration (`physics`: `producers`, `filters`, `analyzers`)
----------------------------------------------------------------------

Naming of the modules (`module_label`) has some constraints that are explained
in the ["Standard ICARUS module labels" section](#standard-icarus-module-labels).

The general guideline for module configuration is to try to stick
as much as possible to predefined configurations, amended as needed
(i.e., same guideline as for services):
```
physics: {
  producers: {
    largeant: {
      @table::icarus_largeant
      KeepParticlesInVolumes: [ "volCryostat" ]
    }
  }
}
```
Prefer the syntax above to the one overriding at the bottom:
```
physics: {
  producers: {
    largeant: @local::icarus_largeant
  }
}
physics.producers.largeant.KeepParticlesInVolumes: [ "volCryostat" ]
```
which is shorter but may fragment the configuration making it harder to read.



Selection of modules (`trigger_paths`, `end_paths`)
----------------------------------------------------

As a reminder, _art_ uses two levels of paths:
1. "paths", lists of modules to be run sequentially
2. lists of paths (`trigger_paths` for producers and filters
   and `end_paths` for analyzer modules) which define which paths to run

The organisation of producer and filter modules into paths may be driven
by necessity if filtering is involved, since `RootOutput` may
(via `SelectEvents`) decide whether to write an event according to the outcome
of a path.
Otherwise, if some sequences of modules are independent it is suggested
that they be kept each one in its own path for clarity,
and they be otherwise merged into a single one.

It is suggested that analysis and output paths (both part of the `end_paths`)
be kept separate for maintenance reasons; _art_ will remind that this
is inconsequential from the point of view of execution though.

Finally, it is recommended that `trigger_paths` and `end_paths` be specified
only if some paths have been defined that are not meant to be run.
Otherwise, for example:
```
physics: {
  producers: {
    ophit:   @local::icarus_ophitfinder
    opflash: @local::icarus_opflash
  }
  
  optical: [ ophit, opflash ]
  stream: [ rootoutput ]
}
outputs.rootoutput: @local::icarus_rootoutput  # from `rootoutput_icarus.fcl`
```
_art_ will figure out that it has to run both `optical` and `stream` paths.



Output (`outputs`)
-------------------

It is recommended that a boilerplate output configuration be used
for all jobs using an input file (`RootInput`: generation jobs are excluded):
```
outputs.rootoutput: @local::icarus_rootoutput  # from `rootoutput_icarus.fcl`
```
This sets the standard output name and compression level;
`dataTier` may need overriding though (current default is `simulation`).

Generation jobs can also derive from `icarus_rootoutput`,
but there is currently not much gain (it may facilitate future changes though).
In particular, the output file name should be overridden to follow
[ICARUS convention](#Output-file-names).



Parameter overriding: customisation and finalisation
-----------------------------------------------------

The recommendation is to reduce overriding at the end to the bare minimum,
since it fragments the configuration, and to rather prefer extending the
configuration tables as shown in
["Customising a table" section below](#customising-a-table)
and mentioned in [Module configuration](#module-configuration).

For the same reason, it is also suggested that overriding directives
be kept close to what they are overriding: immediately after the `services`
table for services, and `physics` for modules.



Job from a "parent" configuration
==================================

It is **strongly recommended** that a standard configuration be used
as a starting point for a new one by means of inclusion rather than
by pasting the content, which is a huge maintenance burden.

In this case, the pattern is:
1. identify the parent configuration that better suites as starting point
2. include it and overwrite the relevant tables

The order of sections in a job configuration is recommended to be
the same as for a self-standing job configuration, with the exception
that prologue sections will be omitted.


### Customising a table

To override or add a single element of a table, a key with full path is the best option:
```
physics.producers.discrimopdaq.OpticalWaveforms: "daqPMT"
```
```
services.IICARUSChannelMap: @local::icarus_channelmappinggservice
```
For more that three overrides, though, it should be considered to "rewrite"
the whole table, possibly referring to the original one as a starting point.
This makes it _a lot_ easier to maintain the configuration.
For example, this is how
[`fcl/gen/single/prod_eminus_0-1GeV_isotropic_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/gen/single/prod_eminus_0-1GeV_isotropic_icarus.fcl)
_should have been written_:
```
physics.producers.generator: {
  @table::physics.producers.generator
  
  PDG: [ 11 ]            # e-
  PosDist: 0             # Flat position dist.
  X0: [ 128.0 ]
  Y0: [ 0.0 ]
  Z0: [ 518.5 ]
  # ... and many more
}
```
To remove a key from a table, remember the `@erase` syntax:
```
services.BackTrackerService: @erase
```


Header: documentation
----------------------

Together with the usual purpose and author contact information,
the header should clearly include which is the parent file,
and what this configuration changes.
For the common details, the parent file documentation should suffice
and in fact it would be better not refer to it rather than
to repeat details from there, since they may change.



FHiCL inclusion (`#include`)
-----------------------------

The standard inclusion order when overriding a job configuration is:
1. additional service and module configuration needed and not present in the parent file;
2. the parent file itself.

It is usually not possible to include configuration files _after_ the parent file,
since the configurations typically contain settings in a prologue
and the parent job configuration will have already included non-prologue definitions.
Nevertheless, specially crafted "override" configuration files may have been
designed explicitly for that use.
For example ([`fcl/gen/corsika/overburden/prodcorsika_overburden_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/gen/corsika/overburden/prodcorsika_overburden_icarus.fcl)): 
```
#include "prodcorsika_standard_icarus.fcl"

# turn to overburden geometry:
#include "use_overburden_geometry_icarus.fcl"
```


Process name
-------------

It should always be checked if it makes sense to keep the inherited process
name — most of the times, that is not the case.
Overriding it is as simple as writing a new one:
```
process_name: GenNuMI
```


Services
---------

It is customary to have the file produced by `TFileService` named after
the FHiCL configuration generating it, which may require overwriting it
from the inherited setting if the job has no input file.
This is from [`fcl/gen/corsika/prodcorsika_protononly_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/gen/corsika/prodcorsika_protononly_icarus.fcl):
```
services.TFileService.fileName: "Supplemental-prodcorsika_protononly_icarus_%tc-%p.root"
```
[Output file naming conventions](#output-file-names) are documented below.


Output
-------

The very same consideration as for `TFileService` output file name
holds for the _art_/ROOT file as well.
This is again from [`fcl/gen/corsika/prodcorsika_protononly_icarus.fcl`](https://github.com/SBNSoftware/icaruscode/blob/develop/fcl/gen/corsika/prodcorsika_protononly_icarus.fcl):
```
outputs.out1.fileName: "prodcorsika_protononly_icarus_%tc-%p.root"
```
[Output file naming conventions](#output-file-names) are documented below.


Where to save FHiCL configuration files
========================================

There are two main location categories for FHiCL configuration files:
1. `fcl` directory tree
2. the source directory where the configured code is into

A rule of thumb for job configuration is:
1. if the configuration is int


Conventions
============


Output file names
------------------

The customary format of output file names follows these examples:
* _art_/ROOT file from `RootOutput`: `simulation_genie_icarus_bnb_volCryostat_20210729T021653-GenGenie_20210729T022529-G4.root`
* output from `TFileService`: `Supplemental-simulation_genie_icarus_bnb_volCryostat_20210729T021653-GenGenie_20210729T022529-G4.root`

Explanation:
* Output file basename reflects the job that has created them first; in the example `simulation_genie_icarus_bnb_volCryostat.fcl`.
* A timestamp makes the files unique, and the name of the process describes
  the content;
  while knowing when the file was created has also some intrinsic value,
  the first timestamp can be replaced by something guaranteed to be unique
  (e.g. run/subrun number if the workflow guarantees uniqueness, or a UUID),
  and they may be omitted for the stages following the first one.
* Each stage extends the base name of its input file (`%ifb` in _art_ lingo).
  At least the name of the stage should always be appended (`%p`).
* The `TFileService` output name is built to be as similar as possible
  to the _art_/ROOT (timestamps will still differ) for ease of matching.
  The `Supplemental-` label is the current standard.


Standard ICARUS module labels
------------------------------

The module labels determine data product names, so standard data product names
require standardised module labels. In this section a few of them are listed.

### Generation (`Gen` stage)

In generation jobs, the reference data products are of type `simb::MCParticle`.
The ones from the "signal" generator module is usually called `generator`.
That is the case for example for `GENIEGen`, `SingleGen`, and sometimes even
`CORSIKAGen`.
Additional generators should describe the interactions,
like `cosmgen` for cosmic rays, `radiogen` for radioactive decay, etc.

### Propagation through the detector (`G4` stage)

The label for GEANT4 module, legacy or not, is so much traditional in LArSoft
that it is even hard-wired in some code, so it is recommended that the _final_
run of it (or any mockup) be called `largeant`.
Relevant data products include `simb::MCParticle`, `sim::SimChannel`,
`sim::SimPhotons`, `sim::SimEnergyDeposit`.
When these data products are produced by separate modules, the one producing
`simb::MCParticle` should be the one called `largeant`.

In case this simulation is split, like for example for the simulation of cosmic rays
in time with the beam spill window, there should be a merge step, and the name
of that step, producing `sim::SimChannels` etc., must be `largeant`.


### Digitisation (`DetSim` stage) and data decoding (`Stage0`)

The output of this stage is equivalent for simulation and data.
Standards are `daqPMT` for `raw::OpDetWaveform` data products,
`raw::RawDigit`

### Reconstruction (`Reco`, `Stage1`)

Higher level reconstruction modules are usually much more flexible
in accepting input, so standardised names are less of an imperative here.
The starting hit collection (`recob::Hit`) has been called `gaushit`.

### Output module and path

The name `rootoutput` is suggested for the `RootOutput` output module.
The name `streams` is suggested for the output end path.
A typical definition would be:
```
physics.streams: [ rootoutput ]
outputs.rootoutput: @local::icarus_rootoutput  # from `rootoutput_icarus.fcl`
```


Glossary
=========

Some of the terms in this document have a technical meaning:

configuration bundle
:   a configuration covering multiple items, e.g. all the services
    required for detector simulation (`icarus_g4_services`);
    it is in the form of a table, and usually included via `@table::` prefix.

job configuration
:   the configuration (and its file) that describes a complete job, that can
    used directly with `lar` command; as opposed to a algorithm, module
    or service configuration, which needs to be included in a larger one
    to be used.
    
key
:   the name of a value in the configuration, either fully specified
    or just the name: `services.Geometry.GDML` and `GDML` are two examples.

prologue
:   a section which contains definitions to be used later on (or discarded).
    ```
    BEGIN_PROLOG
    standard_track_reco: {
      module_type: TrackReco
      ClusterTag:  pandora
    }
    END_PROLOG
    ```
    Note that all prologue sections must happen at the beginning of the file;
    after the first key is defined outside prologues, no more prologues are
    allowed.
    Definitions in prologues do not appear as such in the final configuration
    (for example, there would be no `standard_track_reco` table in the output
    of `config_dumper`).

sequence
:   a list of elements in brackets:
    ```
    [ 0.5, 0.866, 1.0, { type: induction scale: 1.03 } ]
    ```
    In this example the sequence is etherogeneous (it includes numbers
    and a table), but sequences in LArSoft configuration are almost always
    homogeneous.

table
:   a FHiCL section enclosed in braces, e.g.:
    ```
    {
      module_type: "RootOutput"
      fileName: "%ifb_%p-%tc.root"
    }
    ```


