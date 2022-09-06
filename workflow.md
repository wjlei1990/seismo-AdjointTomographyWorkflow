Adjoint tomography includes following steps:

1. forward simulation: generate synthetic seismograms
2. data processing: use raw seismograms (observed and synthetic) to generate adjoint sources
3. adjoint simulation: kernel calculation
4. post-processing: compute gradient and search direction
5. line search: search to optimal step length

---
### 1. Forward Simluation
Set the `save_forward=True` in `DATA/Par_file`.

In the end, you should get all the `synthetic.h5` for each earthquake event. The wavefield snapshot should be saved in the run directory
of each earthquake.

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

### 3. Adjoint Simulation

You need to copy the `adjoint.h5` and `STATIONS_ADJOINT` (generated from the last step) to the forward run directory.

Change the `DATA/Par_file` by setting:
```
simulation_type=3
```

Also the adjoint simulation would take longer time compared to the forwardi simulation. So remember to change the walltime.

### 4. Post-processing
The post-processing includes following steps:




