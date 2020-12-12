---
layout: post
title: "Practical searchlight analyses with nilearn "
date: 2019-11-28
description: A few things I've found helpful when doing searchlight analyses
---

A searchlight analysis is a common fMRI technique that deploys pattern classification or representational similarity analysis across the entire brain. Areas thought to be involved in a task can be readily mapped across brain using local multivariate information rather than gross univariate differences between conditions (as with conventional GLM approaches). 

For some background, I recommend checking out [Kriegeskorete et al, (2006)](https://www.pnas.org/content/103/10/3863.short), which is the flagship paper on the technique. I also recommend [Chen et al (2011)](https://www.sciencedirect.com/science/article/abs/pii/S1053811910010086), [Pereira & Botvinick (2011)](https://www.sciencedirect.com/science/article/abs/pii/S1053811910007585?via%3Dihub), and [Etzel et al (2013)](https://www.sciencedirect.com/science/article/abs/pii/S1053811913002917) for papers that discuss the implementation, assumptions, and challenges of searchlight analyses. 

Here, I go over how I implement a searchlight analysis with [nilearn's `Searchlight`](https://nilearn.github.io/modules/generated/nilearn.decoding.SearchLight.html#nilearn.decoding.SearchLight) class. There is an excellent [user guide](https://nilearn.github.io/decoding/searchlight.html) that is definitely required reading. But, I want to move past basic examples and discuss how to construct a proper searchlight analysis. Essentially, how can we use nilearn to build a searchlight analysis as seen "in the wild"?

### A minimal example

Before we begin, I'll admit that this post isn't *fully* reproducible; it makes the assumption that you have readily available data to use for a searchlight. This could be a series of single-trial beta images or volumes of interest (i.e. the volume in a trial that you wish to use for searchlight). I also make the assumption that your data is in standard 2mm MNI152 space because I use a MNI brain mask.

You can set up your data like so:
```python
imgs = # 4D NIfTI image or list of 3D NIfTI images
y = # array-like of condition labels for each volume in `imgs` 
```
`y` corresponds to your response variable to be used for classification, and its length should equal the number of volumes in `imgs`. 

From here, it's straightforward to construct a bare-bones searchlight that is similar to the one in nilearn's user guide: 

```python
from nilearn.decoding import Searchlight
from nilearn.image import new_img_like
from nilearn.datasets import load_mni152_brain_mask 

# restrict searchlight to brain voxels only
brain_mask = load_mni152_brain_mask()

searchlight = Searchlight(mask_img=brain_mask, radius=4)
searchlight.fit(imgs, y)
results = new_img_like(brain_mask, searclight.scores_)
```
`results` will be a NIfTI image of containing the average cross-validation accuracy for each voxel. 

Obviously, this example is not at all representative of a searchlight analysis you would read about in a paper. The next two sections discuss a couple of aspects we need to consider when building a proper real-world searchlight analysis, which can be easily implemented thanks to scikit-learn. 


### Building a proper estimator

We first need to decide on the estimator we use in our analysis. `Searchlight` uses support vector machines (SVM) by default, which is a solid choice because SVMs generally perform well with fMRI data. It's important to note that this default is an instance of [scikit-learn's `LinearSVC`](https://scikit-learn.org/stable/modules/generated/sklearn.svm.LinearSVC.html#sklearn.svm.LinearSVC) estimator with `C=1` (as per the [user guide](https://nilearn.github.io/decoding/searchlight.html#classifier) and the [source code](https://github.com/nilearn/nilearn/blob/master/nilearn/decoding/searchlight.py#L31)). `LinearSVC` uses `LIBLINEAR` under the hood instead of the more well-known `LibSVM` that is used by [`SVC`](https://scikit-learn.org/stable/modules/generated/sklearn.svm.SVC.html#sklearn.svm.SVC). Although you can expect near identical results between both implementations, you'll want to report the implementation in your paper's methods section. 

`Searchlight` takes any scikit-learn estimator, so you are not limited to using SVMs. For instance, you can pass a logistic regression model with L1 regularization into `Searchlight` like so:

```python
from sklearn.linear_model import LogisticRegression

logreg = LogisticRegression('l1', solver='saga')

searchlight = Searchlight(mask_img=brain_mask, radius=4, estimator=logreg)
searchlight.fit(imgs, y)
results = new_img_like(brain_mask, searclight.scores_)
```

But in most cases, we'll want to move beyond just a classifier for our estimator. Typical decoding analyses, like any machine learning task, involve some sort of feature scaling. We can't rescale our input images directly because this would eliminate the independence of our training and validation sets during cross-validation; parameters of the 'unseen' validation set would influence the training set. Rather, a given iteration of cross-validation should rescale the the training and validation sets only using the parameters of the training data. If this is unclear, check out [this Stack Exchange post](https://stats.stackexchange.com/questions/77350/perform-feature-normalization-before-or-within-model-validation).

Scikit-learn has a number of [transformer classes](https://scikit-learn.org/stable/modules/classes.html#module-sklearn.preprocessing) that do this type of scaling. Much like a scikit-learn estimator, you can *fit* a transformer on the training data, and then *transform* the training and validation sets. These include classes such as `StandardScaler` and `MinMaxScaler`.

In order to combine the scaling step with the classifier, scikit-learn allows you to build a pipeline via its [pipeline module](https://scikit-learn.org/stable/modules/compose.html). Pipelines chain together various scikit-learn objects so that they can be cross-validated together as a single estimator. This can be done either with the `Pipeline` class or the `make_pipeline` utility function. If we want to properly standardize our voxels within our searchlight sphere, we could set up the following pipeline:

```python
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from skearn.svm import LinearSVC

pipeline = make_pipeline(StandardScaler(), LinearSVC())

searchlight = Searchlight(mask_img=brain_mask, radius=4, 
                          estimator=pipeline)
searchlight.fit(imgs, y)
results = new_img_like(brain_mask, searclight.scores_)
```
In the above example, each cross-validation fold properly *z*-transforms each voxel within the searchlight sphere prior to training and validating a linear SVM classifier. We can easily to modify the pipeline to use a different classifier (e.g., logistic regression) or a different rescaling approach (e.g., mean centering). I think the above code example really speaks to how well nilearn works with scikit-learn. 

Another approach to rescaling features is to rescale *across* voxels (i.e. rescale each voxel pattern) instead of rescaling *within* voxels. If you come from a more traditional machine learning background, this sounds crazy because your features typically represent different things. However, fMRI data is a special case where the features are all of the same modality (voxels) and rescaling each voxel pattern might be more appropriate than rescaling within each voxel (e.g., wanting to remove amplitude effects in your patterns). For more, see [Misaki et al (2010)](https://www.sciencedirect.com/science/article/pii/S1053811910007834?via%3Dihub). 

Implementing this approach requires a little bit of effort. We need to somehow get `Searchlight` to only rescale the voxels within the searchlight sphere. We cannot rescale within each volume of `imgs` because we would be including non-sphere voxels. Therefore, pattern rescaling would have to be implemented in some sort of scikit-learn pipeline that could be fed into `Searchlight`. Scikit-learn also doesn't have a way to change the direction of scaling because this is an atypical use-case. Thankfully, there is a way to create our own custom transformer that can be included in a pipeline. This is done via `FunctionTransformer`:

```python
from scipy.stats import zscore
from sklearn.preprocessing import FunctionTransformer

pattern_scaler = FunctionTransformer(zscore, kw_args={'axis': 1}, 
                                     validate=True)

pipeline = make_pipeline(pattern_scaler, LinearSVC())

searchlight = Searchlight(mask_img=brain_mask, radius=4, estimator=pipeline)
searchlight.fit(imgs, y)
results = new_img_like(brain_mask, searclight.scores_)
```

Here, we've created a scikit-learn transformer for scipy's `zscore` function with `axis=1` so that we scale within each pattern instead of within each voxel. I recommend trying each approach and comparing the searchlight maps for curiosity's sake.  

### Defining a cross-validation scheme

Next, we need to consider how we want to cross-validate our classifier, because `Searchlight` uses 3-fold cross-validation by default (a carryover from scikit-learn). In contrast, a typical searchlight analysis uses either leave-one-out cross-validation (LOOCV) or leave-one-run-out cross-validation (LOROCV). 

Of the two methods, LOROCV is preferred. First, the held out validation sets in LOROCV are from entirely independent scanning runs. Meanwhile, training data in LOOCV contains data from the same scanning run as the validation set, making LOOCV more prone to overfitting because the validation data are not truly independent. As well, LOROCV uses fewer cross-validation folds, which is more computationally efficient (this matters a lot with searchlight analyses). For more on choosing a cross-validation scheme, refer to [Misaki et al (2010)](https://www.sciencedirect.com/science/article/pii/S1053811910007834?via%3Dihub) and [Varoquaux et al (2017)](https://www.sciencedirect.com/science/article/pii/S105381191630595X?via%3Dihub).   

Implementing either LOOCV or LOROCV with nilearn is fairly easy. `Searchlight` has a `cv` parameter that takes an instance of a scikit-learn [splitter class](https://scikit-learn.org/stable/modules/classes.html#splitter-classes). Scikit-learn provides a splitter class for both LOOCV and LOROCV, among others.

For LOOCV:
```python
from sklearn.model_selection import LeaveOneOut

searchlight = Searchlight(mask_img=brain_mask, radius=4, estimator=pipeline, 
                          cv=LeaveOneOut())
searchlight.fit(imgs, y)
results = new_img_like(brain_mask, searclight.scores_)
```

LOROCV requires additional labels to group each image according to its scanning run. Then, these labels are fed into `fit_transform`: 

```python
from sklearn.model_selection import LeaveOneGroupOut

run_labels = # array-like of run labels for each volume of `imgs`

searchlight = Searchlight(mask_img=brain_mask, radius=4, estimator=pipeline, 
                          cv=LeaveOneGroupOut())
searchlight.fit(imgs, y, groups=run_labels)
results = new_img_like(brain_mask, searclight.scores_)
```

### Putting it all together

Now we can put things together to turn our minimal example into a fully-fledged analysis (also available as a [gist](https://gist.github.com/danjgale/3988f424c1a86051b2aa84b429ffd84d)). The final product looks like this: 

```python
from nilearn.decoding import Searchlight
from nilearn.datasets import load_mni152_brain_mask
from nilearn.image import new_img_like 
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.svm import LinearSVC
from sklearn.model_selection import LeaveOneGroupOut

imgs = # 4D NIfTI image or list of 3D NIfTI images
y = # array-like of condition labels for each volume in `imgs` 
run_labels = # array-like of run labels for each volume of `imgs`

pipeline = make_pipeline(StandardScaler(), LinearSVC())

brain_mask = load_mni152_brain_mask()
searchlight = Searchlight(mask_img=brain_mask, radius=4, estimator=pipeline, 
                          cv=LeaveOneGroupOut())
searchlight.fit(imgs, y, groups=run_labels)
results = new_img_like(brain_mask, searchlight.scores_)
```
Voila! There is a single-subject searchlight pipeline. You can imagine putting this into a function and running this for each subject.

### Conclusion

I hope that this post can give some insight into how to move beyond simple toy examples and into actual real-world analyses with nilearn's `Searchlight`. When I first started out with searchlight analyses, these details were not immediately obvious. 

Building a realistic "publication-ready" searchlight analysis requires familiarity with scikit-learn and machine learning best practices. `Searchlight` simply runs whatever you give it and its up to you to construct the approach for your data. Thankfully, the scikit-learn and nilearn documentation are excellent, and many of the papers linked in this post do a great job at discussing these necessary details.  

One thing I don't discuss is how to combine multiple single-subject searchlight results into a group-level analysis. Group-level searchlight analysis is a different can of worms altogether, and probably warrants its own post sometime in the future. But for now, I hope that this post is useful for those wanting to get a better sense of a proper implementation of a searchlight analysis in Python. 