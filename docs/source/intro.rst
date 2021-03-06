****************************
Overview
****************************

This chapter describes the ``ws3`` Python software package. ``ws3`` (short for *Wood Supply Simulation System*) is an open-source software package that is designed to model *wood supply planning problems* (WSPP), in the context of sustainable forest management.

Background
=======

The WSPP basically consists of determining the location, nature, and timing of forest managment activities (i.e., *actions*) for a given forest, typically over multiple planning periods (often spanning a planning horizon of 100 years or more). This is a very complex problem, so in practice the WSPP process is typically supported by complex software models that simulate an alternating sequence of *actions* and *growth* for each time step, starting from an initial forest inventory.

Wood supply models require complex input datasets. WSM input data can be divided into *static* and *dynamic* components.

Static WSM input data types include initial forest inventory, growth and yield curves, action definitions, transition definitions, and a schedule of prescribed activities to simulate.
Dynamic WSM input data may include a combination of heuristic and optimization-based processes to automatically derive a dynamic activity schedule (which gets layered on top of the static activity schedule).

The forest inventory data is typically aggregated into a manageable number of *strata* (i.e., *development types*), which simplifies the modelling.  Each development type is linked to *growth and yield* functions describing the change in key attributes attributes (e.g., species-wise standing timber volume, number of merchantable stems per unit area, wildlife habitat suitability index value, etc.) expressed as a function of statum age. Each development type may also be associated with one or more *actions*, which can yield *output products* (e.g., species-wise assortments of raw timber products, cost, treated area, etc.). Applying an action to a development type induces a *state transition* (i.e., applying an action may modify one or more stratification variables, effectively transitioning the treated area to a different development type). 

Given a set of static inputs, a given WSM can be used to simulate a number of *scenarios*. Generally, scenarios differ only in terms of the dynamic activity schedule that is simulated. Comparing output from several scenarios is the basic mechanism by which forest managers derive insight from wood supply models.

There are two basic approached that can be used (independently, or in combination) to generate the dynamic activity schedules for each scenario.

The simplest approach, which we call the *heuristic* activity scheduling method, involves defining period-wise targets for a single key output (e.g., total harvest volume) along with a set of rules that determines the order in which actions are applied to eligible development types. At each time step, the model iteratively applies actions according to the rules until the output target value is met, or it runs out of eligible area. At this point, the model simulates one time-step worth of growth, and the process repeats until the end of the planning horizon.

A slightly more complex approach, which we call the *optimization* activity scheduling method, involves defining an optimization problem (i.e., an objective function and constraints), and solving this problem to optimality (using one of several available third-party mathematical solver software packages).

Although the optimization approach is more powerful than the heuristic approach for modelling harvesting and other anthopic activites, an optimization approach is not appropriate for modelling strongly-stochastic disturbance processes (e.g., wildfire, insect invasions, blowdown). Thus, a hybrid heuristic-optimization approach may be best when modelling a combination of anthopic and natural disturbance processes.

Package Design and Implementation
================

The ``ws3`` package is implemented using the Python programming language. ``ws3`` is basically an aspatial wood supply model, which applies actions to development types, simulates growth, and tracks inventory area at each time step. Aspatial models output aspatial activity schedules---each line of the output schedule specifies the stratification variable values (which constitute a unique key into the list of development types), the time step, the action code, and the area treated.

Because the model is aspatial, the area treated on a given line of the output schedule may not be spatially contiguous (i.e., the area may be geographically dispersed throughout the landscape). Furthermore, in the common case where only a subset of development type area is treated in a given time step, the aspatial model provides not information regarding which subset of available area is treated (and, conversely, not treated). Some applications (e.g., linking to spatially-explicit or highly--spatially-referenced models) require a spatially-explicit activity schedule. ``ws3`` includes a *spatial disturbance allocator* sub-module, which contains functions that can map aspatial multi-period action schedules onto a rasterized spatial representation of the forest.

``ws3`` uses a scripted Python interface to control the model, which provides maximum flexibility and makes it very easy to automate modelling workflows. This ensures reproducible methodologies, and makes it relatively easy to link ``ws3`` models to other software packages to form complex modelling pipelines. The scripted interface also makes it relatively easy to implement custom data-importing functions, which makes it easier to import existing data from a variety of ad-hoc sources without the need to recompile the data into a standard ``ws3``-specific format (i.e., importing functions can be implemented such that the conversion process is fully automated and applied to raw input data *on the fly*). Similarly, users can easily implment custom functions to re-format ``ws3`` output data *on the fly* (either for static serialization to disk, or to be piped live into another process). 

Although we recommend using Jupyter Notebooks as an interactive interface to ``ws3`` (the package was specifically designed with an interactive notebook interface in mind), ``ws3`` functions can also be imported and run in fully scripted workflow (e.g., non-interactive batch processes can be run in a massively-parallelled workflow on high-performance--computing resources, if available). The ability to mix interactive and massively-paralleled non-interactive workflows is a unique feature of ``ws3``.

``ws3`` is a complex and flexible collection of functional software units. The following sections describe some of the main classes and functions in the package, and describe some common use cases, and link to sample notebooks that implement these use cases.

Overview of Main Classes and Functions
=========================

This section describes some of the main classes and functions that make up.

The ``ForestModel`` class is the core class in the package. This class encapsulates all the information used to simulate scenarios from a given dataset (i.e., stratified intial inventory, growth and yield functions, action eligibility, transition matrix, action schedule, etc.), as well as a large collection of functions to import and export data, generate activity schedules, and simulate application of these schedules  (i.e., run scenarios).

At the heart of the ``ForestModel`` class is a list of ``DevelopentType`` instances. Each ``DevelopmentType`` instance encapsulates information about one development type (i.e., a forest stratum, which is an aggregate of smaller *stands* that make up the raw forest inventory input data). The ``DevelopmentType`` class also stores a list of operable *actions*, maps *state variable transitions* to these actions, stores growth and yield functions, and knows how to *grow itself* when time is incremented during a simulation.

.. To Do: Finish documenting main stuff here.
 
Common Use Case and Sample Notebooks
===========================

In this section, we assume an interactive Jupyter Notebook environment is used to interface with ``ws3``.

A typical use case starts with creating an instance of the ``ForestModel`` class. Then, we need to load data into this instance, define one or more scenarios (using a mix of heuristic and optimization approaches), run the scenarios, and export output data to a format suitable for analysis (or link to the next model in a larger modelling pipeline).

The first step in typical workflow is to run a mix of standard ``ws3`` and custom data-importing functions.  These functions import data from various sources, *on-the-fly* reformat this data to be compatible with ``ws3``, and load the reformated data into the ``ForestModel`` instance using standard methods. For example, ``ws3`` includes functions to import legacy Woodstock [#]_ model data (including LANDSCAPE, CONSTANTS, AREAS, YIELDS, LIFESPAN, ACTIONS, TRANSITIONS, and SCHEDULE section data), as well as functions to import and rasterize vector stand inventory data.

For example, one might define the following custom Python function in a Jupyter Notebook, to import data formatted for Woodstock.::

    def instantiate_forestmodel(model_name, model_path, horizon,
                                period_length, max_age, add_null_action=True):
        fm = ForestModel(model_name=model_name, 
	 	 	 model_path=model_path, 
 	 		 horizon=horizon,     
			 period_length=period_length,
			 max_age=max_age)
	fm.import_landscape_section()
	fm.import_areas_section()
	fm.import_yields_section()
	fm.import_actions_section()
	fm.add_null_action()
	fm.import_transitions_section()
	fm.reset_actions()
	fm.initialize_areas()
	fm.grow()
	return fm

The next step in a typical workflow is to define one or more scenarios. Assuming that we are using an optimization approach to harvest scheduling, we need to define an objective function (e.g., maximize total harvest volume) and constraints (e.g., species-wise volume and area even-flow constraints, ending standing inventory constraints, periodic minimum late-seral-stage area constraints) [#]_, build the optimization model matrix, solve the model to optimality [#]_. 


.. [#] Woodstock software is part of `Remsoft Solution Suite <http://www.remsoft.com/forestry.php>`_. 
.. [#] ``ws3`` currently implements functions to formulate and solve *Model I* wood supply optimization problems---however, the package was deliberately designed to make it easy to transparently switch between *Model I*,  *Model II* and *Model III* formulations without affecting the rest of the modelling workflow. ``ws3`` currently has placeholder function stubs for *Model II* and *Model III* formulations, which will be implemented in later versions as the need arises. For more information on wood supply model formulations, see Chapter 16 of the `Handbook of Operations Research in Natural Resources <http://www.springer.com/gp/book/9780387718149>`_.
.. [#] ``ws3`` currently uses the `Gurobi <http://www.gurobi.com/>`_ solver to solve the linear programming (LP) problems to optimality. We chose Gurobi because it is one of the top two solvers currently available (along with the `CPLEX <https://www.ibm.com/analytics/data-science/prescriptive-analytics/cplex-optimizer>`_ solver), has a simple and flexible policy for requesting unlimited licences for free use in research projects, has elegant Python bindings, and we like the technical documentation. However, we deliberately used a modular design, which allows us to transparently switch to a different solver in ``ws3`` without affecting the rest of the workflow---this design will make it easy to implement an interface to addional solvers in future releases.


 
