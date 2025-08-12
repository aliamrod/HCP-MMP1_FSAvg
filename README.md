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
