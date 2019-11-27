---
layout: post
title: "Searchlight Analyses with Nilearn "
date: 2019-11-22
description: A few things I've found helpful when doing searchlight analyses
---

Searchlight analyses are a common fMRI technique that deploys pattern classification or representational similarity analysis across the entire brain. Areas thought to be involved in the task can be readily mapped across brain using multivariate information rather than gross univariate differences between conditions (as with conventional GLM approaches.) 

To learn more about searchlight analyses, I recommend checking out [Kriegeskorete et al, (2006)](https://www.pnas.org/content/103/10/3863.short), which is the flagship paper on the technique. I also recommend [Chen et al (2011)](https://www.sciencedirect.com/science/article/abs/pii/S1053811910010086), [Pereira & Botvinick (2011)](https://www.sciencedirect.com/science/article/abs/pii/S1053811910007585?via%3Dihub), and [Etzel et al (2013)](https://www.sciencedirect.com/science/article/abs/pii/S1053811913002917) for papers that discuss implementation, assumptions, and challenges searchlight approaches. 

In this post, I want to describe how I use implement searchlight analyses with [nilearn's `Searchlight`](https://nilearn.github.io/modules/generated/nilearn.decoding.SearchLight.html#nilearn.decoding.SearchLight) class. Nilearn has an excellent [user guide](https://nilearn.github.io/decoding/searchlight.html), which is required reading if you're hoping use nilearn for searchlight analyses. I want to expand on the user guide and discuss some more realistic aspects of a searchlight analysis, and how it is implemented in a proper analysis pipeline with other packages such as scikit-learn and scipy.       

### A minimal example

Before we begin, I'll admit that this post isn't *fully* reproducible; it makes the assumption that you have readily available data to use for a searchlight. This could be a series of single-trial beta images or volumes of interest (i.e. the volume in a trial that you wish to use for searchlight). I also make the assumption that your data is in standard 2mm MNI152 space.

You can set up your data like so:
```python
imgs = # 4D NIfTI image or list of 3D NIfTI images
img_labels = # array-like of condition labels for each volume in `imgs` 
```
`img_labels` should correspond to your response variable to be used for classification, and its length should equal the number of volumes in `imgs`. Next, running a bare-bones searchlight is pretty straightforward with nilearn: 

```python
from nilearn.decoding import Searchlight
from nilearn.datasets import load_mni152_brain_mask 

brain_mask = load_mni152_brain_mask()
searchlight = Searchlight(mask_img=brain_mask, radius=4)
results = searchlight.fit_transform(imgs, img_labels)
```
`results` will be a NIfTI image of containing the average cross-validation accuaccuracy for each voxel. 

The [user guide](https://nilearn.github.io/decoding/searchlight.html) has a similar example using the Haxby dataset, with some minor adjustments to the default parameters. The minimal example here and the user guide example serve as gentle ways to introduce `Searchlight` and not worry about details typically considered in actual searchlight analyses. In the next two sections I'll move beyond these toy examples and discuss modifications that properly reflect how searchlight analyses are performed "in the wild".

### Defining a cross-validation scheme

We first need to consider how we want to cross-validate our classifier, because `Searchlight` uses 3-fold cross-validation by default (a carryover from scikit-learn). In contrast, a typical searchlight analysis uses either leave-one-out cross-validation (LOOCV) or leave-one-run-out cross-validation (LOROCV). 

Of the two methods, LOROCV is preferred because the held out validation set is from an entirely independent scanning run. Meanwhile, training data in LOOCV contains data from the same scanning run as the validation sample, making LOOCV more prone to overfitting because the validation data is not truly independent. For more on choosing a cross-validation scheme, refer to [Misaki et al (2010)](https://www.sciencedirect.com/science/article/pii/S1053811910007834?via%3Dihub) and [Varoquaux et al (2017)](https://www.sciencedirect.com/science/article/pii/S105381191630595X?via%3Dihub).   

Implementing either LOOCV or LOROCV with nilearn is fairly easy. `Searchlight` has a `cv` parameter that takes an instance of a scikit-learn [splitter class](https://scikit-learn.org/stable/modules/classes.html#splitter-classes). Scikit-learn provides a splitter class for both LOOCV and LOROCV, among others.

For LOOCV:
```python
from sklearn.model_selection import LeaveOneOut

searchlight = Searchlight(mask_img=brain_mask, radius=4, 
                          cv=LeaveOneOut())
results = searchlight.fit_transform(imgs, img_labels)
```

LOROCV requires additional labels to group each image according to its scanning run. Then, these labels are fed into `fit_transform` to inform the cross-validation. 

```python
from sklearn.model_selection import LeaveOneGroupOut

run_labels = # array-like of run labels for each volume of `imgs`

searchlight = Searchlight(mask_img=brain_mask, radius=4, 
                          cv=LeaveOneGroupOut())
results = searchlight.fit_transform(imgs, img_labels,
                                    groups=run_labels)
```

### Building a proper estimator

Next, we need to decide on the estimator we use in our analysis. `Searchlight` uses support vector machines (SVM) by default for its estimator, which is a solid default option because SVMs generally perform well with fMRI data. It's important to note that this default is an instance of [scikit-learn's `LinearSVC`](https://scikit-learn.org/stable/modules/generated/sklearn.svm.LinearSVC.html#sklearn.svm.LinearSVC) estimator with `C=1` (as per the [user guide](https://nilearn.github.io/decoding/searchlight.html#classifier) and the [source code](https://github.com/nilearn/nilearn/blob/master/nilearn/decoding/searchlight.py#L31)). `LinearSVC` uses `LIBLINEAR` under the hood instead of the more well-known `LibSVM`, which is used by [`SVC`](https://scikit-learn.org/stable/modules/generated/sklearn.svm.SVC.html#sklearn.svm.SVC). Although you can expect near identical results between both implementations, you'll want to report the implementation in your paper's methods section. 

But in most cases, we'll want to modify this default anyways. Typical decoding analyses, like any machine learning task, involves some sort of rescaling of the features. Unfortunately, we simply can't take all of our images and apply whatever rescaling to each voxel. Doing so would eliminate the independence of our training and validation sets during cross-validation because now parameters of the 'unseen' validation set are influencing the training set. Rather, a given iteration of cross-validation should rescale the training data using the parameters of only the training data, and apply the obtained parameters from this step to the validation set. If this is unclear, check out [this Stack Exchange post](https://stats.stackexchange.com/questions/77350/perform-feature-normalization-before-or-within-model-validation).

Thankfully, scikit-learn makes this approach to rescaling really easy with its [pipeline module](https://scikit-learn.org/stable/modules/compose.html). Pipelines chain together scikit-learn transformers and estimators so that they can be cross-validated together as a single estimator. This can be done either with the `Pipeline` class or the `make_pipeline` utility function. If we want to properly standardize our voxels within our searchlight sphere, we could set up the following pipeline:

```python
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from skearn.svm import LinearSVC

pipeline = make_pipeline(StandardScaler(), LinearSVC())

searchlight = Searchlight(mask_img=brain_mask, radius=4, 
                          cv=LeaveOneGroupOut(), 
                          estimator=pipeline)
results = searchlight.fit_transform(imgs, img_labels,
                                    groups=run_labels)
```
In the above example, LOROCV is performed using a linear SVM, and each cross-validation fold properly *z*-transforms each voxel within the searchlight sphere. It's easy to modify the pipleline to use a different classifier (e.g., logistic regression), or a different rescaling approach (e.g., mean centering), or an additional step. I think the above code example really speaks to how well nilearn works with scikit-learn. 

Another approach to rescaling features is to rescale *across* voxels (i.e. rescale each voxel pattern) instead of rescaling *within* voxels. If you come from a more traditional machine learning background, this sounds crazy because your features typically represent very different things. However, fMRI data is a special case where the features are all of the same modality (voxels) and rescaling each voxel pattern might be more appropriate than rescaling within each voxel (e.g., wanting to remove amplitude effects in your patterns). For more, see [Misaki et al (2010)](https://www.sciencedirect.com/science/article/pii/S1053811910007834?via%3Dihub). 

Implementing this approach requires a little bit of effort. We need to somehow get `Searchlight` to only rescale the voxels within the searchlight sphere. We cannot rescale each volume of `imgs` because we would be including non-sphere voxels. So, pattern rescaling would have to be implemented in some sort of scikit learn pipepline that could be fed into `Searchlight`. This can be done by creating a custom transformation object using `FunctionTransformer`:

```python
from scipy.stats import zscore
from sklearn.preprocessing import FunctionTransformer

pattern_scaler = FunctionTransformer(zscore, kw_args={'axis': 1}, 
                                     validate=True)

pipeline = make_pipeline(pattern_scaler, LinearSVC())

searchlight = Searchlight(mask_img=brain_mask, radius=4, 
                          cv=LeaveOneGroupOut(), 
                          estimator=pipeline)
results = searchlight.fit_transform(imgs, img_labels,
                                    groups=run_labels)
```

Here, we've created a scikit-learn transformer for scipy's `zscore` function with `axis=1` so that we scale within each pattern. 

### Conclusion

I hope that this post can give some insight into how to move beyond simple toy examples and into actual real-world analyses with nilearn's `Searchlight`. As you can tell, building a realistic "publication-ready" searchlight analysis requires familiarity with scikit-learn and machine learning best practices. `Searchlight` simply runs whatever model you give it and its up to you to construct the best model for your data.

One thing I don't discuss is how to combine multiple single-subject searchlight results into a group-level analysis. Group-level searchlight analysis is a different can of worms altogether, and probably warrants its own post sometime in the future. But for now, I hope that this post is useful for those wanting to get a better sense of they can include nilearn into their projects. 