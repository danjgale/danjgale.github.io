---
layout: post
title: "Searchlight Analyses with Nilearn "
date: 2019-11-22
description: A few things I've found helpful when doing searchlight analyses
---

Searchlight analyses are a common fMRI technique that deploys pattern classification or representational similarity analysis across the entire brain. Areas thought to be involved in the task can be readily mapped across brain using multivariate information rather than gross univariate differences between conditions (as with conventional GLM approaches.) 

To learn more about searchlight analyses, I recommend checking out [Kriegeskorete et al, (2006)](https://www.pnas.org/content/103/10/3863.short), which is the flagship paper on the technique. I also recommend [Chen et al (2011)](https://www.sciencedirect.com/science/article/abs/pii/S1053811910010086), [Pereira & Botvinick (2011)](https://www.sciencedirect.com/science/article/abs/pii/S1053811910007585?via%3Dihub), and [Etzel et al (2013)](https://www.sciencedirect.com/science/article/abs/pii/S1053811913002917) for papers that discuss implementation, assumptions, and challenges searchlight approaches. 

In this post, I want to describe how I use implement searchlight analyses with [nilearn's `Searchlight`](https://nilearn.github.io/modules/generated/nilearn.decoding.SearchLight.html#nilearn.decoding.SearchLight) class. Nilearn has an excellent [user guide](https://nilearn.github.io/decoding/searchlight.html), which is required reading if you're hoping use nilearn for searchlight analyses. I want to expand on the user guide and discuss some more realistic aspects of a searchlight analysis, and how they are implemented in a proper analysis pipeline with other packages such as scikit-learn and scipy.       


### A minimal example


### Building a proper estimator


### Group-level considerations


### Putting it all together