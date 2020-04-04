fMRI-barebones
=====================

 A simple yet efficient preprocessing pipeline for fMRI data using [FSL](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki) &amp; [Freesurfer](https://surfer.nmr.mgh.harvard.edu/). 



### Contents

- [Introduction](#Introduction)
- [Requirements](#Requirements)
- [Alignment](#Alignment)
- [Tissue segmentation](#Tissue-segmentation)
- [Skull stripping (BET)](#Skull-stripping-(BET))
- [Linear registratiion (FLIRT)](#Linear-registratiion-(FLIRT))
- [Non-linear registrationn (FNIRT)](#Non-linear-registrationn-(FNIRT))
- [Functional to structural co-registration (BBR)](#Functional-to-structural-co-registration-(BBR))
- [Automatic Removal of Motion Artifacts (ICA-AROMA)](#Automatic-Removal-of-Motion-Artifacts-(ICA-AROMA))
- [Extracting ROIs from atlases](#Extracting-ROIs-from-atlases)
- [Retrieving BOLD time series from ROIs](#Retrieving-BOLD-time-series-from-ROIs)
- [Projecting data to a cortical surface reconstruction](#Projecting-data-a-cortical-surface-reconstruction)
- [Using Freesurfer coregistration](#Using-freesurfer-coregistration)
- [Using Neuropythy to get visual areas atlase](#Using-Neuropythy-to-get-visual-areas-atlases)
- [Get .MGZ files with surface node X ROI parameter (or time series)](#Get-.MGZ-file-with-surface-node-X-ROI-parameter-(or-time-series))