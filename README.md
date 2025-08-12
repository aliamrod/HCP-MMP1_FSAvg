# HCP-MMP1_FSAvg
This repository resamples HCP-MM1 atlas from HCP fs_LR (32k) to FreeSurfer fsaverage and fsaverage5. Includes lh/rh .annot files and commands for label extraction, spherical registration, and downsampling using Connectome Workbench and FreeSurfer.

* HCP dlabel (fs_LR 32k) → split to L_hemisphere/R_hemisphere label.gii
* Resample fs_LR 32k → fsaverage 164l (HCP's standard)
* Convert to FreeSurfer .annot on fsaverage
* Downsample fsaverage → fsaverage5 via mri_surf2surf
* Results in lh.HCP-MMP1.fsaverage5.annot and rh.HCP-MMP1.fsaverage5.annot for fs5 samples.

**Required files**
* HCP MMP1.0 dlabel.nii (from [BALSA](https://balsa.wustl.edu/WN56))
* Sphere registration files (from [HCP GitHub](https://github.com/Washington-University/HCPpipelines/tree/master/global/templates/standard_mesh_atlases))

**Final Outputs**
* lh.HCP-MMP1.fsaverage5.annot
* rh.HCP-MMP1.fsaverage5.annot
These can be used in FreeSurfer or nibabel for analysis. 

**Dependencies**
* Connectome Workbench (wb_command)
* FreeSurfer (mris_convert, mri_surf2surf)
***

HCP MMP atlas is originally in `fs_LR 32k` mesh (Human Connectome Project's surface format), while FreeSurfer utilizes `fsaverage` (high-res, ~164k vertices/hemisphere) and `fsaverage5` (low-res, ~10,242 vertices/hemisphere). Thus, resampling first to _fsaverage_ ensures proper alignment between HCP and FreeSurfer coordinate systems because their spherical mappings differ. _fsaverage_ acts as a bridge -- _mri_surf2surf_ is optimized for downsampling from _fsaverage_ → _fsaverage5_, and skipping this step risks misalignmnet. Sphere files (e.g., `L.sphere.32k_fs_LR.surf.gii`) define vertex mappings between surfaces, enabling accurate interpolation (e.g., barycentric warping) by ensuring anatomical correspondence (e.g., V1 in HCP matches V1 in FreeSurfer). While direct _fs_LR_ → _fsaverage5_ conversion is possible, it may distort parcels due to registration mismatches. 
