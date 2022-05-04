Install the package
===================

Install FWAT-v1.1
-----------------
The FWAT-v1.1 is based on the stable version of `SPECFEM3D <https://github.com/geodynamics/specfem3d/>`_ 
last modified on Aug 30, 2018 with a commit number of 42abac6d18 on Github.


.. code-block:: bash

    $ python S0A_download_ASDF_MPI.py
    $: git clone --recursive https://github.com/geodynamics/specfem3d.git
    $: cd specfem3d
    $: git checkout -b fwat 42abac6d18
    $: cd src
    $: git clone --recursive https://gitlab.com/wangkaim8/fwat.git
    $: mv fwat fullwave_adjoint_tomo
    $: cd ..

    ===> Change setup/constants_tomography.h.in
         logical, parameter :: USE_ALPHA_BETA_RHO = .true.
         logical, parameter :: USE_ALPHA_BETA_RHO_TISO = .false.  ! Kai added
    ===> Change src/shared/shared_par.F90
         logical :: ANISOTROPIC_KL,SAVE_TRANSVERSE_KL,APPROXIMATE_HESS_KL,SAVE_MOHO_MESH
         logical :: SAVE_AZIMUTH_KL ! Kai added

    $: ./configure FC=ifort CC=icc

    ===> Change Makefile:
    1. set MPILIBS = -Lsrc/fullwave_adjoint_tomo/saclibs -lsacio -lsac -mkl
    2. replace
    SUBDIRS = \
            tomography \
            tomography/postprocess_sensitivity_kernels \
    with
    SUBDIRS = \
            fullwave_adjoint_tomo/optimization \
            fullwave_adjoint_tomo \
            fullwave_adjoint_tomo/post_processing \

    3. add
    DEFAULT = \
            fwat_postproc \
            optimization \
            fullwave_adjoint_tomo \
    delete
    all: postprocess tomography
    add:
    all: fwat_postproc optimization fullwave_adjoint_tomo

Notes on segmentation fault/floating
------------------------------------

.. code-block:: bash

    You may need to change some parameters to reduce the memory requirement to avoid "segmentation fault" or set them larger to avoid "floating point exception"
    ===> Change fullwave_adjoint_tomo_par.f90
    LNPT=17
    NDIM = 1000000
    NWINDOWS = 2000

How to Modify the FWAT to fit your version of `SPECFEM3D`
------------------------------------------------------------

If you have to use a different version, your may take
the following steps to modify the FWAT package in order to fit your version
of `SPECFEM3D`.

.. code-block:: bash
 
 step1 : Modify setup_sources_receivers_fwat.f90 and save_adjoint_kernels_fwat.f90
 a) The file setup_sources_receivers_fwat.f90 is modified from the original file
    src/specfem3d/setup_sources_receivers.f90 to use a user-defined a source_fname
    and station_fname instead of the original CMTSOLUTION/FORCESOLUTION and STATIONS.
    You can modify this file according to your version of
    src/specfem3d/setup_sources_receivers.f90
 b) Similary, modify save_adjoint_kernels_fwat.f90 according to your version of
    src/specfem3d/save_adjoint_kernels.f90

 step 2: Change rule.mk if necessary. The FWAT almost call all objects (*.o) that are
         required by the program "xspecfem3D". So refer to src/specfem3D/rule.mk if
         missing some *.o.

 step 3: Copy ../tomography/postprocess_sensitivity_kernels/* to post_processing/.
         Discard the changes in optimization/rule.mk


 step 4: Copy some files in optimization from ../tomography/ that read "proc*external_mesh.bin"
         ==> optimization/compute_kernel_integral.f90 (Change topo to trim(INPUT_DATABASES_DIR)
         ==> optimization/save_external_bin_m_up.f90
         ==> optimization/read_model.f90 (add irregular mesh)
         Modify this file:
         ==> optimization/get_lbfgs_direction.f90 (This one should be carefully tested
         after modification)

         Note FWAT uses some old programs in tomography to update
         the model and also add some additional programs:
         xadd_model_iso_cg
         xadd_model_iso_lbfgs
         xadd_model_tiso_cg

         !!! Check all codes that read and write "proc*external_mesh.bin"
      


Please report issues on the github page or contact the developers.
