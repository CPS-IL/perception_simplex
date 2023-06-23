# Perception Simplex

This repository provides instructions for software-in-the-loop simulation.
For dataset evaluation and detectability plots refer to [Verifiable Obstacle Detection](https://github.com/CPS-IL/verifiable-OD).


## External Resources

The software-in-the-loop simulation setup for Perception Simplex is based on the LGSVL 2020.06 + Apollo 5.0 simulation environments.
Follow [this setup guide](https://www.svlsimulator.com/docs/archive/2020.06/apollo5-0-instructions/) to ensure the base setup works.
Familiarity with this setup is also helpful to understand the steps in this evaluation as there are many instances of switching between the two simulators.

LGSVL has decided to [sunset its simulator](https://www.svlsimulator.com/news/2022-01-20-svl-simulator-sunset).
LGSVL and therefore this setup depends on online resources that may become unavailable.




## Hardware

<p align="center" >
  <img src="images\hardware-setup.png" width="400"/>
</p>

The system setup used for this evaluation is shown above.
Any different setup that supports the baseline LGSVL + Apollo setup also works. 


## Software Setup - Environment Simulation
The environment and vehicle simulation can reside on the same machine, however, to isolate computational interference we prefer to keep them separated.
The following steps are for the Environment Simulation PC.

### Simulator
Download [LGSVL 2020.06](https://github.com/lgsvl/simulator/releases/tag/2020.06). This contains `simulator.exe` to start the environment simulator.
We will use the `SingleLaneRoad` map and the Lincoln 2017 Cyber-RT vehicle.
Create copies of this vehicle and modify their name and configuration accordingly.
The vehicle name and configuration source are as follows.

* LincolnMission - [json](https://github.com/CPS-IL/apollo/lgsvl_pkgs/lgsvl_vehicles/lincoln_mission.json)
* LincolnSafety - [json](https://github.com/CPS-IL/apollo/lgsvl_pkgs/lgsvl_vehicles/lincoln_safety.json)

The main point of difference between these two configurations is the source of the Apollo Car Control, between `/apollo/control` or `/apollo/safety_layer/decision/control`.

Finally, create an API-controlled simulation and run it. This allows the simulation to be controlled by the PythonAPI control interface.

### Control Interface
LGSVL provides a Python interface to control the simulator. 
Clone our custom Python API repository https://github.com/CPS-IL/lgsvl_pythonapi on the environment simulation PC.
This includes the evaluation scenarios, result plotting scripts, raw results, and final plots.


```
git clone https://github.com/CPS-IL/lgsvl_pythonapi.git
cd lgsvl_pythonapi
pip3 install --user .
pip3 install matplotlib
```

In [ps_scenario\sudden_stop.py](https://github.com/CPS-IL/lgsvl_pythonapi/blob/master/ps_scenario/sudden_stop.py), modify `APOLLO_IP` to the machine running Apollo.


## Software Setup - AV Simulation

AV simulation is based on Apollo 5.0.
It integrates the safety layer of Perception Simplex with the Apollo pipeline.

The Apollo fork https://github.com/CPS-IL/apollo contains other repositories as submodules, directly or nested

* https://github.com/CPS-IL/lgsvl_msgs - Forked from https://github.com/lgsvl/lgsvl_msgs, supports communication between LGSVL and Apollo.
* https://github.com/CPS-IL/safety_layer_module - Adds the safety layer as a Cyber-RT module to Apollo.
* https://github.com/CPS-IL/depth_clustering - Forked from https://github.com/PRBonn/depth_clustering, enables the use of Depth Clustering for obstacle detection within the safety layer.
* https://github.com/CPS-IL/verifiable_obstacle_detection - Provides obstacle projection and comparison logic, using the requirements set forth in [prior work](https://ieeexplore.ieee.org/abstract/document/9978967), to determine obstacle existence detection faults.


```
apt install build-essential cmake tar wget
git clone https://github.com/CPS-IL/apollo.git --recurse-submodules
cd apollo
./docker/scripts/dev_start.sh
./docker/scripts/dev_into.sh  # This gets us into the Apollo Docker, rest of the commands are to be executed inside this docker.

# Build library for detection projection and comparison
cd /apollo/modules/safety_layer/lib/verifiable_obstacle_detection/scripts/verifiable_obstacle_detection
./setup.bash --system
cd /apollo/modules/safety_layer/lib/verifiable_obstacle_detection/build/amd64/verifiable_obstacle_detection/release
make

# Build the Depth Clustering Algorithm as a library
cd /apollo/modules/safety_layer/lib/depth_clustering/scripts/depth_clustering
./setup.bash --system
cd /apollo/modules/safety_layer/lib/depth_clustering/build/amd64/depth_clustering/release
make

# Build Apollo, same as https://www.svlsimulator.com/docs/archive/2020.06/apollo5-0-instructions/
cd /apollo
./apollo.sh build_gpu
```


## Running Tests
The steps to run the evaluation are largely constant, with different configurations for different specific evaluations.
The main points of differences from the [base guide](https://www.svlsimulator.com/docs/archive/2020.06/apollo5-0-instructions/) are:

* Launch the safety layer module after the bridge to LGSVL is up, from inside the Apollo docker, this requires a separate terminal since `bridge.sh` will capture the terminal.

```
cd apollo
./docker/scripts/dev_into.sh  # Assuming docker is already running
cd /apollo
cyber_launch start modules/safety_layer/launch/safety_layer.launch
```

* LGSVL Simulator is controlled via scenario scripts, running on the environment simulator machine (not Apollo).
```
python.exe lgsvl_pythonapi/ps_scenario/sudden_stop.py <args>
```

Each step in the procedure is annotated to signify where it should be run.
* **AD**: Apollo Docker terminal on AV Simulation PC, at current path `/apollo`
* **DV**: Apollo's DreamView web interface, launched by `bootstrap.sh`, by default `http://localhost:8888`, where localhost is the machine running Apollo.
* **ES**: Environment Simulation PC.
* **PL**: Python API for LGSVL, running the scenario scripts, on the Environment Simulation PC.

### Procedure

1. **AD** Exit any running `bridge.sh`, clean up any older logs and restart Apollo `bootstap.sh stop && ./scripts/clear_log.sh && bootstrap.sh && bridge.sh`, 

2. **ES** Start LGSVL and through the browser interface run the API-only simulation.

3. **DV** Select  *Lincoln2017MKZ* vehicle and *SingleLaneRoad* map.

4. **DV** Start Localization, Transform, Perception, Traffic Light, Planning, Prediction, Routing, and Control modules.

5. **AD** In a different Apollo Docker terminal, launch Safety layer module `cyber_launch start modules/safety_layer/launch/safety_layer.launch`

6. **PL** Launch the scenario script. Wait till this prompt appears: `Set up the route in Apollo Dreamview and then press enter here`

7. **DV** From *Route Editing* tab, set the destination to the end of the road and click *Submit Route*.

8. **PL** Press Enter, if another prompt appears follow that, otherwise wait for results to be generated.

9. **DV** Vehicle motion can be observed in DreamView. We prefer to run LGSVL headless to improve its performance.

10. **PL** When finished, collect results, move to `lgsvl_pythonapi/ps_scenario/results`, and rename. Raw results are provided here to show expected file names.

11. **PL** Raw results can be converted to plots via scripts in `lgsvl_pythonapi/ps_scenario/plots`

### Configurations

There are two types of configurations, the first is based on the system control pipeline and the second is based on overall evaluation goals.

#### System Control Configurations

* Mission Only: Apollo 5.0 control the vehicle, safety layer is not involved

  * **PL** Use argument `--vehicle LincolnMission` when launching scenario (Step 6)


* Mission Crash: Synthetic best case for safety, i.e., start braking as soon as the scenario starts

  * **AD** Checkout safety layer module to the tag mission_crash : `cd modules/safety_layer && git checkout mission_crash`
  * **AD** Build Apollo `cd /apollo && ./apollo.sh build_gpu`
  * **PL** Use argument `--vehicle LincolnSafety` when launching scenario (Step 6)

* Fault Injected: Instrumented evaluation target, i.e. obstacle existence detection fault injected

  * **AD** Checkout safety layer module to the tag mission_crash : `cd modules/safety_layer && git checkout fault_inject`
  * **AD** Build Apollo `cd /apollo && ./apollo.sh build_gpu`
  * **PL** Use argument `--vehicle LincolnSafety` when launching scenario (Step 6)

* Perception Simplex: the system without any instrumentation.

  * **AD** Checkout safety layer module to the tag mission_crash : `cd modules/safety_layer && git checkout ps`
  * **AD** Build Apollo `cd /apollo && ./apollo.sh build_gpu`
  * **PL** Use argument `--vehicle LincolnSafety` when launching scenario (Step 6)


#### Evaluation Target Setups

* Collision Avoidance

  * **PL** Use `ps_scenario\sudden_stop.py` and provide argument `--`infront`, this sets the obstacle in the path of the ego vehicle.
  This evaluation takes a few hours to finish and generates a `res.json`. `--scenario` argument appends to this result filename.
  The json records simulation results for each distance and velocity combination. Simulation results are one of *Collision, Safe Stop*, and *Safe Pass*.

* Performance

  * **AD** Checkout safety layer module to the tag mission_crash : `cd modules/safety_layer && git checkout speed`
  * **AD** Build Apollo `cd /apollo && ./apollo.sh build_gpu`
  * **PL** Run `obstacle_in_other_lane.py` scenario script. Do not provide `--infront` argument to it.
  * **AD** Results are recorded to `/apollo/data/log/speed.log.txt`

* Computational Latency
  * **AD** Switch both Apollo and Safety Layer modules to timing branch. `cd /apollo &&  git checkout timing && cd /apollo/modules/safety_layer &&  git checkout timing`
  * **AD** Build Apollo `cd /apollo && ./apollo.sh build_gpu`
  * **AD**  Results are generated as log files, with component names, for example for safety layer decision logic it is `/apollo/data/log/safety_layer.decision_timing.log.txt`

## Contact

[Ayoosh Bansal](mailto:ayooshb2@illinois.edu)
