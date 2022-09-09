Adjoint tomography includes following steps:

1. forward simulation: generate synthetic seismograms
2. data processing: use raw seismograms (observed and synthetic) to generate adjoint sources
3. adjoint simulation: kernel calculation
4. post-processing: compute gradient and search direction
5. line search: search to optimal step length

---
### 1. Forward Simluation
In `DATA/Par_file` set
```
save_forward                    = .true.
SAVE_SOURCE_MASK                = .true.
```

In the end, you should get all the `synthetic.h5` for each earthquake event. The wavefield snapshot should be saved in the run directory
`DATABASES_MPI`of each earthquake. The source mask is used in the post-procesing to remove the source effect.

A example of scripts to setup the forward simulation could be found here:
`/ccs/proj/geo111/wenjie/AdjointTomography/M32/forward`

`nnodes` script could be found here:
`/ccs/proj/geo111/wenjie/AdjointTomography/M32/forward/nnodes_forward`

For specfem settings, it could be found here:
`/ccs/proj/geo111/wenjie/AdjointTomography/M32/forward/specfem3D_globe`


---
### 2. Data Processing

A full data processing workflow includes:
1. ProcessObserved
2. ProcessSynthetic
3. CreateWindows                                  
4. MeasureAdjoint
5. FilterWindows
6. CreateWeights                                       
7. CreateAdjointSource
8. SumAdjoint
9. CreateAdjointStation

However, if the observed data doesn't change from the last iteration, you may reuse some results from the last iterations,
and just run part of the workflow. The simplied workflow could be:
1. ProcessSynthetic: process new synthetic data with the new iteration of model                              
2. MeasureAdjoint: make measurements using filtered window (for misfit calculation)                                     
3. CreateAdjointSource
4. SumAdjoint
5. CreateAdjointStation

##### Step 2: MeasureAdjoint
To calculate misfit values using `pyadjoint` and save them to json files.
The misfit values will be used for calculate the overall misfit value. The overall misfit values
will be used to check the misfit decrease in every iterations.

For simplied workflow (workflow with 5 steps only), please use the filtered window files so we only make measurements for filtered windows.

##### Step 5: CreateAdjointStation
Theoretically we don't need the step 5, which is to create the "STATIONS_ADJOINT" file to the adjoint simulation. However, to prevent
from the missing adjoint components in the final adjoint file (maybe due to the numerical instability in the adjoint calculation), we
can generate the adjoint stations again.

A example of scripts to setup the simplied data processing workflow using `simpy` could be found here:
`/ccs/proj/geo111/wenjie/AdjointTomography/M32/simpy`

The job config file could be found here:
`/ccs/proj/geo111/wenjie/AdjointTomography/M32/simpy/configs`

The parameter files for data processing could be found here:
`/ccs/proj/geo111/wenjie/AdjointTomography/M32/simpy/params`

The path files for simplifiled data processing workflow could be found here:
`/ccs/proj/geo111/wenjie/AdjointTomography/M32/simpy/paths`

---
### 3. Adjoint Simulation

You need to copy the `adjoint.h5` and `STATIONS_ADJOINT` (generated from the last step) to the forward run directory.

Change the `DATA/Par_file` by setting:
```
simulation_type=3
```

Also the adjoint simulation would take longer time compared to the forwardi simulation. So remember to change the walltime.

---
### 4. Post-processing
The post-processing includes following steps:

0. Apply source mask: `/ccs/proj/geo111/wenjie/AdjointTomography/M32/post_proc/KernelProcessing/examples/src_mask`
1. sum kernels: sum each kernels by different weightings: `/ccs/proj/geo111/wenjie/AdjointTomography/M32/post_proc/KernelProcessing/01_sum_kerenels.bash`
2. smooth kernels: the executable is located in the SPECFEM3D_GLOBE package and use laplacian smoothing `xsmooth_laplacian_sem_adios`. `/ccs/proj/geo111/wenjie/AdjointTomography/M32/forward/specfem3D_globe/job_smooth.bash`
3. merge kernels: merge indiviual kernel files for each parameter into one single big kernel files. `/ccs/proj/geo111/wenjie/AdjointTomography/M32/post_proc/KernelProcessing/03_pbs_merge_kernel.bash`
4. Prepare hessian kernels: you may reuse the hessian kernels from the last iteration if you are using L-BFGS method.  If you are just using rho kernel, then you don't need this step. If you are using vp and vs hessian kernels, refer to this step: `/ccs/proj/geo111/wenjie/AdjointTomography/M32/post_proc/KernelProcessing/04_pbs_compute_bulkc_beta_hess.bash`
5. Apply hessian kernels, or precondition kernels: `/ccs/proj/geo111/wenjie/AdjointTomography/M32/post_proc/KernelProcessing/05_pbs_precond_bulkc_beta_hess.bash`
6. Compute Search direction:
    * steep descent: `/ccs/proj/geo111/wenjie/AdjointTomography/M32/post_proc/KernelProcessing/06_pbs_sd_direction.bash`
    * conjugate gradient
    * L-BFGS: `/ccs/proj/geo111/wenjie/AdjointTomography/M32/post_proc/KernelProcessing/examples/lbfgs.M31`
7. Update model for various step length: apply the search direction onto the old model with different step length and get the new model. `/ccs/proj/geo111/wenjie/AdjointTomography/M32/post_proc/KernelProcessing/07_pbs_update_model.bash`

---
### 5. Line Search

This step is to search the optimal step length, and find  the model with minimum overal misfit.

You have to launch forward simulation with a subset of earthquake events, processing the synthetic data, and make measurements.

In this stage, for the forward simulation, you don't need to save the forward wavefield, since there is no need to conduct adjoint simulation. Example scripts are located: `/ccs/proj/geo111/wenjie/AdjointTomography/M32/line_search/line_search.forward`

For the data processing, it only contains two steps:
1. Process Synthetic (for models with different step lengths)
2. Measure Adjoint: compute the misfit for each window

Example scripts for data processing are located: `/ccs/proj/geo111/wenjie/AdjointTomography/M32/line_search/measurements/simpy`

Then you need to sum the misfit to get overal misfit for the subset earthquake events: `/ccs/proj/geo111/wenjie/AdjointTomography/M32/line_search/measurements/misfit`

---
### 6. Other notes

1. Iteration scripts are located: `/ccs/proj/geo111/wenjie/AdjointTomography`. You can check each iteration, for example, `/ccs/proj/geo111/wenjie/AdjointTomography/M32`

2. When you have a new model, you will need to compute the overal misfit values for the overall database, so you can track how the overall misfit
changes between iterations. Example scripts are `/ccs/proj/geo111/wenjie/AdjointTomography/compare_iterations/misfit/misfit.M32` and plotting 
overall misfit scripts are here: `/ccs/proj/geo111/wenjie/AdjointTomography/compare_iterations/misfit/plot_misfit`.



