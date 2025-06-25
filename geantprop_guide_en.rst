
..
.. Copyright (c) 2025 Hani Kimku <hkimku1@icecube.wisc.edu>
.. SPDX-License-Identifier: ISC
..
.. Permission to use, copy, modify, and/ordistribute this software for any
.. purpose with or without fee is hereby granted, provided that the above
.. copyright notice and this permission notice appear in all copies.
..
.. THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
.. WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
.. MERCHANTABILIITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY
.. SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
.. WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION
.. OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN
.. CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
..
..
.. @file geantprop_code.rst
.. @version $LastChangedRevision$
.. @date $Date$
.. @author Hani Kimku

========================================
I3Service Geant Propagator (geantprop)
========================================

Project Introduction
--------------------

`geantprop` is a `I3PropagatorService` implementation that performs particle propagation and interaction based on Geant4 within the Icetray framework. Derived from the Geant4 part of CLSim, it is designed to run Geant4 standalone. It supports the creation of MMC tracks for primary particles, the construction of an MCTree for secondary particles, and hybrid simulations through energy cutoffs.

Parameters
----------

When creating `I3GeantService`, the following parameters can be configured.

*   ``I3RandomServicePtr`` : A pointer to random number generator service. All probabilistic processes in Geant4 are determined using the random numbers provided by this service. (Use `I3MTRandomService`)

*   ``wlenBias`` : Applies a bias according to wavelength in `TrkCerenkov`. It adjusts the number of photons generated per step in the Geant4 simulation. It is obtained from the `icecube.clsim.traysegments.common.setupDetector` function.

*   ``mediumProperties`` : Defines the optical properties of the ice, such as the refractive index, absorption length, and scattering length in `TrkDetectorConstruction`. It is obtained from the `icecube.clsim.traysegments.common.setupDetector` function.

*   ``physicsListName`` : Determine which physical processes are included in the simulation. The default value, `"QGSP_BERT_EMV"`, means using the Quark-Gluon String model for high energies above 20GeV, the Bertini Cascade model for hadrons of energy less than 10GeV, and standard electromagnetic (EM) physics processes tuned for better CPU performance.

*   ``maxBetaChangePerStep`` : The maximum allowed change in a particle's velocity (β = v/c) in a single Geant4 step. This is applied in the `TrkCerenkov` class. Setting this value smaller allows for more precise calculation of trajectories for particles that curve in a magnetic field or lose energy rapidly, but it slows down the simulation.

*   ``maxNumPhotonsPerStep`` : Limits the maximum number of Cerenkov photons that can be generated in a single Geant4 step. This is applied in the `TrkCerenkov` class and serves as a safeguard to prevent very high-energy particles from using excessive memory or computation time by generating a vast number of photons in one step.

*   ``createMCTree`` : If this value is `true`, all secondary particles (electrons, positrons, muons, pions, etc.) generated in the Geant4 simulation are collected and added to the `I3MCTree`. If set to `false`, secondary particle information is not saved. As the particle's energy increases, the number of secondary particles grows, thus increasing memory usage.

*   ``binSize`` : The length of the segment in which the MMC track is divided.

*   ``createMMCTrackList`` : If `createMMCTrackList` is `true`, the path of the primary particle is added to the frame as an `I3MMCTrackList`. `I3MMCTrack` records the particle's entire path by dividing it into several straight segments of `binSize` length (in meters).

*   ``CrossoverEnergyEM`` : Energy cutoff value (in GeV) for electromagnetic (EM) interactions. If the energy of a particle being simulated exceeds this cutoff value, Geant4 will no longer propagate that particle. 

*   ``CrossoverEnergyHadron`` : Energy cutoff value (in GeV) for hadronic interactions. If the energy of a particle being simulated exceeds this cutoff value, Geant4 will no longer propagate that particle.

*   ``skipMuon`` : If this value is `true`, all muon (`MuPlus`, `MuMinus`) particles are immediately skipped without being propagated in Geant4. This is useful when using another propagator (e.g., `PROPOSAL`) to handle muons separately.

Usage & Examples
----------------

The following is a complete example of setting up `I3GeantService` and utilizing callback functions.

Basic Configuration
~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   from icecube import icetray, dataclasses, phys_services, geantprop
   from I3Tray import *
   from icecube.clsim.traysegments.common import setupDetector
   from icecube.sim_services import I3ParticleTypePropagatorServiceMap

   # 1. Get the random service and detector properties
   randomService = phys_services.I3MTRandomService([seed, streamnum, nstreams])

   # 2. Get the detector properties
   propparams = setupDetector(
    GCDFile=GCDFile,
    SimulateFlashers=False,
    IceModelLocation=expandvars("$I3_BUILD/ice-models/resources/models/ICEMODEL/spice_bfr-v2"),
    DisableTilt=False,
    UnWeightedPhotons=False,
    UnWeightedPhotonsScalingFactor=None,
    UseI3PropagatorService=False,
    UseGeant4 = True,
    CrossoverEnergyEM=None,
    CrossoverEnergyHadron=None,
    UseCascadeExtension=True,
    DOMOversizeFactor=1.0,
    UnshadowedFraction=1.2,
    DOMEfficiency=1,
    HoleIceParameterization=expandvars("$I3_BUILD/ice-models/resources/models/ANGSENS/angsens/as.flasher_p1_0.30_p2_-1"),
    WavelengthAcceptance=None,
    DOMRadius=0.16510 * icetray.I3Units.m,
    CableOrientation=None,
    IgnoreSubdetectors=["IceTop", "NotOpticalSensor"])
    
   wlenbias = propparams['WavelengthGenerationBias']
   mediumproperties = propparams['MediumProperties']
   
   # 3. Create a geantprop service instance
   propagator = geantprop.I3GeantService(
       randomService=randomService,
       wlenBias=wlenbias,
       mediumProperties=mediumproperties,
       physicsListName="QGSP_BERT_EMV",  # Physics list to use
       maxBetaChangePerStep=0.1,         # 10%
       maxNumPhotonsPerStep=200,
       createMCTree=True,                # Save secondary particles to MCTree
       binSize=10.0,                     # Create MMC track in 10-meter units
       createMMCTrackList=True,
       CrossoverEnergyEM=0.1,            # Stop EM cascades above 100 MeV
       CrossoverEnergyHadron=100.0,      # Stop hadronic cascades above 100 GeV
       skipMuon=True                     # Do not propagate muons with this service
   )

Utilizing Callback Functions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The callback mechanism allows users to directly intervene in the simulation process. This enables extensions without modifying the core code of `geantprop`. Callback functions are invoked whenever specific conditions are met during the simulation.

**StepCallback - Detailed Step-by-Step Analysis**
Called for each step in Geant4.

.. code-block:: python

   # Callback for analyzing energy loss distribution
   energy_loss_data = []
   
   def analyze_energy_loss(step):
       """Record and analyze the energy loss of each step"""
       if step.GetLength() > 0:
           de_dx = step.GetDepositedEnergy() / step.GetLength()  # dE/dx
           energy_loss_data.append({
               'position': (step.GetPosX(), step.GetPosY(), step.GetPosZ()),
               'energy_loss': step.GetDepositedEnergy(),
               'de_dx': de_dx,
               'num_photons': step.GetNumPhotons(),
               'beta': step.GetBeta()
           })
   
   propagator.SetStepCallback(analyze_energy_loss)

**SecondaryCallback - Particle Filtering and Analysis**
Called whenever a secondary particle is generated in Geant4. When this function returns `True`, the particle is killed and not added to the MCTree.

.. code-block:: python

   secondary_particles = []
   
   def advanced_secondary_filter(particle, pid, process_name):
       """Advanced filtering function called upon secondary particle creation"""
       
       # Record particle information 
       secondary_particles.append({
           'type': particle.GetTypeString(),
           'energy': particle.GetEnergy(),
           'position': (particle.GetX(), particle.GetY(), particle.GetZ()),
           'process': process_name,
           'parent_id': pid
       })
       
       # Selective tracking logic
       # 1. Only track electrons below 1 GeV
       if (particle.type == I3Particle.EPlus or 
           particle.type == I3Particle.EMinus):
           return particle.energy < 1.0 * I3Units.GeV  # True means kill
       
       # 2. Only track particles generated from specific processes
       if process_name in ["eBrem", "eIoni", "phot"]:
           return False  # keep tracking
       
       # 3. Ignore all muons (handled by another propagator)
       if (particle.type == I3Particle.MuPlus or 
           particle.type == I3Particle.MuMinus):
           return True  # kill
       
       return False  # Track all other particles by default
   
   propagator.SetSecondaryCallback(advanced_secondary_filter)

Registering with I3PropagatorModule
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   # Register in the propagator service map
   propagator_map = sim_services.I3ParticleTypePropagatorServiceMap()
   PT = dataclasses.I3Particle.ParticleType

   # Apply geantprop only to specific particle types (hybrid mode)
   em_particles = [PT.EMinus, PT.EPlus, PT.Gamma, PT.Brems, PT.DeltaE, PT.PairProd]
   hadron_particles = [PT.Neutron, PT.PPlus, PT.PMinus, PT.PiPlus, PT.PiMinus, PT.Pi0]
   
   for particle_type in em_particles + hadron_particles:
       propagator_map[particle_type] = propagator

   # Register with I3PropagatorModule
   tray.AddModule("I3PropagatorModule", "propagator",
                  PropagatorServices=propagator_map,
                  RandomService=randomService,
                  InputMCTreeName="I3MCTree_preGeant",
                  OutputMCTreeName="I3MCTree",
                  RNGStateName="I3MCTree_preGeant_RNGState")
                  
Geant4 Simulation Basic Structure
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*   **Run**: The largest unit of simulation. The detector geometry and applied physics laws do not change during a single Run. The entire process from the creation to the destruction of one `I3GeantService` object corresponds to one Run.

*   **Event**: The basic unit of simulation executed independently within a Run. It usually refers to the process from the creation to the annihilation of a single primary particle. In `geantprop`, one Event is created and executed each time the `Propagate` method is called.

*   **Track**: Represents a single particle path being tracked within the simulation world. An Event starts with one Primary Track, and this particle can generate numerous Secondary Tracks through interactions.

*   **Step**: The smallest unit composing a Track. It is the short segment from the point where a particle has a physical interaction to the next interaction point. Geant4 moves the particle's position in steps, and at the end of each step, it calculates physical processes such as energy loss, particle annihilation, and secondary particle generation.

Class Structure Overview
------------------------

``I3GeantService``
~~~~~~~~~~~~~~

The central manager that oversees all functions of `geantprop`. It inherits from `I3PropagatorService` to integrate with the Icetray framework.

*   **Singleton Pattern Implementation**: The `std::atomic<bool> thereCanBeOnlyOneGeant4` flag allows only one instance per process. Attempting to create a second instance results in a runtime error.

*   **Particle Filtering Logic**: The `ShouldSkip()` method pre-filters particles based on the following rules:
    - All neutrinos are automatically skipped.
    - If `skipMuon_` is true, muons are skipped.
    - EM and Hadronic particles with energy exceeding `CrossoverEnergyEM`/`CrossoverEnergyHadron` are skipped.

*   **Actual Propagation Execution**: The `Propagate()` method performs the following steps:
    1. Converts `I3Particle` to `G4ParticleGun`.
    2. Registers callback functions with each Action class.
    3. Executes a single event by calling `runManager_->BeamOn(1)`.
    4. Collects the simulation results as a vector of `I3Particle` and adds them to the MCTree / MMCtrackList before returning.

``TrkEventAction``
~~~~~~~~~~~~~~

A class that controls the simulation at the event level. It stores the `StepCallback` and `SecondaryCallback` registered by the user in the event information, making them accessible to other Action classes.

``TrkTrackingAction``
~~~~~~~~~~~~~~~~~

A class that manages the tracks of individual particles. It records the relationship between parent and child particles and also records the particle's path length.

``TrkSteppingAction``
~~~~~~~~~~~~~~~~~

A class responsible for step-by-step processing. It only processes the **primary particle** to which the Geant service is assigned.

*   **MMC Track Segment Creation**: When the accumulated path of the current track's particle exceeds `binSize`, it finalizes the current track and starts a new one.

*   **Energy Loss Calculation**: It calculates the amount of energy lost by recording the start and end energies for each segment.

``TrkStackingAction``
~~~~~~~~~~~~~~~~~

A class that passes newly created secondary particles to the callback.

``TrkDetectorConstruction``
~~~~~~~~~~~~~~~~~~~~~~~

A class that defines the geometry and materials of the simulation world.

*   **Complex Ice Model Construction**: Defines the optical properties of the medium, such as refractive index, absorption length, and scattering length of ice, via `mediumProperties`.

*   **3D Geometric Structure**: Models a realistic IceCube geometry including the World Volume, a rock layer, and an air layer.

``TrkOpticalPhysics``
~~~~~~~~~~~~~~~~~

A class that registers optical physics processes with the Geant4 engine.

*   **Cerenkov Process Registration** (`ConstructProcess`): Registers the Cerenkov process.

*   **Wavelength Bias Function Setting**: Sets the wavelength weights for importance sampling via `SetWlenBiasFunction()`.

``TrkCerenkov``
~~~~~~~~~~~

A class where the core optimization of Cerenkov radiation is implemented.

*   **Statistical Photon Calculation** (`PostStepDoIt`): Calculates the number of photons to be generated in the current step.

*   **SimStep Information Passing**: Packages the calculated number of photons and step information into an `I3SimStep` and passes it to the user callback.

``TrkPrimaryGeneratorAction``
~~~~~~~~~~~~~~~~~~~~~~~~~~

A class that injects the initial particle at the starting point of the simulation.

``TrkUserEventInformation``
~~~~~~~~~~~~~~~~~~~~~~~

A container class that stores per-event state information.

*   **Callback Function Storage**: Stores the `StepCallback` and `SecondaryCallback` registered by the user.

*   **Medium Information**: Stores `maxRefractiveIndex` to provide necessary information for Cerenkov calculations.

``I3ParticleG4ParticleConverter``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Handles the two-way conversion between `I3Particle` and Geant4 data formats.

*   **Particle Gun Setup** (`SetParticleGun`): A class that injects the initial particle.

*   **PDG Code Conversion**: Converts IceCube particle types to Geant4's PDG encoding.

*   **Unit Conversion**: Handles the conversion between the IceCube unit system (`I3Units`) and the Geant4 unit system (`CLHEP`).

``TrkUISessionToQueue``
~~~~~~~~~~~~~~~~~~~

A bridge class that connects Geant4 messages to the IceCube logging system.

*   **Message Queuing**: Stores all output from Geant4 (`G4cout`, `G4cerr`) in a queue.

*   **Log Level Classification**: Forwards error messages as `log_warn()` and normal messages as `log_debug()`.

*   **Thread Safety**: Ensures thread safety through asynchronous message processing via the queue.

Tests
-----

To ensure the accuracy and stability of `geantprop`, unit tests are included via the `resources/test/test_service.py` Python script. This test propagates a muon of about 2 GeV in a simple virtual geometry with Geant4 and then verifies the following key points. This guarantees that the service operates correctly both physically and technically.

1.  **Step Generation**: Verifies that at least one step is recorded during the simulation process. This means the particle has actually propagated through the medium.
2.  **Secondary Generation**: Verifies that secondary particles are generated as a result of the primary particle's interaction. This shows that the physics list is correctly applied and interactions occur normally.
3.  **MMCTrack Division**: Verifies that the `I3MMCTrack` is correctly divided into several segments according to the set `binSize`, thus validating the data format's correctness.
4.  **Energy Conservation**: Verifies that the sum of the primary particle's initial energy, the remaining energy after propagation, and the total energy loss recorded in the `MMCTrack` are consistent. This ensures that the most fundamental law of physics (energy conservation) is satisfied in the simulation. 
