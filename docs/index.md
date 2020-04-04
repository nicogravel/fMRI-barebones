---
layout: default
title: "fMRI barebones"
---

# fMRI-barebones

A simple yet useful preprocessing pipeline for fMRI data using [FSL](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki) &amp; [Freesurfer](https://surfer.nmr.mgh.harvard.edu/). 


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
- [Using Neuropythy to get anatomically defined ROIs for visual areas](#Using-Neuropythy-to-get-anatomically-defined-ROIs-for-visual-areas)
- [Get .MGZ files with surface node X ROI parameter (or time series)](#Get-.MGZ-file-with-surface-node-X-ROI-parameter-(or-time-series))

---
## Introduction

As a tutorial, fMRI barebones seeks to document and illustrate a set of important and useful steps commonly used in fMRI data preprocessing. As a toolbox, fMRI barebones provides a set of basic examples and scripts to help harness the power of [FSL](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki) &amp; [Freesurfer](https://surfer.nmr.mgh.harvard.edu/).

---
## Requirements

The tools are implemented from the command line using shell scripts commands. It has been devolped in OSX (should be easily adapted to Linux), FSL, Freesurfer, Python (ICA-AROMA and Neuropithy) and Matlab.
