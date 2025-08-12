# HCP-MMP1_FSAvg
This repository resamples HCP-MM1 atlas from HCP fs_LR (32k) to FreeSurfer fsaverage and fsaverage5. Includes lh/rh .annot files and commands for label extraction, spherical registration, and downsampling using Connectome Workbench and FreeSurfer.

* HCP dlabel (fs_LR 32k) → split to L_hemisphere/R_hemisphere label.gii
* Resample fs_LR 32k → fsaverage 164k (HCP's standard)
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

```bash 
#!/usr/bin/env bash
#SBATCH --job-name=build_mmp_fs5
#SBATCH --output=build_mmp_fs5_%j.out
#SBATCH --error=build_mmp_fs5_%j.err
#SBATCH --time=01:00:00
#SBATCH --cpus-per-task=1
#SBATCH --mem=4G

set -euo pipefail

# -------------------------------
# 0) EDIT THESE PATHS (once)
# -------------------------------

# FreeSurfer subjects dir (must contain fsaverage and fsaverage5)
export SUBJECTS_DIR="${SUBJECTS_DIR:-$HOME/freesurfer_subjects}"

# HCP Glasser dlabels (fs_LR 32k) — put these files in CWD or set full paths
HCP_L="${HCP_L:-Q1-Q6_RelatedParcellation210.L.CorticalAreas_dil_Colors.32k_fs_LR.dlabel.nii}"
HCP_R="${HCP_R:-Q1-Q6_RelatedParcellation210.R.CorticalAreas_dil_Colors.32k_fs_LR.dlabel.nii}"

# Source spheres (fs_LR 32k) — from the BALSA fsaverage_LR32k package
SRC_SPH_LR32K_L="${SRC_SPH_LR32K_L:-L.sphere.32k_fs_LR.surf.gii}"
SRC_SPH_LR32K_R="${SRC_SPH_LR32K_R:-R.sphere.32k_fs_LR.surf.gii}"

# Target spheres (fsaverage 164k, HCP standard) — we’ll auto-download if missing
TGT_SPH_FSAVG164_L="fs_L-to-fs_LR_fsaverage.L_LR.spherical_std.164k_fs_L.surf.gii"
TGT_SPH_FSAVG164_R="fs_R-to-fs_LR_fsaverage.R_LR.spherical_std.164k_fs_R.surf.gii"

# Outputs you’ll get (don’t edit)
LH_LABEL_32K="Q1Q6.L.32k.label.gii"
RH_LABEL_32K="Q1Q6.R.32k.label.gii"
LH_LABEL_FSAVG164="left.fsaverage164.label.gii"
RH_LABEL_FSAVG164="right.fsaverage164.label.gii"
LH_ANNOT_FSAVG="lh.HCP-MMP1.annot"
RH_ANNOT_FSAVG="rh.HCP-MMP1.annot"
LH_ANNOT_FS5="lh.HCP-MMP1.fsaverage5.annot"
RH_ANNOT_FS5="rh.HCP-MMP1.fsaverage5.annot"

# -------------------------------
# 1) Tools check (load modules here if needed)
# -------------------------------
command -v wb_command   >/dev/null || { echo "ERROR: wb_command not found in PATH"; exit 1; }
command -v mris_convert >/dev/null || { echo "ERROR: mris_convert not found (load FreeSurfer)"; exit 1; }
command -v mri_surf2surf >/dev/null || { echo "ERROR: mri_surf2surf not found (load FreeSurfer)"; exit 1; }

# -------------------------------
# 2) Download target fsaverage 164k spheres if missing (from HCP templates)
# -------------------------------
if [[ ! -f "$TGT_SPH_FSAVG164_L" ]]; then
  echo "[DL] $TGT_SPH_FSAVG164_L"
  curl -L -o "$TGT_SPH_FSAVG164_L" \
    "https://raw.githubusercontent.com/Washington-University/HCPpipelines/master/global/templates/standard_mesh_atlases/resample_fsaverage/$TGT_SPH_FSAVG164_L"
fi
if [[ ! -f "$TGT_SPH_FSAVG164_R" ]]; then
  echo "[DL] $TGT_SPH_FSAVG164_R"
  curl -L -o "$TGT_SPH_FSAVG164_R" \
    "https://raw.githubusercontent.com/Washington-University/HCPpipelines/master/global/templates/standard_mesh_atlases/resample_fsaverage/$TGT_SPH_FSAVG164_R"
fi

# -------------------------------
# 3) Verify required inputs exist
# -------------------------------
for f in "$HCP_L" "$HCP_R" "$SRC_SPH_LR32K_L" "$SRC_SPH_LR32K_R" \
         "$TGT_SPH_FSAVG164_L" "$TGT_SPH_FSAVG164_R"; do
  [[ -f "$f" ]] || { echo "ERROR: Missing file: $f"; exit 1; }
done

# -------------------------------
# 4) dlabel (fs_LR 32k) → hemi label.gii (fs_LR 32k)
# -------------------------------
echo "[1/5] Extracting LH/RH labels from dlabel..."
wb_command -cifti-separate "$HCP_L" COLUMN -label CORTEX_LEFT  "$LH_LABEL_32K"
wb_command -cifti-separate "$HCP_R" COLUMN -label CORTEX_RIGHT "$RH_LABEL_32K"

# -------------------------------
# 5) label.gii: fs_LR 32k → fsaverage 164k (HCP std)
# -------------------------------
echo "[2/5] Resampling labels to fsaverage 164k..."
wb_command -label-resample \
  "$LH_LABEL_32K" \
  "$SRC_SPH_LR32K_L" \
  "$TGT_SPH_FSAVG164_L" \
  BARYCENTRIC "$LH_LABEL_FSAVG164" -largest

wb_command -label-resample \
  "$RH_LABEL_32K" \
  "$SRC_SPH_LR32K_R" \
  "$TGT_SPH_FSAVG164_R" \
  BARYCENTRIC "$RH_LABEL_FSAVG164" -largest

# -------------------------------
# 6) fsaverage 164k label.gii → FreeSurfer .annot on fsaverage
#    (mris_convert needs the matching surf geometry for each hemi)
# -------------------------------
echo "[3/5] Converting fsaverage164 label.gii → FreeSurfer .annot on fsaverage..."
mris_convert --annot "$LH_LABEL_FSAVG164" "$TGT_SPH_FSAVG164_L" "./$LH_ANNOT_FSAVG"
mris_convert --annot "$RH_LABEL_FSAVG164" "$TGT_SPH_FSAVG164_R" "./$RH_ANNOT_FSAVG"

# -------------------------------
# 7) fsaverage .annot → fsaverage5 .annot (nearest-neighbor)
# -------------------------------
echo "[4/5] Downsampling fsaverage → fsaverage5..."
mri_surf2surf \
  --srcsubject fsaverage --trgsubject fsaverage5 \
  --hemi lh --srcsurfreg sphere.reg --trgsurfreg sphere.reg \
  --sval-annot "$LH_ANNOT_FSAVG" --tval "./$LH_ANNOT_FS5" \
  --mapmethod nnf

mri_surf2surf \
  --srcsubject fsaverage --trgsubject fsaverage5 \
  --hemi rh --srcsurfreg sphere.reg --trgsurfreg sphere.reg \
  --sval-annot "$RH_ANNOT_FSAVG" --tval "./$RH_ANNOT_FS5" \
  --mapmethod nnf

# -------------------------------
# 8) Sanity checks
# -------------------------------
echo "[5/5] Sanity check (.annot present on fsaverage5/ matching vertex counts not printed here)"
ls -lh "$LH_ANNOT_FS5" "$RH_ANNOT_FS5"
echo "DONE. Outputs:"
echo "  $LH_ANNOT_FS5"
echo "  $RH_ANNOT_FS5"
