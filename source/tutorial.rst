4. Tutorial
==================

This tutorial will teach you how to perform an iterative full waveform inversion with **FWAT** and **SPECFEM3D**.

.. contents::
    :local:
    :depth: 3

1.1 Prepare inputs for FWAT
-----------------------------------

Before running FWAT, the user should first make a new project by copy the folder **example_scripts**.
For example, to build a new seismic project named `FWAT_test`, run the following commands:

.. code-block:: bash
 
 $: mkdir FWAT_test
 $: cd FWAT_test
 $: cp -r ${path_to_specfem3d}/src/fullwave_adjoint_tomo/example_scripts/* .

After that, the user should write their own scripts to make **data** and **src_rec** in
the format described in section 3. The corresponding example scripts are `generate_data .bash` and `mk_forcesolution.bash`.

1.2 Prepare inputs for SPECFEM3D and build mesh
------------------------------------------------
Run the following commands for the preparation of `SPECFEM3D`:

.. code-block:: bash

 $: ln -s  ${path_to_specfem3d}/DATA .
 $: ln -s  ${path_to_specfem3d}/bin .
 $: mkdir OUTPUT_FILES
 $: mkdir OUTPUT_FILES/DATABASES_MPI 

The input files for `SPECFEM3D` should be tested to make sure that they are ready for running any simulations.
This means that the user should set up the meshing files at `DATA/meshfem3D_files`, initial xyz model at 
`DATA/tomo_files` or gll model in a user defined folder, and DATA/Par_file. Set ``APPROXIMATE_HESS_KL = .true.``
and ``USE_RHO_SCALING = .true.`` in Par_file, which will be used in **Stage II**.

For testing if the mesh and initial model is correctly set up, set ``SAVE_MESH_FILE = .true.`` in Par_file and
plot the velocity model `procXXXXXX.bin`` files in `OUTPUT_FILES/DATABASES_MPI`, and set it back to .false. after the test.
Uncomment the following lines in the PBS script `pbs_fwat1_fwd_measure_adj.sh` or SLURM script `sbash_fwat1_fwd_measure_adj.sh`.

.. code-block:: bash

   $: mpirun -np NPROC ./bin/xmeshfem3D
   $: mpirun -np NPROC ./bin/xgenerate_databases
   $: exit

The user should also check `output_mesher.txt` and `output_meshfem3D.txt` for the minimum period resolved,
suggested time step, and other useful information regarding the meshing and solver parameters.

1.3 Prepare **fwat_params**
----------------------------

There are two user-defined parameter files of **FWAT**: `FWAT.PAR` and `MEASUREMENT.PAR`.
The format of `FWAT.PAR` is:

.. code-block:: bash

 ############################### Source ########################################
 ### Always set NSCOMP=1 for tele simulations and ANAT using only vertical components.
 ### NSCOMP>1 is not permitted in v1.1 version.
 NSCOMP: 1
 #
 SCOMPS: Z
 #
 ############################### Receiver ########################################
 NRCOMP: 1
 # in order of Z R T
 RCOMPS: Z
 #
 CH_CODE: BX
 #
 ############################### Filtering ########################################
 NUM_FILTER: 3
 #
 SHORT_P: 6 10 15
 #
 LONG_P: 15 20 30
 ############################### Measuring Window ##################################
 ### Noise FWI only, GROUPVEL_MIN and GROUPVEL_MAX are used to set time windows.
 #
 GROUPVEL_MIN: 2.65 2.65 2.65
 #
 GROUPVEL_MAX: 4.20 4.20 4.20
 #
 ### TeleFWI only, TW_BEFORE and TW_BEFORE are optional to set time windows
 #   (default is to read the time windows from FKtimes.dat) .
 TW_BEFORE: 5
 #
 TW_AFTER: 45
 # Set ADJ_SRC_NORM=.true. for noise FWI if normalizing adjoint sources at
 # different period bands. Always set it .false. for other inversions.
 ADJ_SRC_NORM: .true.
 # Noise FWI only, Set USE_NEAR_OFFSET=.false. to reject EGFs with distances
 # less than one average wavelength
 USE_NEAR_OFFSET: .true.
 # Noise FWI only, Set SUPPRESS_EGF=.false. to convert noise correlations to EGFs.
 # Set it .true. for checkerboard tests of noise FWI.
 SUPPRESS_EGF: .false.
 ################################# Postprocessing ##################################
 # Set SAVE_OUTPUT_EACH_EVENT=.true. to save event kernels;
 SAVE_OUTPUT_EACH_EVENT: .false.
 # Horizontal radius of the Gaussian smoothing function, in meter
 SIGMA_H: 10000
 # Vertical radius of the Gaussian smoothing function, in meter
 SIGMA_V: 10000
 # Optimization method. Options: SD / CG / LBFGS
 OPT_METHOD: SD
 # Preconditioner. Options: default / z_precond
 PRECOND_TYPE: default
 # Maximum step length if not using a line search method
 MAX_SLEN: 0.02
 # Set DO_LS .true. if doing a line search
 DO_LS: .true.
 # Number of trial models for a line search
 NUM_STEP: 5
 # Step lengths of trial models for the line search. Format: F5.3
 STEP_LENS: 0.010 0.020 0.030 0.040 0.050
 #


The format of `MESUREMENT.PAR` is the same as that used in the software ``meaure_adj``. An example is:

.. code-block:: bash

  -2.0 0.025   6000  # tstart, DT, npts: time vector for simulations
                      7  # imeas (1-8; see manual)
                     BX  # channel: BH or LH
      30.000      6.000  # TLONG and TSHORT: band-pass periods for records
                .false.  # RUN_BANDPASS: use band-pass on records
                .false.  # DISPLAY_DETAILS
                .false.  # OUTPUT_MEASUREMENT_FILES
                 .true.  # COMPUTE_ADJOINT_SOURCE
     -4.5000     4.5000  # TSHIFT_MIN; TSHIFT_MAX
     -1.5000     1.5000  # DLNA_MIN; DLNA_MAX
                  0.800  # CC_MIN
                      1  # ERROR_TYPE -- 0 none; 1 CC, MT-CC; 2 MT-jack-knife
                  1.000  # DT_SIGMA_MIN
                  0.500  # DLNA_SIGMA_MIN
                      1  # ITAPER -- taper type: 1 multi-taper; 2 cosine; 3 boxcar
            0.020  2.50  # WTR, NPI (ntaper = 2*NPI)
                  2.000  # DT_FAC
                  2.500  # ERR_FAC
                  3.500  # DT_MAX_SCALE
                  1.500  # NCYCLE_IN_WINDOW
                 .false. # USE_PHYSICAL_DISPERSION

.. note::

   Remember to change ``tstart``, ``DT``, ``npts``, ``imeas``, ``channel``, ``TSHIFT_MIN/TSHIFT_MAX``, ``DLNA_MIN/DLNA_MAX``, ``CC_MIN``, ``ITAPER``.

   For FWAT of different data types (``noise``/``tele``), I recommend to use different parameter files, such as 
   `FWAT.PAR.noise/FWAT.PAR.tele`, `MEASUREMENT.PAR.noise/MEASUREMENT.PAR.tele`.

1.4 Running forward and adjoint simulations (Stage I)
-------------------------------------------------------

As long as the SEM mesh/model is set up, comment out the following lines in the PBS script 
`pbs_fwat1_fwd_measure_adj.sh` or SLURM script `sbash_fwat1_fwd_measure_adj.sh`.

.. code-block:: bash

   #mpirun -np NPROC ./bin/xmeshfem3D
   #mpirun -np NPROC ./bin/xgenerate_databases
   #exit

and run the script `submit_job_fwat1.sh`. We should first set the model name (such as mod=M01), and 
step length (such as step=0.02) for reading updated model from previous iterations.

.. code-block:: bash
 
 Usage: ./submit_job_fwat1.bash M?? simu_type setb sete
 M??       --- name of the current model, such as M00, M01, ..., etc.
 simu_type --- simulation type: noise, tele, or leq.
 setb      --- begin number of sources_set, such as 1, 2, ..., etc.
 sete      --- end number of sources_set, such as 1, 2, ..., etc.

This script will call `pbs_fwat1_fwd_measure_adj.sh` or `sbash_fwat1_fwd_measure_adj.sh` to submit a PBS/SLURM job
for each event set, in which the MPI program `xfwat1_fwd_measure_adj` is called to run the forward and adjoint simulations.

.. code-block:: bash

 USAGE:  mpirun -np NPROC bin/xfwat1_fwd_measure_adj model set simu_type run_opt
 model     --- name of current model, such as M00, M01, ...
 set       --- name of event set, such as set1, set2, ...
 simu_type --- simulation type: noise, tele, leq
 run_opt   --- run_opt= 1 (fwd), 2 (fwd_meas), 3 (fwd_meas_adj)

Note the program has an option (``run_opt``) to run forward simulations only (``run_opt=1``), or with measuring misfit and
adjoint sources (``run_opt=2``), and adjoint simulations (``run_opt=3``). The program can be divided into three parts:

1. Making forward simulation directory. The program will make all forward simulation directories under the 
   directory **solver** (Figure 2) by looping over event sets defined by a variable ipart according to the file 
   sources_setXX.dat. 

2. Running `run_preprocessing.f90`. We adopt a multi-scale strategy in misfit measurement, thus the author 
   should choose how many and what frequency bands needed in current iteration depending on the data,
   then make changes to FWAT.PAR. Note in file `run_preprocessing.f90`, we have three options (``simu_type``) to
   measure misfits and adjoint sources for data sets of noise, tele and leq. The misfits are stored in the 
   directory **misfits** as shown in Figure 2.

3. Run adjoint simulations to obtain event kernels, sum and save them into one event gradient. In the `FWAT.PAR`,
   we have an option (``SAVE_OUTPUT_EACH_EVENT``: ``.true.``) to save each event kernel.

1.5 Post-processing and model update (Stage II)
-------------------------------------------------------
Then, run `submit_job_fwat2.sh` to do post-processing on event kernels and obtain the total misfit gradient and update the model.

.. code-block:: bash

 Usage: ./submit_job_fwat2.bash M?? setb sete
 M??       --- name of the current model, such as M00, M01, ..., etc.
 setb      --- begin number of sources_set, such as 1, 2, ..., etc.
 sete      --- end number of sources_set, such as 1, 2, ..., etc.

This script will call  `pbs_fwat2_postproc_opt.sh` or `sbash_fwat2_postproc_opt.sh` to submit a PBS/SLURM job
for a number of event sets, in which the MPI program `xfwat2_postproc_opt` is called to run postprocessing and optimization.

.. code-block:: bash

 USAGE:  mpirun -np NPROC bin/xfwat2_postproc_opt model setb sete is_smooth
  model     --- name of current model, such as M00, M01, ...
  setb      --- name of beginning event set, such as set1, set2, ...
  sete      --- name of  ending event set (sete>=setb), such as set1, set2, ...
  is_smooth --- true or false, whether to do kernel smoothing or not.

This program can be divided into three parts:

- Summation and precondition of event kernels to obtain the final misfit gradient. The program will apply
  a preconditioner (``PRECOND_TYPE``: ``default``/``z_precond`` in `FWAT.PAR`) to each event kernel and sum all 
  preconditioned event kernels to obtain the final misfit gradient in the directory `./optimize/SUM_KERNELS_M??`
  (Figure 2). Gradient files are in the format of `proc000???_alpha_kernel.bin`, `proc000???_beta_kernel.bin`,
  `proc000???_rhop_kernel.bin`. I would recommend using  PRECOND_TYPE: ``default`` for ANAT and 
  PRECOND_TYPE: ``z_precond`` for TeleFWI.

- A 3-D Gaussian smoothing of the misfit gradient. Smoothed gradients are saved in the directory `./optimize/SUM_KERNELS_M??`  with the format of `proc000???_alpha_kernel_smooth.bin`, `proc000???_beta_kernel_smooth.bin`, `proc000???_rhop_kernel_smooth.bin`.

- Model update. There are two ways to update the model by either using a fixed step length (``DO_LS``: ``.false.``) or an optimal
  step length based on a line search method (``DO_LS``: ``.true.``).

.. note::

 If you choose ``DO_LS``: ``.false.``, then the model will be updated by a fix step length using ``MAX_SLEN``. The output of the
 new model will be saved in the directory `./optimize/MODEL_M??`. Then, go back to **Stage I** and II to run the next iteration.

 If you choose ``DO_LS``: ``.true.``, the program will generate a number of trial models according the ``STEP_LENS`` such as
 `./optimize/MODEL_M??_step0.020`, `./optimize/MODEL_M??_step0.040`, etc. Then, you should run a line search in **Stage III**.


1.6 Line search for optimal step length (Stage III)
-----------------------------------------------------

After all trial models are generated (``DO_LS``: ``.true.``), we use them to do a line search in order to obtain the optimal step length.
The script responsible for this is `submit_job_fwat3.sh`.

.. code-block::

 Usage: ./submit_job_fwat3.bash M?? simu_type
 M??       --- name of the current model, such as M00, M01, ..., etc.
 simu_type --- simulation type: noise, tele, or leq.

This script will call  `pbs_fwat3_linesearch.sh` or `sbash_fwat3_linesearch.sh` to submit a PBS/SLURM job for each event set,
in which the MPI program `xfwat3_linesearch` is called to run the forward simulations and misfit measuring.

.. code-block::

 USAGE:  mpirun -np NPROC bin/xfwat3_linesearch model set simu_type
 model     --- name of current model, such as M00_step0.020, M00_step0.040, ...
 set       --- name of event set, set it to "ls".
 simu_type --- simulation type: noise, tele, leq

The program will make all forward simulation directories by looping over step lengths, and each of them has a 
selected number of events from the file `sources_ls.dat`. The data processing is similar to `xfwat1_fwd_measure_adj`
but with only forward simulations and misfit measuring.

As long as all the forward simulations for different trial models are finished, the corresponding traveltime misfits
are collected in the directory **outputs/M??**. Then, we use the script `plt_line_search.mtltiband.ANAT.bash` to plot
the total misfit curve over step length and choose the optimal step length. After that, assign the trial model with the
optimal step length (such as `./optimize/MODEL_M00_step0.040`) to our new model (`./optimize/MODEL_M01`) for 
running the next iteratoin. 
