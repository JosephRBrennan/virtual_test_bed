# Model Inputs

There are two VTR models stored on the virtual test bed; a standalone
neutronics model and multi-physics model.
There are additional sub-app input files that are called by the multi-physics model
that can also be ran standalone with slight modification.
In the preceding sections, we will walk through the input files for the
Griffin standalone model and multi-physics model.

## Griffin Standalone

The complete input file for the Griffin neutronics model is shown below.

!alert note
The mesh for the Griffin model and the cross sections and equivalence factors are hosted on LFS.
Please refer to [LFS instructions](resources/how_to_use_vtb.md#lfs) to download them.

!listing sfr/vtr/griffin_only.i


### Model Parameters

The first block is for model parameters.
This could include the reactor state point (fuel temperature,
coolant temperature/density, etc.), reactor power level, or any
user-desired information.
For consistency, the initial state point is set with the state on
the cross section file.
The cross sections for this model are not parameterized, instead, there only
exists one state point.
We'll see that the standalone model is used to verify the neutronics solution
and multi-group cross section library to Serpent.
The power for the VTR is 300 MW and set with `tpow`.

### Global Parameters

Global parameters are parameters that may be referenced
throughout the input file.
This block is user specific; the purpose is
to simplify repeated variable entries.
However, be careful to not override a default parameter
option in a later block without meaning to.

For this model, we store cross section information here.
Notice that the library name and file location are defined.
For simplicity, all of the cross section files are stored in the
`xs` directory.
Also, the cross section parameterization (names and variables)
on the xml file are set.
Even though the library is not parameterized, we still note the state variables
at which the cross sections are generated at.
The unit of the cross sections are specified with the logical, `is_meter`,
either meters or centimeters.
We also define the materials as a "pseudo" mixture with
a density of 1.0, which refers to the homogenized cross
sections generated with Serpent and processed with ISOXML.

If these definitions do not make sense now, we will
walk-through their uses and meanings in the subsequent
blocks.

!listing sfr/vtr/griffin_only.i
         block=GlobalParams

### Geometry and Mesh

In this section, we will cover the mesh inputs.
The full mesh block can be found below.

!listing sfr/vtr/griffin_only.i
         block=Mesh

The computational mesh is generated with CUBIT, an external code developed at
Sandia National Laboratory, from the VTR geometry.
An internal INL tool is used to post-process the mesh to ensure
consistency with the Serpent generated multi-group cross section
library (i.e. material region and equivalent region identification).
The resulting output is an Exodus file that is read by MOOSE
using the [!style color=orange](FileMeshGenerator) type.
We need to directly specify the ids on the mesh using
[!style color=red](exodus_extra_element_integers)
which will then read in the material ids and equivalent ids
(for the SPH factors).
This is performed with the [!style color=orange](ExtraElementIDCopyGenerator)
type by defining the "equivalence_id" as the source
(from [!style color=red](exodus_extra_element_integers))
and "equivalence_id" as the target.
A simple way to identify the material and equivalent ids is to
open the mesh file `vtr_core.e`
in an Exodus supported visualization tool (i.e. ParaView).
All of the mesh files are stored in the `mesh` directory.

### Equivalence

Equivalent cross sections are handled in the `[Equivalent]` block.
Super homogenization (SPH) factors are generated from a Serpent solution to
correct homogenization errors and preserve the reaction rates
of the heterogenous solution.
For convenience, all of the equivalence files are stored in the
`sph` directory.
The equivalence library is first defined with
[!style color=red](equivalence_library) which is an xml file
generated by the ISOXML pre-processing code.
The [!style color=red](equivalence_grid_names) and
[!style color=red](equivalence_grid_variables) are then specified
with the state point variables on the equivalence library.
These variables define the SPH factor parameterization.
In this model, the cross section library consists of only one
state point, made up of fuel temperature, coolant temperature,
and control rod fraction state parameters.
The `compute_factors = false` simply tells the system to use the SPH factors
directly from the equivalence library.
The "CFEM-Diffusion" is defined in the `[Transport Systems]` block.

!listing sfr/vtr/griffin_only.i
         block=Equivalence

### AuxVariables and AuxKernels

There are three AuxVariables that are defined in this model;
the fuel temperature, coolant temperature, and control rod
fraction.
These variables are defined to represent the state parameters
that parameterize the cross section and equivalence libraries.
The values for these variables are set with a parsed expression from
the model parameters defined at the top of the input file.

!listing sfr/vtr/griffin_only.i
         block=AuxVariables

There are no AuxKernels that act on these variables since there are
no derived quantities to obtain.

### Initial Conditions and Functions

User specified functions are defined in this block.
Here, we construct a function for the control rod movement.
The type is set with the
[!style color=orange](ConstantFunction) parameter and the value
equal to the all rods out case (1.92).
This value can be changed to obtain ARO, ARI, and fractional rod cases.

!listing sfr/vtr/griffin_only.i
         block=Functions

### Materials

Material cross sections are specified with the multi-group
cross section library defined by [!style color=red](library_file)
in the `[GlobalParameters]` block.
In this model, we define three types of materials: fuel materials,
materials for the reflector/shield/SR assemblies, and control rod materials.
The first two materials are given the
[!style color=orange](CoupledFeedbackMatIDNeutronicsMaterial) type.
This type uses neutronics properties based on mixed isotopes
from the multi-group cross section library.
It also allows the assignment of cross sections based on dynamic state
variables (i.e. the AuxVariables defined above).
In this example, we have a pseudo mixture and density that is
defined in the `[GlobalParameters]` block.
We must specify which blocks are fissile materials with the
[!style color=red](block) card.
The block ids (or names) can be identified on the mesh with an Exodus
supported viewer (i.e. ParaView).
For each block id, we also need to identify the densities and isotopes.
For convenience, these parameters were put in the `[GlobalParameters]` block.
The [!style color=red](plus) card indicates if absorption, fission, and
kappa fission are to be used (for fissile materials).

The control rod materials are given by the
[!style color=orange](CoupledFeedbackRoddedNeutronicsMaterial) type.
First, the [!style color=red](segment_material_ids) sets the materials
of the control rod.
The absorption region of the control rod is identified with the `116` id.
The length of the rod segments are set with
[!style color=red](segment_material_ids).
Only the active control rod segment lengths are declared.
Next, a control rod function is specified for withdrawal as well as
the withdrawal direction with [!style color=red](front_position_function)
and [!style color=red](rod_withdrawn_direction), respectively.
The control rod function is defined in the `[Functions]` block.
Lastly, the pseudo isotope mixture and density is declared for the control
rod segments.

!listing sfr/vtr/griffin_only.i
         block=Materials

The power density is not calculated implicitly and must be defined in
the `[PowerDensity]` block.
This block provides the power density with a user specified power and
the proper flux scaling factor in the postprocessor.
The two required arguments are [!style color=red](power) and
[!style color=red](power_density_variable).
The power is in units of Watts.
The [!style color=red](power_density_variable) simply sets
the variable name.
Optional arguments include the post-processors to use.
The [!style color=red](integrated_power_postprocessor) sets the
desired name for the integrated power post-processor.
Likewise, the [!style color=red](power_scaling_postprocessor)
sets the desired name for the power scaling post-processor.
Lastly, the family and order parameters are the polynomial
representations  of the power density corresponding to the
underlying FEM.

!listing sfr/vtr/griffin_only.i
         block=PowerDensity

### Transport System

The transport system is the heart of the Griffin simulation.
We define the transport particle and eigenvalue equation with
[!style color=red](particle) and
[!style color=red](equation_type), respectively.
The number of energy groups is set to "6" with the parameter
[!style color=red](G).
Vacuum boundary conditions are imposed on side sets "1 2 3".
The `[CFEM-Diffusion]` sub-block defines the method to solve the
problem.
We use Griffin's diffusion method built on the MOOSE
continuous finite element method solver.
Finite element parameters are set with the
[!style color=red](family) and
[!style color=red](order), which define the family of
functions used to approximate the solution and
polynomial order.
The last two options,
[!style color=red](assemble_scattering_jacobian) and
[!style color=red](assemble_fission_jacobian) are
required to use the "PJFNKMO" method defined later in the
`[Executioner]`.
This tells the finite element solver to not update the material
cross sections at each linear iteration.

!listing sfr/vtr/griffin_only.i
         block=TransportSystems

### Executioner

The `[Executioner]` block tells the solver what type of problem
it needs to solve.
Here, we select [!style color=orange](Eigenvalue) as the executioner
type which will solve the eigenvalue problem for the criticality
and multi-group scalar flux (eigenvalue and eigenvectors).
We also specify to solve with a Preconditioned Jacobian Free
Newton Krylov Matrix Only method by setting
[!style color=red](solve_type) equal to "PJFNKMO".
The Matrix Only method forces the solver to not update
material properties (i.e. cross sections) in the linear iterations
of the solve.
To use this solver, the [!style color=red](constant_matrices)
parameter must be set to "true".
There are a number of optional arguments that may also be included.
For example, we define a few petsc options and non-linear solver
tolerances.

The quadrature sub-block adds a custom quadrature rule.
In this case, it is set to a fourth order.

!listing sfr/vtr/griffin_only.i
         block=Executioner

### Multi-apps and Transfers

This model is to run Griffin neutronics only.
In the multi-physics model, we will declare sub-apps that include
BISON and SAM.

### Post-processors, Debug, and Outputs

The last blocks are for post-processors, debug options, and outputs.
A post-processor can be thought of as an operator to compute a single scalar quantity
of interest from the solution.
For example, the total core power.
This is defined with the
[!style color=orange](ElementIntegralVariablePostprocessor)
type and integrates the power density variable we created earlier,
[!style color=red](power_density).
The [!style color=red](execute_on) parameter declares at what point
in the calculation to compute the total power.

!listing sfr/vtr/griffin_only.i
         block=Postprocessors

Finally, the output block sets the output files from the simulation.
Two of the most common options include the exodus and csv file.
The Exodus file can be viewed with the same software as the mesh,
but now will show the solution and solution derived quantities such
as the multi-group scalar flux and power distribution.
The csv file stores a summary of the solution that includes
the k-effective.
The [!style color=red](perf_graph) parameter is helpful to evaluate
the computational run time.

!listing sfr/vtr/griffin_only.i
         block=Outputs

# How to run the model

The model can be ran with Griffin in serial or parallel.
In serial, the solve takes approximately 1 min 40 sec on a
Xeon Platinum 8268 processor.
In parallel, with 48 processors, the solve takes approximately 6 seconds.


Run it via:

 `griffin-opt -i griffin_only.i`

and in parallel,

 `mpirun -n 48 griffin-opt -i griffin_only.i`

## Multi-physics Model

The complete input file for the multi-physics model is shown below.

!listing sfr/vtr/griffin_multiphysics.i

The neutronics input is similar to the standalone model that is described
in the previous section.
Still, we will highlight input additions that allow for multi-physics
coupling (multi-apps and transfers).
We'll also go into details on the sub-app input files (BISON, SAM, Tensor Mechanics).

### MultiApps id=griffin_multiapps

In addition to the main-app neutronics input, there are now
BISON inputs, SAM inputs, and a MOOSE Tensor Mechanics input.
These sub-apps are defined in the `[MultiApps]` block.
Note that only the sub-apps that are directly coupled to Griffin are defined here.
In other words, the SAM model will be declared as a sub-app in the BISON model.

!listing sfr/vtr/griffin_multiphysics.i
         block=MultiApps

First, we start with the BISON sub-apps.
There are three BISON MultiApps that correspond to the fuel rods in the
three orifices of the VTR.
For each of these models, we give the sub-app a name (i.e. bison_1, bison_2, bison_3).
The [!style color=red](type) and [!style color=red](app_type) is set with
[!style color=orange](FullSolveMultiApp) and
[!style color=orange](BlueCrabApp), respectively.
A different command line argument is passed to override a single parameter (the mass flow rate) in the input files
to form the three different simulations.
The former performs a complete simulation every time the sub-app executes.
The latter informs the code what type of app to build.
We want BlueCrab for multi-physics simulations because it is a
package of Griffin, BISON, and SAM.
Next, the [!style color=red](positions_file) points to a file that contains
spatial coordinates that define a point for the app location.
The [!style color=red](input_files) simply points to the sub-app input file.
We'll also set the sub-app to run its simulation at the end of the main-app solve (Griffin)
with [!style color=red](execute_on) = 'TIMESTEP_END'.
Next, we specify the sub-app to run on one processor with [!style color=red](max_procs_per_app).
Finally, the [!style color=red](output_in_position) allows the output to be 'moved' by its
position vector.

The core support plate is modeled with the Tensor Mechanics module included in MOOSE.
The sub-app is defined in a similar manner as the BISON apps.
However, we only need to run this sub-app at the beginning of the simulation since
the inlet temperature is fixed (the resulting radial displacements are also fixed).
This is done by setting [!style color=red](execute_on) = 'INITIAL'.

### Transfers id=griffin_transfers

Variables are transfered internally to each app with MOOSE.
We just need to specify which variables are transfered to each app and in which direction.

!listing sfr/vtr/griffin_multiphysics.i
         block=Transfers

At a high level, the scheme works as follows.
The power density is solved with Griffin and passed to BISON.
BISON solves for the temperature distribution of the fuel and transfers the
wall temperature to SAM.
SAM calculates the coolant temperature and heat transfer coefficient,
which then gets passed back to BISON.
Following the BISON and SAM iterations, the fuel temperature gets passed back to
Griffin and the cross sections are updated.

In a separate iteration between the thermal and mechanical BISON models, the mechanical model
solves and returns the thermal expansion of the fuel.
The thermal expansion of the core support plate is solved in a native MOOSE Tensor Mechanics module
and the radial displacements are passed back to the Griffin main-app.

The transfers are organized in the following way: transfer type, direction, and
variable.
For example, the power density transfer from Griffin to BISON is specified with the
[!style color=orange](MultiAppProjectionTransfer) type and direction with the
[!style color=red](to_multi_app) parameter. The
[!style color=red](source_variable) and [!style color=red](variable) is the
`power_density`.

### How to run the model

The model can be ran by executing the BlueCrab app that consists of
Griffin, BISON, and SAM.
In parallel, with 48 processors, the solve takes approximately 52 seconds.

Run it via:

 `mpirun -n 48 blue_crab-opt -i griffin_multiphysics.i`

## BISON Thermal Model id=bison_thermal

The input file for the BISON thermal model of a VTR fuel rod is displayed below.

!listing sfr/vtr/bison_thermal_only.i

The top of the input file houses the model parameters.
For a BISON model, this would include the rod diameter, clad thickness, fuel height, etc.

### Global Parameters

Global parameters are parameters that are used in more than one block.
For example, the family of functions and polynomial order are defined for the finite element
solver.

!listing sfr/vtr/bison_thermal_only.i
         block=GlobalParams

### Geometry and Mesh

The pellet mesh is setup with the [!style color=orange](SmearedPelletMeshGenerator) type.
This object generates a smeared pellet mesh for use in simulating fuel rods with pellet fuel.
In a smeared mesh, the fuel stack is treated as a single rectangle of fuel (i.e., dishes and chamfers are not included).
There are a number of geometry and meshing options that are shown but will not be discussed.
The clad is added around the pellet mesh with the [!style color=orange](SubdomainBoundingBoxGenerator) type.
More information about the mesh generation may be found in the BISON documentation for the
[SmearedPelletMeshGenerator](https://mooseframework.inl.gov/bison/source/meshgenerators/SmearedPelletMeshGenerator.html#smearedpelletmeshgenerator)
and the
[SubdomainBoundingBoxGenerator](https://mooseframework.inl.gov/bison/source/meshgenerators/SubdomainBoundingBoxGenerator.html#subdomainboundingboxgenerator).

!listing sfr/vtr/bison_thermal_only.i
         block=Mesh

### Variables and Kernels

A variable is a desired quantity to be solved for in the finite element solution.
In the BISON thermal model, we want the temperature distribution of the fuel element.


!listing sfr/vtr/bison_thermal_only.i
         block=Variables

Kernels are the inner product terms that make up a weak form of a differential equation.
The kernels define the differential equation to solve.
The active kernels that are used in the solve are identified with
[!style color=red](active).
For example, a time derivative kernel is defined but not activated as we are interested
in the steady state solution.

!listing sfr/vtr/bison_thermal_only.i
         block=Kernels

### Modules

`ThermalContact` is a MOOSE module to control the heat transfer in regions without meshes.
In this model, we use `[ThermalContact]` to model the gap between the pellet and clad.
We do this by defining the [!style color=red](GapHeatTransfer) type and coresponding parameters.
Detailed information about the ThermalContact module may be found in the MOOSE
[documentation](https://mooseframework.inl.gov/source/actions/ThermalContactAction.html).

!listing sfr/vtr/bison_thermal_only.i
         block=ThermalContact

### AuxVariables and AuxKernels

AuxVariables are variables that are derived from the solution variable (in this case, temperature).
These can be quantities such as the fuel temperature, wall temperature, coolant temperature, etc.
We also define the `power_density`, a variable transfered from the Griffin main-app.

!listing sfr/vtr/bison_thermal_only.i
         block=AuxVariables

AuxKernels are kernels that act on a variable to derive an AuxVariable.
Consider the fuel temperature and wall temperature.
These AuxKernels are of type [!style color=red](CoupledAux). These copy the values from
the solution variable, Temperature, into the respective variables twall and tfuel on the desired blocks.
AuxKernels are also defined to obtain (and scale) the power density from Griffin.

!listing sfr/vtr/bison_thermal_only.i
         block=AuxKernels

### Initial Conditions and Functions

Users may specify custom functions in the `Functions` block.
For example, we have defined the linear heat generation rate, heterogenous
and homogenous power densities.
These functions would be used in a standalone BISON model if the power density is not
provided by Griffin.

!listing sfr/vtr/bison_thermal_only.i
         block=Functions

### Materials and UserObjects

Material characteristics are defined in the `[Materials]` block.
They include the thermal properties and density of the fuel and clad.

!listing sfr/vtr/bison_thermal_only.i
         block=Materials

### Boundary Conditions

The boundary conditions in the model are used to couple with SAM, where
SAM will provide the coolant temperature and heat transfer coefficient.
As such, the type is defined as a [!style color=orange](CoupledConvectiveHeatFluxBC)
and coupled to the Temperature solution variable.

!listing sfr/vtr/bison_thermal_only.i
         block=BCs

### Execution Parameters id=bison_exec

Preconditioners transform the given problem into a form that is more suitable for the numerical solver,
thus improving solver performance.
The single matrix preconditioner (SMP) is defined here that builds a preconditioner using user defined off-diagonal parts of the Jacobian matrix.

!listing sfr/vtr/bison_thermal_only.i
         block=Preconditioning

The `[Executioner]` block tells the solver what type of problem it is to solve.
Here, we select [!style color=orange](Steady) as the executioner type to identify that the problem
is steady state.
We also specify to solve with a Preconditioned Jacobian Free
Newton Krylov method by setting
[!style color=red](solve_type) equal to "PJFNK".

!listing sfr/vtr/bison_thermal_only.i
         block=Executioner

### MultiApps and Transfers

There are two sub-apps that extend from this BISON sub-app input deck; a SAM thermal fluids model and a BISON mechanics model.
Both are defined here in the same manner as discussed in [#griffin_multiapps].

!listing sfr/vtr/bison_thermal_only.i
         block=MultiApps

To accompany the multi-apps, variable transfers must be specified for coupling.
We want to pass the outer wall temperature calculated in the BISON thermal model to SAM.
SAM will then compute the coolant temperature and heat transfer coefficient to pass back to the BISON thermal model.
There is also coupling between the thermal BISON model and mechanical model.
A transfer is defined to pass the temperature distribution from the thermal model to the mechanical model.
The mechanical model then solves for the thermal expansion and transfers the x and y displacements back to the thermal model.

!listing sfr/vtr/bison_thermal_only.i
         block=Transfers

### Post-processors, Debug, and Outputs

Postprocessors are used to derive desired quantities from the solution variable(s).
Some of the quantities that we may be interested in include the average, maximum, and minimum fuel, clad, and wall temperatures.

!listing sfr/vtr/bison_thermal_only.i
         block=Postprocessors

The outputs files are supressed but may be turned on if desired.

!listing sfr/vtr/bison_thermal_only.i
         block=Outputs

## SAM Thermal-hydraulic Model

The input file for the SAM thermal hydraulic model of a VTR assembly is displayed below.

!listing sfr/vtr/sam_channel.i

The top of input file houses the model parameters.
For a SAM model, this could include the number of rods, rod diameter, channel dimensions, etc.

### Global Parameters

Global parameters are parameters that are used in more than one block.
This model includes the initial pressure, temperature, and velocity variables of the fluid.

!listing sfr/vtr/sam_channel.i
         block=GlobalParams

### Equations of State

The equations of state (EOS) block is unique to SAM.
Because the VTR is sodium cooled, the EOS type is given by
[!style color=orange](PBSodiumEquationofState).

!listing sfr/vtr/sam_channel.i
         block=EOS

### Components

This block defines the components of the simulation.
In SAM, a component is defined as the physics modeling (fluid flow
and heat transfer) and mesh generation of a reactor component.

!listing sfr/vtr/sam_channel.i
         block=Components

The fuel channel is modelled with a [!style color=orange](PBOneDFluidComponent) type that simulates
1-D fluid flow using the primitive variable based fluid model.
The channel geometry is declared with a number of parameters.
Additional information can be found in the SAM User's Manual.

The channel inlet and outlet are specified with the
[!style color=orange](PBTDJ) and [!style color=orange](PBTDV) types, respectively.
The [!style color=orange](PBTDJ) component is an inlet boundary in which the flow velocity
and temperature are provided by pre-defined functions.
The [!style color=orange](PBTDV) component is a boundary in which pressure and temperature
conditions are provided by pre-defined functions.

The last block specifies heat transfer from the fuel with the
The [!style color=orange](HeatTransferWithExternalHeatStructure) type.
Further information can be found in the SAM User's Manual.

### Executioner

The same preconditioner is used as previous BISON input. See [#bison_exec].

!listing sfr/vtr/sam_channel.i
         block=Preconditioning

!listing sfr/vtr/sam_channel.i
         block=Executioner

### Post-processors, Debug, and Outputs

Postprocessors are used to derive desired quantities from the solution variable(s).
Some of the quantities that we may be interested in include the inlet and outlet temperatures,
heat transfer coeficient, and temperature maximums and minimums.

!listing sfr/vtr/sam_channel.i
         block=Postprocessors

## BISON Mechanical Model

The BISON mechanical model is constructed in the same way as the thermal model, however, the solver focuses on the
mechanical behavior.
As such, we'll focus our discussion on input blocks that are different than that of the thermal model and refer the reader
to the [#bison_thermal] for more information.

The input file for the BISON mechanical model of a VTR fuel rod is displayed below.

!listing sfr/vtr/bison_mecha_only.i

### Global Parameters

The displacements `disp_x` and `disp_y` have been added to the global parameters list, this
will make sure every object uses the displaced mesh instead of the original mesh.

!listing sfr/vtr/bison_mecha_only.i
         block=GlobalParams

### Modules

The mechanical model utilizes the solid mechanics module in MOOSE.
The model is setup such that the mechanical behavior is modeled in both the fuel and clad.
For a detailed explanation of the TensorMechanics model, the authors defer to the
[TensorMechanics](https://mooseframework.inl.gov/syntax/Modules/TensorMechanics/Master/)
section of the MOOSE documentation.

!listing sfr/vtr/bison_mecha_only.i
         block=Modules

### Materials and UserObjects

Material properties are defined in the `[Materials]` block.
The properties include the elasticity tensor, elastic stress,
thermal expansion, and density of the fuel and clad.

!listing sfr/vtr/bison_mecha_only.i
         block=Materials

### Boundary Conditions

Dirichlet boundary conditions are specified at the centerline and boundary of the problem
for each of the solution variables (`disp_x` and `disp_y`).
Because the problem is defined in RZ geometry, `disp_x` refers to the radial displacement
and `disp_y`, the axial displacement.

!listing sfr/vtr/bison_mecha_only.i
         block=BCs

### Post-processors, Debug, and Outputs

Postprocessors are used to derive desired quantities from the solution variable(s).
Some of the quantities that we may be interested in include the maximum displacement in the
radial and axial directions, and the strains.

!listing sfr/vtr/bison_mecha_only.i
         block=Postprocessors

## Core Support Plate

The input file for the core support plate tensor mechanics model is displayed below.
The objective of this simulation is to capture the radial thermal expansion of the support plate.

!listing sfr/vtr/core_support_plate_3d.i

The top of input file houses the model parameters.
Here, we specify the inlet temperature and reference temperature.

### Global Parameters

The displacements that we wish to solve for (`disp_x`, `disp_y`, and `disp_z`) are specified
as a global parameter.
This is a 3D model of the core support plate and all coordinate directions will be solved for, however,
we are only interested in the radial displacements (x and z).

!listing sfr/vtr/core_support_plate_3d.i
         block=GlobalParams

### Geometry and Mesh

This block defines the computational mesh at which the solution is to be computed on.
The mesh is supplied with an Exodus file, titled 'cyl_plate_3d.e' and read in using the
[!style color=orange](FileMeshGenerator) type.

Next, the mesh is centered at coordinate position (0,0,0) by defining a nodeset with the
[!style color=orange](BoundingBoxNodeSetGenerator) type.

!listing sfr/vtr/core_support_plate_3d.i
         block=Mesh

### AuxVariables and AuxKernels

There is one AuxVariable that is defined for this model; the core support plate temperature.
The temperature is set with the inlet temperature.

!listing sfr/vtr/core_support_plate_3d.i
         block=AuxVariables

There are no AuxKernels.

### Modules

The core support plate model utilizes the solid mechanics module that is native to MOOSE.
We are interested in the thermal expansion of the core support plate so we set the
[!style color=red](eigenstrain_names) to `thermal_expansion` and generate a stress and strain
output in each coordinate direction.
For a detailed explanation of the TensorMechanics model, the authors defer to the
[TensorMechanics](https://mooseframework.inl.gov/syntax/Modules/TensorMechanics/Master/)
section of the MOOSE documentation.

!listing sfr/vtr/core_support_plate_3d.i
         block=Modules/TensorMechanics/Master

### Initial Conditions and Functions

User specified functions are defined in this block.
Here, we construct a function for the alpha mean of 316 stainless steel.
The type is set with the
[!style color=orange](ParsedFunction) parameter and the value
equal to a second order polynomial with specified coefficients.

!listing sfr/vtr/core_support_plate_3d.i
         block=Functions

### Materials and UserObjects

Material characteristics are defined in the `[Materials]` block.
Characteristics include the mechanical properties of 316 stainless steel such as the
elasticity tensor, stress, and thermal expansion.
For a detailed explanation of the material types, the authors defer the reader to
the [TensorMechanics](https://mooseframework.inl.gov/syntax/Modules/TensorMechanics/Master/)
section of the MOOSE documentation.

!listing sfr/vtr/core_support_plate_3d.i
         block=Materials

### Boundary Conditions

Dirichlet boundary conditions are set at the problem boundary
for each of the solution variables (`disp_x`, `disp_y`, and `disp_z`).
The radial `x` and `z` directional boundary is the bottom surface of the
core support plate center.
The axial `y` directional boundary is specified at the bottom of the support plate.

!listing sfr/vtr/core_support_plate_3d.i
         block=BCs

### Execution Parameters

The same preconditioner is used as previous BISON inputs. See [#bison_exec].
For proper convergence the nonlinear iterations are forced to three in the `Executioner`.
Note that without [!style color=red](nl_forced_its) the default criteria would result in two
nonlinear iterations and the solution convergence would not be satisfied.

!listing sfr/vtr/core_support_plate_3d.i
         block=Executioner

### Post-processors, Debug, and Outputs

Some of the quantities that we may be interested in include the maximum displacement in the coordinate
directions as well as the strain.

!listing sfr/vtr/core_support_plate_3d.i
         block=Postprocessors
