---
layout: page
title: software
permalink: /software/
---

## Software

I initially developed these packages to address gaps that I identified in our research group's analysis workflows, which have subsequently streamlined our research. However, these packages aim to be useful beyond our lab; I hope that they benefit anyone analyzing (f)MRI data. Please feel free to get in touch and/or raise an issue on Github if you are interested in contributing, spot a bug, or have questions. Feedback always welcome!

### Surfplot

[surfplot](https://github.com/danjgale/surfplot) is a Python package for plotting publication-ready brain surfaces figures (e.g., statistical maps, parcellations). Users can easily add plotting layers, customize figure styling, and integrate plots into existing matplotlib workflows. See [documentation](https://surfplot.readthedocs.io/en/latest/) for more detail.
 
### Reg-fusion

[Reg-fusion](https://github.com/danjgale/reg-fusion) is a Python package that accurately projects MRI data in standard volumetric space (e.g., MNI space) to the *fsaverage* cortical surface--a common task in neuroimaging. It is a pure-Python implementation of the highly accurate registration fusion approach described [Wu et al. (2018)](https://onlinelibrary.wiley.com/doi/full/10.1002/hbm.24213). 

### Nixtract

[nixtract](https://github.com/danjgale/nixtract) is a command-line tool to extract timeseries from different MRI file types (NIFTI, GIFTI, and CIFTI). nixtract is designed to be programming language-agnostic and compatible with any neuroimaging pipeline. Finally, nixtract can also run several automated quality control analyses to check the quality the timeseries data, which is a critical step in any fMRI project.

# Open-Source Involvement

In addition to my own projects, I've contributed to the following projects:

- [AtlasReader](https://github.com/miykael/atlasreader)
- [Nilearn](https://nilearn.github.io/index.html)
- [Brainspace](https://github.com/MICA-MNI/BrainSpace)