---
layout: default
title: "fMRI-barebones"
---


# fMRI-barebones **tutorial** 

---

### Contents:


1. [Setup]({{site.baseurl}}/howto.html)
2. [Cortical surface reconstruction using Freesurfer]({{site.baseurl}}/howto.html#2)
3. [Obtain Freesurfer retinotopic templates and atlases for cortical visual regions using Neuropythy]({{site.baseurl}}/howto.html#3)
4. [Skull stripping using FSL (optional)]({{site.baseurl}}/howto.html#4)
5. [Linear registration to standard space using FSL]({{site.baseurl}}/howto.html#5)
6. [Non-linear registration to standard space using FSL]({{site.baseurl}}/howto.html#6)
7. [Functional to structural co-registration using FSL]({{site.baseurl}}/howto.html#7)
8. [Functional to structural co-registration using Freesurfer (optional)]({{site.baseurl}}/howto.html#8)
9. [Remove motion artifacts using ICA-AROMA]({{site.baseurl}}/howto.html#9)
10. [Obtain BOLD time series from ROIs in FSL atlases]({{site.baseurl}}/howto.html#10)
11. [Create .MGZ files with surface-node X ROI parameter/time series]({{site.baseurl}}/howto.html#11)
12. [Project data to the cortical surface reconstruction]({{site.baseurl}}/howto.html#12)
13. [Import Freesurfer data into Pycortex]({{site.baseurl}}/howto.html#13)
14. [Compute cortical (geodesic) distances in millimeters using Pycortex]({{site.baseurl}}/howto.html#14)
15. [Load data in Python]({{site.baseurl}}/howto.html#15)


---

<a name="1"></a>

## **1.-** Setup 

In its present form, fMRI-barebones is a minimal fMRI preprocessing pipeline implemented in terminal using shell scripts (files containing a series of commands). At the moment it works in OSX, but should be easily adapted to Linux. To succesfully edit and run fMRI-barebones template scripts, you need FSL, Freesurfer and a Python environment (with the correct depencies for ICA-AROMA, Neuropythy and Pycortex) correctly installed, and Matlab/Jupyter (for analytics and visualzation). 
  
As input, you need MRI data: 1) a structural nifti file (T1.nii.gz) and 2) functional runs named run001.nii.gz, run002.nii.gz, etc. It outputs: 1) cleaned single-subject functional data in native volumetric space (as cleanfunc.nii.gz) and its associated cortical surface reconstruction (func2surf_lh.mgz and func2surf_rh.mgz). 2) retinotopic template maps for early visual cortex (Benson). 3) probabilistic templates for early and higher and visual cortical regions (Wang-Kastner). 4) optional: geodesic distance between cortical surface nodes in millimiters. 


---

<a name="2"></a>

## **2.-** Cortical surface reconstruction using Freesurfer

This is an important and useful step commonly used in human visual neuroscience to study topographically organized cortical regions (rather than just tissue un-specific blobs in volumetric voxel space). It accurately segments brain tissues and reconstructs individual cortcal surfaces.

Jump the next step in case you are interested in obtaining BOLD data in volumetric voxel space only (e.g. from FSL atlases).

The following commands assume you have your T1 file inside a folder "anat", inside your subject data folder (e.g. ".../S15/anat/").


```shell
export FREESURFER_HOME=/Applications/freesurfer
source $FREESURFER_HOME/SetUpFreeSurfer.sh
export data_folder=/Users/nicogravel/Documents/fMRI/Tutorials/fMRI-barebones_mwe

# Reconstruct cortical surface
export subjnum=15
export subjid=S${subjnum}
export pth=${data_folder}/${subjid}
export t1file=T1_orig.nii
recon-all -s ${subjid} -i ${pth}/anat/${t1file} -all -3T
```


> Note: It is recommended to pre-align the original T1 to AC-PC coordinates. This helps Freesurfer find the center of mass of the head improving recon-all performance (use the flag -3T if you are preprocessing 3T data).

---

<a name="3"></a>


## **3.-** Obtain Freesurfer retinotopic templates and atlases for cortical visual regions using Neuropythy


```shell
conda create -n neuroConda python=2.7

source activate neuroConda
pip install neuropythy nipype numpy matplotlib ipyvolume

# Benson and Wang-Kasnter Atlases
python -m neuropythy benson14_retinotopy --verbose S$subjnum 
python -m neuropythy atlas --verbose S$subjnum 

deactivate neuroConda
```

---

<a name="4"></a>

## **4.-** Skull stripping using FSL (optional)


A decently stripped brain is mandatory for for the following steps to work. One possibility is to take the stripped brains from Freesurfer (if it has been run previously), it does a better job than BET.

It is recommended to do intensity normalization using Freesurfer before using FSL's tool BET. 
The original T1 typically has signal intensity gradients and dropouts. A solution is to use fieldmaps to correct for the artifacts produced the magnetic field inhomogeneity. Fieldmaps are not always available. In this case, a simple N3 normalization improves things a bit.

```shell
export FREESURFER_HOME=/Applications/freesurfer
source $FREESURFER_HOME/SetUpFreeSurfer.sh
export data_folder=/Users/nicogravel/Documents/fMRI/Tutorials{{site.baseurl}}_mwe
export subjnum=15
export subjid=S${subjnum}
export pth=${data_folder}/${subjid}
export t1file=T1_orig.nii

mri_convert ${pth}/anat/${t1file} ${pth}/anat/T1.mgz
mri_nu_correct.mni --i ${pth}/anat/T1.mgz --o ${pth}/anat/T1_N3.mgz --n 2
mri_normalize -g 1 -mprage ${pth}/anat/T1_N3.mgz ${pth}/anat/T1_norm.mgz
mri_convert ${pth}/anat/T1_norm.mgz ${pth}/anat/T1_norm.nii.gz

bet ${pth}/anat/T1_norm.nii.gz ${pth}/anat/T1_norm_brain -f 0.5 -R -B
```

![orig]({{site.baseurl}}/assets/orig.png)
![n3norm]({{site.baseurl}}/assets/n3norm.png)

> Note: It is also recommended to pre-align the T1 to ACPC coordinates. This helps BET find the center of mass of the head improving its performance considerably.


---

<a name="5"></a>


## **5.-** Linear registration to standard space using FSL

The following snippet shows how to do a structural to standard linear registration using FLIRT on the outputs of Freesurfer recon-all. This is preferable and more accurate than using the output from FSL's BET (which may require extra tweaking).

```shell
mri_convert ${FREESURFER_HOME}/subjects/S$subjnum/mri/T1.mgz ${pth}/anat/T1.nii.gz
mri_convert ${FREESURFER_HOME}/subjects/S$subjnum/mri/brain.mgz ${pth}/anat/brain.nii.gz
mri_convert ${FREESURFER_HOME}/subjects/S$subjnum/mri/brainmask.mgz ${pth}/anat/brainmask.nii.gz
```

The following snippet shows how to reorient Freesurfer's recon-all outputs to the FSL coordinate system. This is important, since FSL algorithms depend on FSL's own coordinate system to work.

```shell
fslswapdim ${pth}/anat/T1.nii.gz z -x -y ${pth}/anat/T1.nii.gz
fslswapdim ${pth}/anat/brain.nii.gz z -x -y ${pth}/anat/brain.nii.gz
fslswapdim ${pth}/anat/brainmask.nii.gz z -x -y ${pth}/anat/brainmask.nii.gz
fslreorient2std ${pth}/anat/T1.nii.gz
fslreorient2std ${pth}/anat/brain.nii.gz
fslreorient2std ${pth}/anat/brainmask.nii.gz
```

The following snippet shows how to run FLIRT to register the preprocessed T1 to standard MNI152 space. Creates the affine matrix transform needed later by FNIRT and other steps.


```shell
# Make folder to store the registration files:
mkdir ${pth}/reg

# Run FLIRT:
flirt -ref ${FSLDIR}/data/standard/MNI152_T1_2mm_brain.nii.gz -in ${pth}/anat/brain \
-out ${pth}/reg/T1toMNIlin.nii.gz -omat ${pth}/reg/T1toMNIlin.mat -cost normcorr \
-dof 12 -interp spline #-refweight -searchrx -1 1 -searchry -1 1 -searchrz -1 1 # -in ${pth}/anat/brain -nosearch 

# Check if results are going well:
#fsleyes ${FSLDIR}/data/standard/MNI152_T1_2mm_brain ${pth}/reg/T1toMNIlin 
```

This gives the affine matrix: T1toMNIlin.mat

---

<a name="6"></a>


## **6.-** Non-linear registration to standard space using FSL

This step depends on the affine matrix previously obtained. 

```shell
export struct=T1 

fnirt --in=${pth}/anat/${struct} --aff=${pth}/reg/T1toMNIlin.mat --cout=${pth}/reg/T1toMNI_coef.nii.gz \
--fout=${pth}/reg/T1toMNI_warp.nii.gz --ref=${FSLDIR}/data/standard/MNI152_T1_2mm.nii.gz \
--refmask=${FSLDIR}/data/standard/MNI152_T1_2mm_brain_mask_dil.nii.gz --config=T1_2_MNI152_2mm.cnf
```

This gives the files T1toMNI_coef.nii.gz and T1toMNI_warp.nii.gz

To see if things are going well, we apply the tranform to the stripped brain and visualize the results using fsleyes: 


```shell
applywarp -i ${pth}/anat/brain -r $FSLDIR/data/standard/MNI152_T1_2mm_brain.nii.gz \
-w ${pth}/reg/T1toMNI_warp.nii.gz --interp=nn -o ${pth}/reg/T1toMNInonlin_brain.nii.gz

# Visualize results:
#fsleyes ${FSLDIR}/data/standard/MNI152_T1_2mm ${pth}/reg/T1toMNIlin ${pth}/reg/T1toMNInonlin_brain 
```

---

<a name="7"></a>

## **7.-** Functional to structural co-registration using FSL

```shell
export FREESURFER_HOME=/Applications/freesurfer
source $FREESURFER_HOME/SetUpFreeSurfer.sh
export data_folder=/Users/nicogravel/Documents/fMRI/Tutorials/fMRI-barebones_mwe

export subjnum=15
export subjid=S${subjnum}
export pth=${data_folder}/${subjid}

export file=rest_YYYYMMDD_01.nii
export scan=run01
export cond=01

# Reorient functional volumes:

fslreorient2std ${pth}/func/${file} ${pth}/func/${scan}

# Discard first volumes:

export discard=0
export keep=210
fslroi ${pth}/func/${scan} ${pth}/func/${scan} ${discard} ${keep}

# Motion correction:

mcflirt -in ${pth}/func/${scan} -out ${pth}/func/${scan}_mcf -plots -refvol 1 -stats -report 

fslmaths ${pth}/func/${scan} -s 6 ${pth}/func/${scan}_smoothed

fsl_motion_outliers -i ${pth}/func/${scan} -o ${pth}/func/temp -s ${pth}/func/${scan}_fd.txt --fdrms
rm ${pth}/func/temp


# Create brain mask for functional data:

fslmaths ${pth}/func/${scan} -Tmean ${pth}/func/${scan}_mean
bet ${pth}/func/${scan}_mean ${pth}/func/${scan}_brain -f 0.3 -n -m -R 

epi_reg --epi=${pth}/func/${scan}_mean.nii.gz --t1=${pth}/anat/${struct}.nii.gz \
--t1brain=${pth}/anat/brain.nii.gz --out=${pth}/reg/T2toT1_${scan}

# Obtain a struct2func transform by inverting the func2struct transform:
convert_xfm -omat ${pth}/reg/T1toT2_${scan}.mat -inverse ${pth}/reg/T2toT1_${scan}.mat
```

---

<a name="8"></a>


##  **8.-** Functional to structural co-registration using Freesurfer (optional)

Replace FSL's epi_reg by Freesurfer's mri_coreg and bbregister in the previous snippet by:

```shell
# Functional to structural co-registration using Freesurfer's mri_coreg  and bbregister:

mri_coreg --s S${subjnum} --mov ${pth}/func/${scan}_mean.nii.gz --reg ${pth}/reg/tkreg_run${cond}.lta --debug

bbregister --s S${subjnum} --mov ${pth}/func/${scan}_mean.nii.gz --bold --init-fsl \
--fslmat ${pth}/reg/fsT2toT1_${scan}.mat --reg ${pth}/reg/fsT2toT1_${scan}.dat --wm-proj-abs 2 

# Obtain a struct2func transform by inverting the func2struct transform:
convert_xfm -omat ${pth}/reg/fsT1toT2_${scan}.mat -inverse ${pth}/reg/fsT2toT1_${scan}.mat
```

---

<a name="9"></a>


## **9.-** Remove motion artifacts using ICA-AROMA

First we need to install ICA-AROMA:

```shell
export arom_pth=/Users/username/Documents/Python/ICA-AROMA-master_new
cd ${arom_pth} 
curl https://bootstrap.pypa.io/get-pip.py | python
python2.7 -m pip install -r requirements.txt
```

Now, to use ICA-AROMA we need: 

* The T2toT1 affine matrix.
* The T1toMNI warping coefficients.
* The T2 motion coefficients.
* The smoothed T2.
* The T2 brain mask. 
* The repetition time (TR).

Read ICA-AROMA tutorial and papers!
 
Then try:

```shell
mkdir ${pth}/mc
export arom_pth=/Users/username/Documents/Python/ICA-AROMA-master_new

% Repetition time in seconds:
export TR=2

# Run ICA-AROMA
python ${arom_pth}/ICA_AROMA.py -in ${pth}/func/${scan}_smoothed.nii.gz -out ${pth}/mc/ica_aroma_${scan} \
-mc ${pth}/func/${scan}_mcf.par -affmat ${pth}/reg/T2toT1_${scan}.mat -warp ${pth}/reg/T1toMNI_warp.nii.gz \
-m ${pth}/func/${scan}_brain_mask.nii.gz -tr ${TR} -den no

# Regress motion components from data:
fsl_regfilt -i ${pth}/func/${scan}.nii.gz -d ${pth}/mc/ica_aroma_${scan}/melodic.ica/melodic_mix -o ${pth}/mc/${scan}_denoised.nii.gz -f ${pth}/mc/ica_aroma_${scan}/classified_motion_ICs.txt

# Check results:
#fsleyes ${pth}/func/${scan}.nii.gz ${pth}/mc/${scan}_denoised.nii.gz

# flirt -in ${pth}/mc/${scan}_denoised.nii.gz -ref ${pth}/anat/${struct}.nii.gz -out ${pth}/mc/${scan}_denoised_reg.nii.gz -init ${pth}/reg/T2toT1_${scan}.mat  -applyxfm4D 

```

---

<a name="10"></a>


## **10.-** Obtain BOLD time series from ROIs in FSL atlases 

The following is the Harvard-Oxford subcortical atlas overlaid on top of the structural data of a single subject: 

![subc]({{site.baseurl}}/assets/subc.png)


```shell
mkdir ${pth}/results
mkdir ${pth}/results/nat

invwarp --ref=${pth}/anat/${struct}.nii.gz \
--warp=${pth}/reg/T1toMNI_warp --out=${pth}/reg/MNItoT1_warp

##########################################################################################
# Compute BOLD % signal change for all voxels
##########################################################################################
fslmaths ${pth}/mc/${scan}_denoised.nii.gz -Tmean ${pth}/mc/${scan}_baseline
fslmaths ${pth}/mc/${scan}_denoised.nii.gz -div ${pth}/mc/${scan}_baseline -mul 100 -sub 100 ${pth}/mc/${scan}_BOLD.nii.gz 

##########################################################################################
# Create mask for left and right hemispheres
##########################################################################################
export atlas_subc=/usr/local/fsl/data/atlases/HarvardOxford/HarvardOxford-sub-prob-2mm.nii.gz

fslroi ${pth}/func/${scan} ${pth}/func/${scan}_1stvol 0 1


applywarp -i ${atlas_subc} -r ${pth}/func/${scan}_mean --postmat=${pth}/reg/T1toT2_${scan}.mat \
 -w ${pth}/reg/MNItoT1_warp -o ${pth}/atlas_subc_nat --interp=nn

############################################################
# Extract masks for left and right cortical hemispheres:
fslroi ${pth}/atlas_subc_nat leftCortex 01 1 # Left gray matter
fslmaths leftCortex -thr 20 leftCortex
fslmaths leftCortex -bin leftCortex
fslroi ${pth}/atlas_subc_nat rightCortex 12 1 # Right gray matter
fslmaths rightCortex -thr 20 rightCortex
fslmaths rightCortex -bin rightCortex
```

```shell
############################################################
# Oxford-Harvard cortical and subcortical atlases:
############################################################
export atlas_cort=/usr/local/fsl/data/atlases/HarvardOxford/HarvardOxford-cort-prob-2mm.nii.gz
applywarp -i ${atlas_cort} -r ${pth}/func/${scan}_mean --postmat=${pth}/reg/T1toT2_${scan}.mat -w ${pth}/reg/MNItoT1_warp -o ${pth}/atlas_cort_nat --interp=nn

############################################################
# Left hemisphere
# Loop through ROIs:
for roinum in 0{0..9} {10..47}; do
# Extract ROI from atlas:
fslroi ${pth}/atlas_cort_nat roimask_$roinum $roinum 1
# Threshold probabilistic ROI mask:
fslmaths roimask_$roinum -thr 50 -bin -mas leftCortex roimask_$roinum 
# Extract ROI mean time series:
fslmeants -i ${pth}/mc/${scan}_BOLD.nii.gz -o ts_$roinum.txt -m roimask_$roinum 
rm ${pth}/roimask_$roinum.nii.gz
done 
# Combine time series
paste ${pth}/ts_*.txt > ${pth}/results/nat/tseries_cort_lh.txt
rm ${pth}/ts_*


############################################################
# Right hemisphere
# Loop through ROIs:
for roinum in 0{0..9} {10..47}; do
# Extract ROI from atlas:
fslroi ${pth}/atlas_cort_nat roimask_$roinum $roinum 1
# Threshold probabilistic ROI mask:
fslmaths roimask_$roinum -thr 50 -bin -mas rightCortex roimask_$roinum 
# Extract ROI mean time series:
fslmeants -i ${pth}/mc/${scan}_BOLD.nii.gz -o ts_$roinum.txt -m roimask_$roinum 
rm ${pth}/roimask_$roinum.nii.gz
done 
# Combine time series
paste ${pth}/ts_*.txt > ${pth}/results/nat/tseries_cort_rh.txt
rm ${pth}/ts_*
```

```shell
############################################################
# Subcortical structures
############################################################
# Loop through ROIs:
for roinum in 0{0..9} {10..20}; do
# Extract ROI from atlas:
fslroi ${pth}/atlas_subc_nat roimask_$roinum $roinum 1
# Threshold probabilistic ROI mask:
fslmaths roimask_$roinum -thr 50 -bin roimask_$roinum 
# Extract ROI mean time series:
fslmeants -i ${pth}/mc/${scan}_BOLD.nii.gz -o ts_$roinum.txt -m roimask_$roinum 
rm ${pth}/roimask_$roinum.nii.gz
done
# Combine time series
paste ${pth}/ts_*.txt > ${pth}/results/nat/tseries_subc.txt
rm ${pth}/ts_*
```

```shell
############################################################
# Thalamic  subfields
############################################################
export atlas_thal=/usr/local/fsl/data/atlases/Thalamus/Thalamus-prob-2mm.nii.gz
applywarp -i ${atlas_thal} -r ${pth}/func/${scan}_mean --postmat=${pth}/reg/T1toT2_${scan}.mat \
 -w ${pth}/reg/MNItoT1_warp -o ${pth}/atlas_thal_nat --interp=nn
############################################################
# Loop through ROIs:
for roinum in 0{0..6}; do
# Extract ROI from atlas:
fslroi ${pth}/atlas_thal_nat roimask_$roinum $roinum 1
# Threshold probabilistic ROI mask:
fslmaths roimask_$roinum -thr 50 -bin roimask_$roinum 
# Extract ROI mean time series:
fslmeants -i ${pth}/mc/${scan}_BOLD.nii.gz -o ts_$roinum.txt -m roimask_$roinum 
rm ${pth}/roimask_$roinum.nii.gz
done
# Combine time series
paste ${pth}/ts_*.txt > ${pth}/results/nat/tseries_thal.txt
rm ${pth}/ts_*
```

---


<a name="11"></a>

## **11.-** Create .MGZ files with surface-node X ROI parameter and time series

```shell
export subjnum=15

mkdir ${data_folder}/S$subjnum/fsurf
export pth=${data_folder}/S$subjnum
cd ${pth}/fsurf
export cond=01
export input=${pth}/fsurf/func_run$cond

cp ${pth}/mc/run${cond}_denoised.nii.gz ${input}.nii.gz

fslswapdim ${input}.nii.gz x -z y ${input}.nii.gz
fslmaths ${input}.nii.gz -Tmean ${input}_mean.nii.gz

##########################################################################################
# Iterative alignment with bbregister and tkregister
##########################################################################################
# Start from header (if lucky) or from FLIRT or mri_coreg alignment:

mri_coreg --s S${subjnum} --mov ${input}_mean.nii.gz --reg ${pth}/fsurf/tkreg_run${cond}.lta --debug

# Check results:
# tkregisterfv --mov ${input}_mean.nii.gz --targ $FREESURFER_HOME/subjects/S${subjnum}/mri/brainmask.mgz --reg ${pth}/fsurf/tkreg_run${cond}.lta --s S${subjnum}  --surfs

# Boundary based registration (based on segmentation):

bbregister --s S${subjnum} --mov ${input}_mean.nii.gz --bold --init-reg ${pth}/fsurf/tkreg_run${cond}.lta \
 --reg ${pth}/fsurf/tkreg_run${cond}.dat --fslmat ${pth}/fsurf/fsT2toT1_run${cond}.mat --wm-proj-abs 2 

# Check:
# tkregisterfv --mov ${input}_mean.nii.gz --reg ${pth}/fsurf/tkreg_run${cond}.dat --surfs 

# Awesome! and can be improved my hand and also iterating.

# Obtain registered T2 by writing alignment to header
mri_vol2vol --mov ${input}.nii.gz --reg ${pth}/fsurf/tkreg_run${cond}.dat --o ${input}_reg.nii.gz --trilin --fstarg --no-resample

# Check results:
# tkregisterfv --mov ${input}_mean.nii.gz --targ $FREESURFER_HOME/subjects/S${subjnum}/mri/orig.mgz \

## Interpolate functional to anatomical surface (similar to_freesurfer.py)
export method=trilinear
export hemis=lh
mri_vol2surf --mov ${input}_reg.nii.gz --srcreg ${pth}/fsurf/tkreg_run${cond}.dat --hemi ${hemis} --cortex \
--out ${input}_reg_func2surf_${hemis}.mgz --interp ${method} --projfrac 0.5
export hemis=rh
mri_vol2surf --mov ${input}_reg.nii.gz --srcreg ${pth}/fsurf/tkreg_run${cond}.dat --hemi ${hemis} --cortex \
--out ${input}_reg_func2surf_${hemis}.mgz --interp ${method} --projfrac 0.5
```

---

<a name="12"></a>

## **12.-** Project data to the cortical surface reconstruction  

```shell
# # Interrogate interpolated time series in the cortical surface
# freeview \
# -f $FREESURFER_HOME/subjects/S${subjnum}/surf/lh.inflated:overlay=${input}_reg_func2surf_lh.mgz \
# -f $FREESURFER_HOME/subjects/S${subjnum}/surf/rh.inflated:overlay=${input}_reg_func2surf_rh.mgz

# freeview -f "$SUBJECTS_DIR"/S${subjnum}/surf/${hemis}.inflated
export subjnum=15
export hemis=lh
freeview -f $FREESURFER_HOME/subjects/S${subjnum}/surf/${hemis}.inflated
```

![Picture]({{site.baseurl}}/assets/benson.png){:height="320px" width="320px"}![Picture]({{site.baseurl}}/assets/wang.png){:height="320px" width="320px"}


<a name="13"></a>

## **13.-** Import Freesurfer data in Pycortex

```shell
python3 -m venv piCortex
source piCortex/bin/activate
pip install --upgrade pip
pip install -U setuptools wheel numpy cython
pip install -U pycortex
pip install ipython
source /Users/nicogravel/piCortex/bin/activate
export FREESURFER_HOME=/Applications/freesurfer
source $FREESURFER_HOME/SetUpFreeSurfer.sh
jupyter notebook
```

```python
import cortex
import cortex.polyutils
from cortex.options import config

print(cortex.options.usercfg)
print(config.get('basic', 'filestore'))


subject = "S15"

cortex.freesurfer.import_subj(subject, whitematter_surf="white")
```

If the import fails. Delete the subject folder from the pyCortex database before trzing again:
```shell
cd /Users/nicogravel/piCortex/share/pycortex/db
ls
rm -r S15
```

Restart kernel before importing the next subject

```python
import os
from IPython.core.display import HTML

HTML("<script>Jupyter.notebook.kernel.restart()</script>")
```



---

<a name="14"></a>

## **14.-** Compute cortical (geodesic) distances in millimeters using Pycortex

```python
import cortex
import cortex.polyutils
import numpy as np, h5py 
import matplotlib.pyplot as plt
import scipy.io as sio
import nibabel.freesurfer.mghformat as mgh
from cortex.options import config
    

print(cortex.options.usercfg)
print(config.get('basic', 'filestore'))

folder = ('/Volumes/Data/yourData/')
subject = ('S12', 'S13', 'S14', 'S15', 'S16', 'S17', 'S18', 'S19', 'S20', 'S21')
rois = ('V1', 'V2','V3') # rois: V1, V2, V3, hV4, VO1, VO2, LO1, LO2, TO1, TO2, V3b, V3a


    
for s in range(10):
        
    
    # First we need to import the surfaces for this subject
    surfs = []
    surfs = [cortex.polyutils.Surface(*d) for d in cortex.db.get_surf(subject[s], "fiducial")]

    for roi in range(2):

        # Left hemisphere
        atlas = mgh.load(str(folder) + str(subject[s]) + '/fsurf/lh.benson14_varea.mgz')
        areas =  np.squeeze(atlas.get_fdata())
        print('shape of data array:', areas.shape)
        roi_lh = [] 
        roi_lh = [i for i in range(len(areas)) if areas[i] == roi + 1] 
        roi_lh = np.asarray(roi_lh, dtype=int)
        print(roi_lh.shape)
        cdist_lh = np.zeros([len(roi_lh),len(roi_lh)])
        for p in range(0,len(roi_lh)-1):
            vert = roi_lh[p] 
            dists = [surfs[0].geodesic_distance(roi_lh[p])]
            lh = dists[0]
            for q in range(0,len(roi_lh)-1):
                cdist_lh[p,q] = lh[roi_lh[q]]
        print('shape of data array:', cdist_lh.shape)
        # Save cortical distance matrix 
        sio.savemat(str(folder) + str(subject[s]) + '/fsurf/cdis_' + str(rois[roi]) + '_lh.mat', {'arr': cdist_lh})

        # Right hemisphere
        areas = [] 
        atlas = mgh.load(str(folder) + str(subject[s]) + '/fsurf/rh.benson14_varea.mgz')
        areas = np.squeeze(atlas.get_fdata())
        print('shape of data array:', areas.shape)
        roi_rh = [] 
        roi_rh = [i for i in range(len(areas)) if areas[i] == roi + 1] 
        roi_rh = np.asarray(roi_rh, dtype=int)
        print(roi_rh.shape)
        cdist_rh = np.zeros([len(roi_rh),len(roi_rh)])
        for p in range(0,len(roi_rh)-1):
            vert = roi_rh[p] 
            dists = [surfs[1].geodesic_distance(roi_rh[p])]
            rh = dists[0]
            for q in range(0,len(roi_rh)-1):
                cdist_rh[p,q] = rh[roi_rh[q]]
        print('shape of data array:', cdist_rh.shape)
        # Save cortical sdistance matrix 
        sio.savemat(str(folder) + str(subject[s]) + '/fsurf/cdis_' + str(rois[roi]) + '_rh.mat', {'arr': cdist_rh})
```

Plot cortical distances

```python
plt.rcParams['figure.figsize'] = [20, 20]
plt.figure()
plt.imshow(cdist_rh)
plt.colorbar()
plt.xlabel('vertex', fontsize=14)
plt.ylabel('vertex', fontsize=14)
plt.title('Cortical distance (mm)')
```


---

<a name="15"></a>

## **15.-** Load data in Python  

```python
import os
import numpy as np
import scipy.signal as spsg
import scipy.stats as stt
import matplotlib  
from matplotlib import pyplot as plt
from scipy.signal import butter, filtfilt


# Time resolution for fMRI signals (in seconds)
TR = 2.0
T = 210    # number of TRs of the recording


# Filter the BOLD signals between 0.01 and 0.2 Hz
n_order = 4
Nyquist_freq = 0.5 / TR
low_f = 0.01 / Nyquist_freq
high_f = 0.1 / Nyquist_freq
b,a = spsg.iirfilter(n_order, [low_f,high_f], btype='bandpass', ftype='butter')

data = np.loadtxt('/Users/nicogravel/Documents/fMRI/Tutorials/fMRI-barebones_mwe/S15/results/nat/tseries_cort_lh.txt')
#data = np.loadtxt('/Users/nicogravel/Documents/fMRI/Tutorials/fMRI-barebones_mwe/S15/results/nat/tseries_cort_rh.txt')
#data = np.loadtxt('/Users/nicogravel/Documents/fMRI/Tutorials/fMRI-barebones_mwe/S15/results/nat/tseries_subc.txt')
#data = np.loadtxt('/Users/nicogravel/Documents/fMRI/Tutorials/fMRI-barebones_mwe/S15/results/nat/tseries_thal.txt')

# Filtered time series
plt.figure()
plt.rcParams['figure.figsize'] = [20, 5]
ts = spsg.filtfilt(b,a,data, axis=-1)
plt.plot(range(T),ts)
#plt.plot(range(T),data)
plt.xlabel('time')

# Correlations
actmat = np.corrcoef(data.T)
print('shape of data array:', actmat.shape)
plt.figure()
plt.imshow(actmat)
plt.colorbar()
```

```python
######################
# fMRI data properties    
n_sub = 7  # number of subjects
n_run = 2  # conditions / treatments
#N = 103     # number of ROIs + thalamus
N = 96     # number of ROIs
T = 210    # number of TRs of the recording
scrt = np.zeros([n_sub,n_run,N,T])

# fMRI data files 
rootdir = '/Users/nicogravel/Documents/fMRI/Meditech/jpNotebook/data'
idx = -1
for subdir, dirs, files in sorted(os.walk(rootdir)):
    for file in files:
        #print os.path.join(subdir, file)
        filepath = subdir + os.sep + file
        if filepath.endswith("lh.txt"):
            #print (subdir)
            lh = np.loadtxt(os.path.join(subdir, 'tseries_cort_lh.txt'))
            rh = np.loadtxt(os.path.join(subdir, 'tseries_cort_rh.txt'))
            th = np.loadtxt(os.path.join(subdir, 'tseries_thal.txt'))
            #ts = np.hstack((lh,rh,th))
            ts = np.hstack((lh,rh))
            subj = os.path.split(os.path.dirname(filepath))[1]
            #print(int(subj)-2)
            idx += 1
            #print(idx)
            #scrt[(int(subj)-2),0,:,:] = ts.T  # SubjectConditionRoiTime 
            scrt[idx,0,:,:] = ts.T  # SubjectConditionRoiTime 
            #print('shape of data array:', scrt.shape)
            fig, axs = plt.subplots(2)
            fig.suptitle('S' + subj)
            axs[0].plot(lh)
            axs[1].plot(rh)
            plt.show(block=False)
```


---




