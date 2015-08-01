Quick Start Guide
-----------------

This guide illustrates how to use DSP for solving the following farmer problem taken from :cite:`birge2011introduction`.

.. math::

   \min \quad & 150 x_1 + 230 x_2 + 260 x_3 \\
   & \quad + \sum_{s=1}^3 \frac{1}{3} (238 y_{1s} + 210 y_{2s} - 170 y_{3s} - 150 y_{4s} - 36 y_{5s} - 10 y_{6s}) \\
   \text{s.t.} \quad
   & x_1 + x_2 + x_3 \leq 500, \\
   & T_{1s} x_1 + y_{1s} - y_{3s} \geq 200, \quad s=1,2,3, \\
   & T_{2s} x_2 + y_{2s} - y_{4s} \geq 240, \quad s=1,2,3, \\
   & T_{3s} x_3 - y_{5s} - y_{6s} \geq 0, \quad s=1,2,3, \\
   & y_{5s} \leq 6000, \quad s=1,2,3, \\
   & x_1,x_2,x_3 \geq 0 \\
   & y_{1s},y_{2s},y_{3s},y_{4s},y_{5s},y_{6s} \geq 0, \quad s=1,2,3,

where :math:`T_{is}` is an :math:`(i,s)`-element of matrix

.. math::

   T := \left[\begin{array}{ccc}
   3.0 & 3.6 & 24.0 \\
   2.5 & 3.0 & 20.0 \\
   2.0 & 2.4 & 16.0
   \end{array}\right].

Modeling in StochJuMP
^^^^^^^^^^^^^^^^^^^^^

We model the farmer problem by using the StochJuMP package :cite:`huchette2014parallel`. StochJuMP is a scalable algebraic modeling package for stochastic programming problems based on the mathematical programming modeling package JuMP :cite:`bezanson2012julia,lubin2013computing`.

.. note:: StochJuMP enables the generation of large-scale problems in parallel environments and thus overcomes memory and timing bottlenecks. Moreover, it exploits the algebraic modeling capabilities of JuMP to specify problems in a concise format and Julia programming language, which enables the easy handling of data and the use of other tools (e.g., statistical analysis and plotting tools).

The farmer problem can be written as follows.

.. code-block:: julia
   :linenos:
   :caption: Model file: farmer_model.jl

   using StochJuMP
   m = StochasticModel(NS);
   @defVar(m, x[i=CROPS] >= 0, Int);
   @setObjective(m, Min, sum{Cost[i] * x[i], i=CROPS});
   @addConstraint(m, const_budget, sum{x[i], i=CROPS} <= Budget);
   @second_stage m s begin
       sb = StochasticBlock(m, probability[s]);
       @defVar(sb, y[j=PURCH] >= 0);
       @defVar(sb, w[k=SELL] >= 0);
       @setObjective(sb, Min,
           sum{Purchase[j] * y[j], j=PURCH} 
           - sum{Sell[k] * w[k], k=SELL});
       @addConstraint(sb, const_minreq[j=PURCH],
                      Yield[s,j] * x[j] + y[j] - w[j] >= Minreq[j]);
       @addConstraint(sb, const_minreq_beets,
                      Yield[s,3] * x[3] - w[3] - w[4] >= Minreq[3]);
       @addConstraint(sb, const_aux, w[3] <= 6000);
   end

The model file ``farmer_model.jl`` shows the StochJuMP model. In the first line of this script we include ``StochJuMP.jl`` package. The first-stage is defined in lines 3 to 5 and the second stage in lines 7 to 17 for each scenario. The data can be described in a separate file ``farmer_data.jl`` defined as follows.

.. code-block:: julia
   :caption: Data file: farmer_data.jl

   NS = 3;
   CROPS = 1:3;
   PURCH = 1:2;
   SELL  = 1:4;
   probability = [1/3 1/3 1/3];
   Cost = [150 230 260];
   Purchase = [238 210];
   Sell = [170 150 36 10];
   Minreq = [200 240 0];
   Yield = [3.0 3.6 24.0; 2.5 3.0 20.0; 2.0 2.4 16.0];

Solving with DSPsolver.jl
^^^^^^^^^^^^^^^^^^^^^^^^^

We now solve the farmer model by using different DSP methods. The ``DSPsolver.jl`` package is required to run DSP in Julia.

Parallel dual decomposition
***************************

The following file ``farmer_run.jl`` reads the farmer model and runs the dual decomposition method in parallel via the ``MPI.jl`` package.

.. code-block:: julia
   :linenos:
   :caption: Run file: farmer_run.jl

   using DSPsolver, MPI
   MPI.Init();
   include("farmer_data.jl");
   include("farmer_model.jl");
   DSPsolver.loadProblem(m);
   DSPsolver.solve();
   MPI.Finalize();

Line 1 includes the required packages. The MPI library is initialized and fianlized in lines 2 and 7, respectively. The StochJuMP model is given in lines 3 and 4. Note that only two lines of code (5 and 6) are required to invoke the parallel decomposition method. The following command is an example of running ``farmer_run.jl`` with MPI library::

   mpiexec -np 3 julia farmer_run.jl

Benders decomposition
*********************

Alternatively, users can use Benders decomposition by replacing line 6 of ``farmer_run.jl`` with::

   DSPsolver.solve(DSP_SOLVER_BD);

The ``MPI.jl`` pakcage is no longer required for Benders decomposition.

Extensive form solution
***********************

Users can also solve the extensive form of the problem by replacing line 6 of ``farmer_run.jl`` with::

  DSPsolver.solve(DSP_SOLVER_DE);

The ``MPI.jl`` pakcage is no longer required for solving the extensive form.

Reading model in SMPS format
****************************

DSP can also read a model provided in SMPS files :cite:`birge1987standard`. In this format, a model is defined by three files: core, time, and stochastic with file extensions of ``.cor``, ``.tim``, and ``.sto``, respectively. The core file defines the deterministic version of the model with a single reference scenario, the time file indicates a row and a column that split the deterministic data and stochastic data in the constraint matrix, and the stochastic file defines random data. DSP can read model in SMPS format (e.g., ``farmer.cor``, ``farmer.tim`` and ``farmer.sto``) as follows::

   DSPsolver.readSmps("farmer");

.. bibliography:: dsp-manual.bib