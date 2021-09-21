# SENTIENT
Toolchain for abstraction and scheduling of event-triggered controllers.

##Installation

---------------------------------------

### Prerequisites
This tool requires Python 3.8 to function correctly, so make sure it is installed beforehand. 

First, download or clone the project to a local directory.
Secondly, download and copy the third-party tools, dReal, dReach, and Flow* in the 'third-party' folder in the root project
directory if you want to install them. You can download them from below location:

    dReal - https://github.com/dreal/dreal4
    dReach - https://github.com/dreal/dreal3/releases/tag/v3.16.06.02
    Flow* - https://flowstar.org/dowloads/

 - Make sure to get Flow* version v2.0.0, and run ``make`` in its directory. 
 - Build dReal4 by following the instructions provided in its ``README.md``
 - In the file ``/dreal-3.16.06.02-*/bin/dReach``, replace the following line
    ``` bash
   SCRIPT_PATHNAME=`python -c "import os,sys; print os.path.realpath(\"$0\")"`
    ```
   By:
   ``` bash
   SCRIPT_PATHNAME=`python2 -c "import os,sys; print(os.path.realpath(\"$0\"))"`
    ```
   To make sure Python 2 is used here.
   
 Finally, add the paths to ``config.py``, e.g.
   ```python
   dreal_path = '<path>/sentient/third-party/dreal4/bazel-bin/dreal/dreal'
   dreach_path = '<path>/sentient/third-party/dReal-3.16.06.02-*/bin/dReach'
   flowstar_path = '<path>/sentient/third-party/flowstar-2.0.0/flowstar-2.0.0/flowstar'
   ```

   To be able to create schedulers with timed automata, [UPPAAL Stratego](https://people.cs.aau.dk/~marius/stratego/) also needs to be installed. Then the path to ``verifyta`` has to be set in ``config.py`` as well:
   ```python
   VERIFYTA = '<path>/uppaal64-4.1.20-stratego-7/bin-Linux/verifyta'
   ```
### Conda

To install with conda, make sure a C/C++ compiler (e.g. ``gcc``/``g++``) and GMP is installed. Then simply run
``` shell
   $ conda env create --file environment.yml
   $ conda activate sentient
   $ conda install conda-build
   $ conda develop .
```
where the last two lines are necessary (temporarily, until proper conda package has been built) to import ``SENTIENT`` in a script.

### Pip

Installation with pip is a bit more involved, as more packages will have to be build from source (depending on your OS).
First run one of the scripts in ``scripts/`` (corresponding to your OS) to install the dependencies. Then run
``` shell
   $ pip install wheel
   $ pip install .
```
If you want to use the CUDD bindings for [dd](https://github.com/tulip-control/dd), then also run:
``` shell
   $ pip install --force-reinstall -r \
         <(echo "dd --install-option='--fetch' --install-option='--cudd'")
```

# Quickstart

***
For a quickstart, the command line tool `etc2pta.py` can be used. 
It can generate traffic models for linear PETC systems, and for nonlinear 
ETC systems. The command line interface is used as follows:

```shell
$ python etc2pta.py <systemtype> <inputfile> [options]
```

Systemtype is either `linear' or 'nonlinear', from which the corresponding abstraction 
algorithm is chosen. Input file is a file containing the details for the ETC system. 
The contents of this file depend on the abstraction algorithm. 
The following shows an example input file for linear PETC systems: ![example_file](docs/linear_ifac2020.jpg) 


For example, running the example `linear_ifac.txt` and outputting is done as:
```shell
$ python etc2pta.py linear examples/linear_ifac.txt --output_file=ex.json
```


-------------------------
Full documentation see [Documentation](docs/build/Documentation.html)

# Examples


## Traffic Model Examples
Here we discuss examples of how to construct traffic models for linear PETC and nonlinear ETC systems.

Alternative to these approaches, the functions ``sentient.util.construct_linearPETC_traffic_from_file`` and ``sentient.util.construct_nonlinearETC_traffic_from_file`` can be used to construct the traffic models from a file.

### Linear PETC
In this example, a traffic model of a linear Periodic Event Triggered Control system will be generated.
In the tool, a PETC system is then defined as follows:

``` python
    # Define LTI system matrices
    A = np.array([[1.38, -0.208, 6.715, -5.676], [-0.581, -4.29, 0, 0.675], [1.067, 4.273, -6.654, 5.893], [0.048, 4.273, 1.343, -2.104]])
    B = np.array([[0, 0],[5.679, 0], [1.136, 3.146],[1.136, 0]])
    K = np.array([[0.518, -1.973, -0.448, -2.1356], [-3.812, -0.0231, -2.7961, 1.671]])

    # PETC parameters
    h = 0.01
    kmax = 20
    sigma = 0.01

    # Triggering condition
    Qtrigger = np.block([[(1-sigma)*np.eye(4), -np.eye(4)], [-np.eye(4), np.eye(4)]])

    # Construct object representing the PETC system
    import sentient.Systems as systems

    plant = systems.LinearPlant(A, B)
    controller = systems.LinearController(K, h)
    trigger = systems.LinearQuadraticPETC(plant, controller, kmax=kmax, Qbar=Qtrigger)
```

Then a traffic model is generated as follows. More arguments can be specified as described in :class:`sentient.Abstractions.TrafficModelLinearPETC`.

``` python
    import sentient.Abstractions as abstr

    traffic = abstr.TrafficModelLinearPETC(trigger)
    regions, transitions = traffic.create_abstraction()
    # Returns: {(2,): 2, (5,): 5, (4,): 4, (1,): 1, (3,): 3}, {((2,), 1): {(1,), (2,), (3,)}, ((2,), 2): {(2,), (5,), (4,), (1,), (3,)}, ...}
```
Alternatively, the regions/transitions can also be accessed separately:

``` python
    regions = traffic.regions
    # Returns: {(2,): 2, (5,): 5, (4,): 4, (1,): 1, (3,): 3}
    transitions = traffic.transitions
    # Returns: {((2,), 1): {(1,), (2,), (3,)}, ((2,), 2): {(2,), (5,), (4,), (1,), (3,)}, ...}
```

This will instead cause the regions/transitions to be computed on first access (after caches are reset by for instance :func:`~sentient.Abstractions.TrafficModelLinearPETC.refine`).
Use

``` python
    region_descriptors = traffic.return_region_descriptors()
    # Returns: {(2,): (x1*(0.0325014371942073*x1 - 0.758236561497541*x2 + 2.28413716988318*x3 - 1.57991935089754*x4) + x2*(-0.758236561497541*x1 + 2.38486724990143*x2 + 0.198562111037632*x3 + 2.68392804714407*x4) + x3*(2.28413716988318*x1 + 0.198562111037632*x2 + 3.0913841666531*x3 - 2.03625414224163*x4) + x4*(-1.57991935089754*x1 + 2.68392804714407*x2 - 2.03625414224163*x3 + 2.14744812716726*x4) <= 0) & (x1*(-34.1084257021831*x1 + 22.2685952023407*x2 - 68.7949413314049*x3 + 47.7543148417454*x4) + x2*(22.2685952023407*x1 - 102.519813292189*x2 - 8.36602721281553*x3 - 82.3989681651887*x4) + x3*(-68.7949413314049*x1 - 8.36602721281553*x2 - 121.315797670082*x3 + 56.7655786995279*x4) + x4*(47.7543148417454*x1 - 82.3989681651887*x2 + 56.7655786995279*x3 - 97.0796601856019*x4) < 0), ... }
```
to obtain the expressions describing the actual regions.

Finally, the traffic model can be saved for future use:

``` python
    # To pickle the object:
    traffic.export('traffic_petc', 'pickle')

    # To save to a .json file:
    traffic.export('traffic_petc', 'json')
```
The files will be saved to the ``saves`` folder.

###Nonlinear ETC

In this example, a traffic model for a nonhomogeneous nonlinear system will be generated. 
The dynamics are first declared and the controller is specified in ETC form:

``` python
    import sympy
    import sentient.util as utils

    # Define
    state_vector = x1, x2, e1, e2 = sympy.symbols('x1 x2 e1 e2')

    # Define controller (in etc form)
    u1 = -(x2+e2) - (x1+e1)**2*(x2+e2) - (x2+e2)**3

    # Define dynamics
    x1dot = x1
    x2dot = x1**2*x2 + x2**3 + u1
    dynamics = [x1dot, x2dot, -x1dot, -x2dot]
```

These dynamics are not yet homogeneous, so they are homogenized (see ...):

``` python
    # Make the system homogeneous (with degree 2)
    hom_degree = 2
    dynamics, state_vector = utils.make_homogeneous_etc(dynamics, state_vector, hom_degree)
    dynamics = sympy.Matrix(dynamics)
```

Then we define the triggering condition and the portion of the state space we want to consider.

``` python
    # Triggering condition & other etc.
    trigger = ex**2 + ey**2 - (x1**2+y1**2)*(0.0127*0.3)**2

    # State space limits
    state_space_limits = [[-2.5, 2.5], [-2.5, 2.5]]
```

And lastly, we define the traffic model (since we homogenized the dynamics, ``homogenization_flag`` should be set to ``True``):

``` python
    import sentient.Abstractions as abstr

    traffic = abstr.TrafficModelNonlinearETC(dynamics, hom_degree, trigger, state_vector, homogenization_flag=True, state_space_limits=state_space_limits)
    regions, transitions = traffic.create_abstraction()
    # Result: {'1': 0.003949281693284397, '2': 0.003924684110791467, ...}, {('1', (0.00358211491454367, 0.003949281693284397)): [1, 2, 6, 7], ... }
```

Now, the state space has been partitioned by gridding (default). To partition the state space by means of manifold, set ``partition_method=manifolds``.
Alternatively, the regions/transitions can also be accessed separately:

``` python
    regions = traffic.regions
    # Returns: {'1': 0.003949281693284397, '2': 0.003924684110791467, ...}
    transitions = traffic.transitions
    # Returns: {('1', (0.00358211491454367, 0.003949281693284397)): [1, 2, 6, 7], ... }
```

This will instead cause the regions/transitions to be computed on first access.
Use

``` python
    region_descriptors = traffic.return_region_descriptors()
    # Returns: {'1': (-1.0*x1 <= 2.5) & (1.0*x1 <= -1.5) & (-1.0*x2 <= 2.5) & (1.0*x2 <= -1.5), ...}
```
to obtain the expressions describing the actual regions.

Finally, the traffic model can be saved for future use:

``` python
    # To pickle the object:
    traffic.export('traffic_etc', 'pickle')

    # To save to a .json file:
    traffic.export('traffic_etc', 'json')
```
The files will be saved to the ``saves`` folder.

## Scheduling Examples

In the two following examples, two identical linear PETC systems are used. These have been computed and saved before hand, and are loaded as follows:

``` python
    import sentient.Abstractions as abstr
    traffic1 = abstr.TrafficModelLinearPETC.from_bytestream_file('traffic1.pickle')
    traffic2 = abstr.TrafficModelLinearPETC.from_bytestream_file('traffic1.pickle')
```

To determine which of the scheduling algorithms should be used see ...

### Scheduling using NTGAs and UPPAAL Stratego

Here a scheduler is generated by representing the traffic models by TGA and adding a network. Then using [UPPAAL Stratego](https://people.cs.aau.dk/~marius/stratego/), a strategy is generated and automatically parsed.
First both traffic models are converted:

``` python
    import sentient.Scheduling.NTGA as sched
    cl1 = sched.controlloop(traffic1)
    cl2 = sched.controlloop(traffic1)
```
And a network is defined:

``` python
    net = sched.Network()
    nta = sched.NTA(net, [cl1, cl2])
```
Then a scheduler is generated by:

``` python
    nta.generate_strategy(parse_strategy=True)
    # Result: {"('7', '15')": [[[[1, 0]], [[0.07]], [[0, -1], [0, 1], [0, -1], [0, 1]], [[-0.09], [0.0015], [0.018500000000000003], [0.15]], 0], [[[1, 0], [1, -1], [0, 1]], [[0.07], [0], [0.07]], [], [], 0]], ...
```
This will save the parsed strategy to a file in ``strategy``. The contents of the file is a dictionary which maps a tuple of states to a collection of (in)equality conditions on the clocks of the control loops and which controlloop to trigger. I.e.
```python
{('7', '15'): [(A1 @ c = b1 & E1 @ c <= d1), loop_to_trigger), (), ...], ('7', '16'): ... }
```


### Scheduling by solving safety games


Similar to before, first both traffic models are converted:

``` python
    import sentient.Scheduling.fpiter as sched
    # For the example do not use BDDs to represent the models
    cl1 = sched.controlloop(traffic1, use_bdd=False)
    cl2 = sched.controlloop(traffic1, use_bdd=False)
```
These are then combined into a system, and a scheduler is generated:

``` python
    S = sched.system([cl1, cl2])
    Ux = S.generate_safety_scheduler() # Scheduler
    # Results: ({('T12', 'W12,1'): {('w', 't'), ('w', 'w'), ('t', 'w')}, ('T12', 'W18,7'): {('w', 't'), ('w', 'w'), ...}, None)
```
The method ``generate_safety_scheduler`` will automatically choose the (likely) most efficient algorithm.








[comment]: <> (To install the project download or clone the project to a local directory.)

[comment]: <> (In project root directory there is scripts folder. There are two subfolders 'mac' and 'ubuntu' )

[comment]: <> (containing scripts to install dependencies. For Linux, there are three subfolders again '16.04',)

[comment]: <> ('18.04', and '20.04'. Depending on your Ubuntu version run the appropriate install_deps.sh. Scripts )

[comment]: <> (should be run as sudo user in Linux and normal user on Mac.)

[comment]: <> (Download and copy the third-party tools, dReeal, dReach, and Flow* in the 'third-party' folder in the root project)

[comment]: <> (directory if you want to install them. You can download them from below location:)

[comment]: <> (    dReal - https://github.com/dreal/dreal4)

[comment]: <> (    dReach - https://github.com/dreal/dreal3/releases/tag/v3.16.06.02)

[comment]: <> (    Flow* - https://flowstar.org/dowloads/)
    
[comment]: <> (For example '/{location to project}/control_system_abstractions/third-party/dreal4/')

[comment]: <> (should contain the extracted contents of the dReal4 project. Similarly for Flow*, copy the source files directly under)

[comment]: <> (the '/third-party/flowstar-2.0.0' directory. Follow the instructions to build the tools. For dReach copy the executable for )

[comment]: <> (either Linux or Mac directly under the '/third-party/dreach/ubuntu' or '/third-party/dreach/mac'.)

[comment]: <> (Alternatively, if you already have any or all of dReal, dReach, and Flow* setup on you system, update the location)

[comment]: <> (in config.py. For e.g. 'dreal_path = /home/user/dreal4/bazel-bin/dreal/dreal'.)

[comment]: <> (Create a virtual environment to install the Python dependencies:)

[comment]: <> (    export VENV=~/projects/control_systems_abstractions/env)

[comment]: <> (    python3 -m venv $VENV)

[comment]: <> (Build the tool using the following command:)
    
[comment]: <> (    $VENV/bin/pip3 install -vvv .)

[comment]: <> (Run the application using following command:)

[comment]: <> (    python3 etc2pta -h  # For details of input args)

[comment]: <> (    python3 etc2pta -i {path_to_file} -s linear)

[comment]: <> (    python3 etc2pta -i {path_to_file} -s non-linear)
    
[comment]: <> (Alternatively, a GUI based input can be used by running command:)

[comment]: <> (    python3 etc2pta_GUI )
