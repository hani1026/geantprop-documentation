..
.. Copyright (c) 2025 Hani Kimku <hkimku1@icecube.wisc.edu>
.. SPDX-License-Identifier: ISC
..
.. @file geantprop_guide_ko.rst
.. @author Hani Kimku

========================================
I3Service Geant Propagator (geantprop)
========================================

.. contents:: 목차
   :local:

프로젝트 소개
------------

`geantprop`은 IceCube 실험의 Icetray 프레임워크에서 Geant4 기반 입자 전파를 수행하는 전문화된 `I3PropagatorService` 구현체입니다. CLSim의 Geant4 부분에서 파생되어 Geant4 시뮬레이션에 집중하도록 설계되었습니다. 주 입자의 MMC 트랙 생성, 2차 입자의 MCTree 구축, 에너지 컷오프를 통한 하이브리드 시뮬레이션을 지원합니다.

파라미터 (Parameters)
----------------------

`I3GeantService`를 생성할 때 다음과 같은 파라미터들을 설정할 수 있습니다.

*   ``I3RandomServicePtr``
    Icetray의 중앙 난수 생성기(random number generator) 서비스에 대한 포인터입니다. Geant4의 모든 확률적 과정(상호작용 발생 여부, 감쇠 길이 등)은 이 서비스가 제공하는 난수를 사용하여 결정됩니다. 이를 통해 전체 시뮬레이션의 재현성을 보장합니다.

*   ``wlenBias`` (`I3CLSimFunctionConstPtr`)
    `TrkCerenkov`에서 wavelength에 따른 bias를 적용합니다. Geant4 시뮬레이션의 step 단위에서 생성되는 Number of photon 값을 조절합니다. 

*   ``mediumProperties`` (`I3CLSimMediumPropertiesConstPtr`)
    얼음의 굴절률, 흡수 길이, 산란 길이 등 매질의 광학적 특성을 정의하는 객체입니다. 이 정보는 `TrkDetectorConstruction` 클래스에서 Geant4의 물질 및 광학 속성을 설정하는 데 사용됩니다.

*   ``physicsListName`` (`std::string`)
    Geant4의 물리 리스트(Physics List) 이름입니다. 물리 리스트는 어떤 물리 과정을 시뮬레이션에 포함할지를 결정합니다. 기본 값인 `"QGSP_BERT_EMV"`는 고에너지 강입자 상호작용에 Quark-Gluon String 모델을, 저에너지 영역에 Bertini Cascade 모델을, 그리고 표준 전자기(EM) 물리 과정을 사용하는 것을 의미합니다. 

*   ``maxBetaChangePerStep`` (`double`)
    Geant4의 한 스텝에서 허용되는 입자의 속도(β = v/c) 변화량의 최댓값입니다. `TrkCerenkov` 클래스에서 적용되며 이 값을 작게 설정하면 자기장 속에서 휘거나 에너지를 빠르게 잃는 입자의 궤적을 더 정밀하게 계산하지만 시뮬레이션 속도는 느려집니다.

*   ``maxNumPhotonsPerStep`` (`uint32_t`)
    Geant4의 한 스텝에서 생성될 수 있는 체렌코프 광자의 최대 개수를 제한합니다. `TrkCerenkov` 클래스에서 적용되며 매우 높은 에너지의 입자가 한 스텝에 수많은 광자를 생성하여 메모리나 계산 시간을 과도하게 사용하는 것을 방지하기 위한 안전장치입니다.

*   ``createMCTree`` (`bool`)
    이 값이 `true`이면, Geant4 시뮬레이션에서 생성된 모든 2차 입자들(전자, 양전자, 뮤온, 파이온 등)이 수집되어 `I3MCTree`에 추가됩니다. `false`로 설정하면 2차 입자 정보는 저장되지 않습니다. 입자의 에너지가 클수록 2차 입자의 수가 많아지므로 메모리 사용량이 증가합니다.

*   ``binSize`` (`double`)
*   ``createMMCTrackList`` (`bool`)
    `createMMCTrackList`가 `true`일 경우, 주 입자(primary particle)의 경로를 `I3MMCTrackList` 형식으로 프레임에 추가합니다. `I3MMCTrack`은 입자의 전체 경로를 `binSize` (미터 단위) 길이의 여러 직선 구간으로 나누어 기록하는 방식입니다. 

*   ``CrossoverEnergyEM`` (`double`)
*   ``CrossoverEnergyHadron`` (`double`)
    전자기(EM) 또는 강입자(Hadronic)에 대한 에너지 컷오프 값(GeV 단위)입니다. 시뮬레이션 중인 입자의 에너지가 이 컷오프 값보다 크면, Geant4는 해당 입자를 전파시키지 않습니다. 

*   ``skipMuon`` (`bool`)
    이 값이 `true`이면, 모든 뮤온(`MuPlus`, `MuMinus`) 입자는 Geant4에서 전파되지 않고 즉시 건너뜁니다. 다른 전파기(예: `PROPOSAL`)를 사용하여 뮤온을 별도로 처리 할 때 유용합니다.

사용법 및 코드 예시 (Usage & Examples)
--------------------------------------

다음은 `I3GeantService`를 설정하고 콜백 함수를 활용하는 완전한 예시입니다.

기본 설정
~~~~~~~~~

.. code-block:: python

   from I3Tray import I3Tray
   from icecube import icetray, dataclasses, dataio, phys_services, sim_services
   from icecube import geantprop

   # 1. geantprop 서비스 인스턴스 생성
   propagator = geantprop.I3GeantService(
       randomService=randomService,
       wlenBias=wlenbias,
       mediumProperties=mediumproperties,
       physicsListName="QGSP_BERT_EMV",  # 사용할 물리 리스트
       maxBetaChangePerStep=0.1,         # 10%
       maxNumPhotonsPerStep=200,
       createMCTree=True,                # 2차 입자들을 MCTree에 저장
       binSize=10.0,                     # 10미터 단위로 MMC 트랙 생성
       createMMCTrackList=True,
       CrossoverEnergyEM=0.1,            # 100 MeV 이상의 EM 캐스케이드는 중단
       CrossoverEnergyHadron=100.0,      # 100 GeV 이상의 강입자 캐스케이드는 중단
       skipMuon=True                     # 뮤온은 이 서비스로 전파하지 않음
   )

콜백 함수 활용
~~~~~~~~~~~~~~

사용자가 시뮬레이션 과정에 직접 개입할 수 있도록 하는 콜백(Callback) 함수 메커니즘입니다. 이를 통해 `geantprop`의 핵심 코드를 수정하지 않고 확장이 가능합니다. 콜백 함수는 시뮬레이션 중 특정 조건이 만족될 때마다 호출됩니다.

**StepCallback - 스텝별 상세 분석**
Geant4의 각 스텝마다 호출됩니다.

.. code-block:: python

   # 에너지 손실 분포 분석을 위한 콜백
   energy_loss_data = []
   
   def analyze_energy_loss(step):
       """각 스텝의 에너지 손실을 기록하고 분석"""
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

**SecondaryCallback - 입자 필터링 및 분석**
Geant4의 2차 입자가 생성될 때마다 호출됩니다.

.. code-block:: python

   secondary_particles = []
   
   def advanced_secondary_filter(particle, pid, process_name):
       """2차 입자 생성 시 호출되는 고급 필터링 함수"""
       
       # 입자 정보 기록
       secondary_particles.append({
           'type': particle.GetTypeString(),
           'energy': particle.GetEnergy(),
           'position': (particle.GetX(), particle.GetY(), particle.GetZ()),
           'process': process_name,
           'parent_id': pid
       })
       
       # 선택적 추적 로직
       # 1. 1 GeV 이상의 전자만 추적
       if (particle.type == I3Particle.EPlus or 
           particle.type == I3Particle.EMinus):
           return particle.energy < 1.0 * I3Units.GeV  # True면 kill
       
       # 2. 특정 프로세스에서 생성된 입자만 추적
       if process_name in ["eBrem", "eIoni", "phot"]:
           return False  # keep tracking
       
       # 3. 뮤온은 모두 무시 (다른 전파기에서 처리)
       if (particle.type == I3Particle.MuPlus or 
           particle.type == I3Particle.MuMinus):
           return True  # kill
       
       return False  # 기본적으로 모든 입자 추적
   
   propagator.SetSecondaryCallback(advanced_secondary_filter)

I3PropagatorModule 등록
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   # 전파기 서비스 맵에 등록
   propagator_map = sim_services.I3ParticleTypePropagatorServiceMap()
   PT = dataclasses.I3Particle.ParticleType

   # 특정 입자 타입에만 geantprop 적용 (하이브리드 모드)
   em_particles = [PT.EMinus, PT.EPlus, PT.Gamma, PT.Brems, PT.DeltaE, PT.PairProd]
   hadron_particles = [PT.Neutron, PT.PPlus, PT.PMinus, PT.PiPlus, PT.PiMinus, PT.Pi0]
   
   for particle_type in em_particles + hadron_particles:
       propagator_map[particle_type] = propagator

   # I3PropagatorModule에 등록
   tray.AddModule("I3PropagatorModule", "propagator",
                  PropagatorServices=propagator_map,
                  RandomService=randomService,
                  InputMCTreeName="I3MCTree_preGeant",
                  OutputMCTreeName="I3MCTree",
                  RNGStateName="I3MCTree_preGeant_RNGState")
                  
Geant4 시뮬레이션 기본 구조
~~~~~~~~~~~~~~~~~~~~~~~~~~

*   **Run**: 시뮬레이션의 가장 큰 단위입니다. 하나의 Run 동안에는 검출기 기하 구조나 적용되는 물리 법칙이 변하지 않습니다. `I3GeantService` 객체 하나가 생성되어 소멸될 때까지의 전체 과정이 하나의 Run에 해당합니다.

*   **Event**: Run 내에서 독립적으로 실행되는 시뮬레이션의 기본 단위로, 보통 하나의 초기 입자(primary particle)가 생성되어 소멸될 때까지의 과정을 의미합니다. `geantprop`에서는 `Propagate` 메소드가 한 번 호출될 때마다 하나의 Event가 생성되고 실행됩니다.

*   **Track**: 시뮬레이션 세계 내에서 추적되는 하나의 입자 경로를 의미합니다. 하나의 Event는 하나의 초기 입자(Primary Track)로 시작하며, 이 입자가 상호작용하면서 수많은 2차 입자(Secondary Tracks)들을 생성할 수 있습니다.

*   **Step**: Track을 구성하는 가장 작은 단위입니다. 입자가 물리적 상호작용을 일으키는 지점부터 다음 상호작용 지점까지의 짧은 구간을 의미합니다. Geant4는 스텝 단위로 입자의 위치를 옮기고, 각 스텝의 끝에서 에너지 손실, 입자 소멸, 2차 입자 생성 등 물리 프로세스를 계산합니다.

클래스 구조 개요
~~~~~~~~~~~~~~~~

`geantprop`는 Geant4의 표준 인터페이스("User Action" 등)를 상속받아 구현한 여러 클래스들로 구성됩니다. 이 클래스들은 크게 **최상위 서비스**, **시뮬레이션 제어**, **시뮬레이션 환경**, **데이터 처리 및 유틸리티**으로 나눌 수 있습니다.

### 최상위 서비스 (Top-Level Service)

**I3GeantService** 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`geantprop`의 모든 기능을 총괄하는 중앙 관리자입니다. `I3PropagatorService`를 상속받아 Icetray 프레임워크와 통합됩니다.

*   **싱글톤 패턴 구현**: `std::atomic<bool> thereCanBeOnlyOneGeant4` 플래그를 통해 프로세스당 하나의 인스턴스만 허용합니다. 두 번째 인스턴스 생성 시도 시 런타임 에러가 발생합니다.

*   **입자 필터링 로직**: `ShouldSkip()` 메소드에서 다음 규칙에 따라 입자를 사전 필터링합니다:
    - 모든 중성미자는 자동으로 스킵
    - `skipMuon_`이 true면 뮤온 스킵
    - 에너지가 `CrossoverEnergyEM`/`CrossoverEnergyHadron`을 초과하는 EM, Hadronic 입자 스킵

*   **실제 전파 실행**: `Propagate()` 메소드에서 다음 단계를 수행합니다:
    1. `I3Particle`을 `G4ParticleGun`으로 변환
    2. 콜백 함수들을 각 Action 클래스에 등록
    3. `runManager_->BeamOn(1)` 호출로 단일 이벤트 실행
    4. 시뮬레이션 결과를 `I3Particle` 벡터로 수집하여 MCTree / MMCtrackList에 추가하고 반환

### 시뮬레이션 제어 (User Actions)

Geant4 시뮬레이션의 주 흐름(이벤트, 트랙, 스텝)을 직접 제어하는 클래스들입니다.

**TrkEventAction**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

이벤트 수준에서 시뮬레이션을 제어하는 클래스입니다. 사용자가 등록한 `StepCallback`과 `SecondaryCallback`을 이벤트 정보에 저장하여 다른 Action 클래스들이 접근할 수 있도록 합니다.

**TrkTrackingAction**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

개별 입자의 트랙을 관리하는 클래스입니다. Parent particle과 Child particle의 관계를 기록합니다. 또한 입자의 경로 길이를 기록합니다.

**TrkSteppingAction**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

스텝별 처리를 담당하는 클래스입니다. Geant service가 할당된 **주 입자(Primary particle)만** 처리합니다. 

*   **MMC 트랙 세그먼트 생성**: 현재 트랙의 입자의 누적 경로가 binSize를 초과하면 현재 트랙을 완료하고 새로운 트랙을 시작합니다.

*   **에너지 손실 계산**: 각 세그먼트별로 시작 에너지와 종료 에너지를 기록하여 에너지 손실량을 계산합니다.

**TrkStackingAction**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

새로 생성된 2차 입자를 콜백에 전달하는 클래스입니다.

### 시뮬레이션 환경 (Physics & Geometry)

**TrkDetectorConstruction**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

시뮬레이션 세계의 기하학과 물질을 정의하는 클래스입니다.

*   **복합 얼음 모델 구축**: mediumProperties를 통해 얼음의 굴절률, 흡수 길이, 산란 길이 등 매질의 광학적 특성을 정의합니다. 

*   **3차원 기하학 구조**: 세계 부피(World Volume), 암석층, 공기층을 포함한 현실적인 IceCube 기하학을 모델링합니다.

**TrkOpticalPhysics**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

광학 물리 프로세스를 Geant4 엔진에 등록하는 클래스입니다.

*   **체렌코프 프로세스 등록** (`ConstructProcess`): 체렌코프 프로세스를 등록합니다.

*   **파장 편향 함수 설정**: `SetWlenBiasFunction()`을 통해 중요도 샘플링을 위한 파장 가중치를 설정합니다. 

**TrkCerenkov**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

체렌코프 방사의 핵심 최적화가 구현된 클래스입니다. 

*   **통계적 광자 계산** (`PostStepDoIt`): 현재 스텝에서 생성될 광자의 개수를 계산합니다.

*   **SimStep 정보 전달**: 계산된 광자 수와 스텝 정보를 `I3SimStep`으로 패키징하여 사용자 콜백에 전달합니다. 

### 데이터 처리 및 유틸리티

위 클래스들이 원활하게 동작하도록 돕는 보조 클래스들입니다.

**TrkPrimaryGeneratorAction**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

시뮬레이션의 시작점에서 초기 입자를 주입하는 클래스입니다. 

**TrkUserEventInformation**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

이벤트별 상태 정보를 저장하는 컨테이너 클래스입니다.

*   **콜백 함수 저장**: 사용자가 등롱한 `StepCallback`과 `SecondaryCallback`을 저장합니다. 

*   **매질 정보**: `maxRefractiveIndex`를 저장하여 체렌코프 계산에 필요한 정보를 제공합니다.


**I3ParticleG4ParticleConverter**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`I3Particle`과 Geant4 데이터 형식 간의 양방향 변환을 담당합니다.

*   **입자 총 설정** (`SetParticleGun`): 초기 입자를 주입하는 클래스입니다.

*   **PDG 코드 변환**: IceCube의 입자 타입을 Geant4의 PDG 인코딩으로 변환합니다. 

*   **단위 변환**: IceCube 단위계(`I3Units`)와 Geant4 단위계(`CLHEP`) 간의 변환을 처리합니다. 

**TrkUISessionToQueue**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Geant4 메시지를 IceCube 로깅 시스템으로 연결하는 브릿지 클래스입니다. 

*   **메시지 큐잉**: Geant4의 모든 출력(`G4cout`, `G4cerr`)을 큐에 저장합니다.

*   **로그 레벨 분류**: 에러 메시지는 `log_warn()`으로, 일반 메시지는 `log_debug()`로 전달합니다.

*   **스레드 안전**: 큐를 통한 비동기 메시지 처리로 스레드 안전성을 보장합니다.

테스트 (Tests)
--------------

`geantprop`의 정확성과 안정성을 보장하기 위해 `resources/test/test_service.py` 파이썬 스크립트를 통한 단위 테스트가 포함되어 있습니다. 이 테스트는 약 2 GeV의 뮤온을 가상의 단순한 기하학 구조 내에서 Geant4로 전파시킨 후, 다음과 같은 핵심 사항들을 검증합니다. 이를 통해 서비스가 물리적으로, 그리고 기술적으로 올바르게 동작함을 보장합니다.

#. **스텝 생성 (Step Generation)**: 시뮬레이션 과정에서 최소 하나 이상의 스텝이 기록되는지를 확인합니다. 이는 입자가 실제로 매질 내에서 전파되었음을 의미합니다.
#. **2차 입자 생성 (Secondary Generation)**: 주 입자의 상호작용 결과로 2차 입자가 생성되는지를 검증합니다. 이는 물리 리스트가 올바르게 적용되고 상호작용이 정상적으로 일어남을 보여줍니다.
#. **MMCTrack 분할 (MMCTrack Division)**: `I3MMCTrack`이 설정된 `binSize`에 따라 올바르게 여러 구간으로 나뉘어 생성되는지를 확인하여 데이터 형식의 정확성을 검증합니다.
#. **에너지 보존 (Energy Conservation)**: 주 입자의 초기 에너지와, 전파 후 남은 에너지 및 `MMCTrack`에 기록된 총 손실 에너지의 합이 일치하는지를 검증합니다. 이는 시뮬레이션의 가장 기본적인 물리 법칙(에너지 보존)이 만족됨을 보장합니다. 