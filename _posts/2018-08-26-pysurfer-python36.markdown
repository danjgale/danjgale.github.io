---
layout: post
title: "Setting up Pysurfer in Python 3.6"
date: 2018-08-26
description: Problems getting Pysurfer installed? Check here!
---

{:, .notice}
**Update:** Pysurfer now supports Python 3.6+ according to the [documentation](http://pysurfer.github.io/install.html). Installation worked well out-of-the-box with `pysurfer>=0.9.0` when I tested it with both Linux and OSX. This is great news. I did have a small issue configuring `QT` to work with pysurfer and I discuss the solution [here](https://github.com/nipy/PySurfer/issues/262#issuecomment-467574981). If you are using earlier versions of Pysurfer, then keep reading.

[Pysurfer](http://pysurfer.github.io) is an excellent Python library for visualizing neuroimaging data using [Freesurfer](https://surfer.nmr.mgh.harvard.edu). With an elegant API and fantastic documentation, Pysurfer is a pleasure to use. Despite this, I _did_ encounter some initial difficulties getting Pysurfer installed in my Python 3.6 workflow.

<!--more-->

For starters, getting some of Pysurfer's dependencies (e.g., [Mayavi](http://docs.enthought.com/mayavi/mayavi/index.html), [VTK](https://www.vtk.org/)) caused headaches either during the installation process or after; this was no surprise given what's been discussed out there (for example, see [1](https://github.com/enthought/mayavi/issues/625), [2](https://stackoverflow.com/questions/45100010/conflict-while-installing-mayavi-into-anaconda), [3](http://bluesimplex.com/2017/02/04/installing_mayavi_with_python_3.html)). As well, I also expected some difficulty given that the [documentation](http://pysurfer.github.io/install.html), at the time of this writing, states that Pysurfer only works with Python 2. I saw that there had been some [discussion](https://github.com/nipy/PySurfer/issues/217) on getting Pysurfer to work in Python 3 using `conda`, but I was still having some issues, particularly with `VTK` and `PyQt`. Also notice that in the discussion they were able to get it working with Linux, but not MacOs (what I'm using); I suspect that was the source of my problem, or at least part of it.

Sitting down with [Ross Markello](https://twitter.com/rossdavism), we were able solve the dependency issues and get everything working exactly the way I had hoped in 3.6. The trick is to run Pysurfer in a fresh `conda` environment and set `PyQT` and `VTK` versions specific to the environment. We largely based our installation off of [Kirstie Whitaker's great instructions](https://github.com/KirstieJane/DESCRIBING_DATA/wiki/Making-Surface-Plots-with-Pysurfer) for 2.7 (if you _are_ interested in using Pysurfer with Python 2, I suggest you stick to those instructions).

Here, I've documented the installation steps to get Pysurfer working with Python 3 on MacOS (I'm guessing these steps also hold for Linux) in a new `conda` environment.


## Installation Steps
**Requirements:** Before we begin, you'll need to install [conda](https://conda.io/docs/) and Freesurfer on your computer (follow the installation steps for your operating system, as is, [here](https://surfer.nmr.mgh.harvard.edu/fswiki/DownloadAndInstall)) if they are not already installed.

First, let's set up our `conda` environment. We'll want to specify Python 3.6 and install some initial dependencies (note the specific `PyQT` and `VTK` versions):

```
$ conda create -n pysurfer python=3.6 pip mkl numpy scipy matplotlib "pyqt>=5.9" "vtk>8" pygments traits traitsui pyface
```

Now let's activate the `pysurfer` environment, install the remaining dependencies, and finally Pysurfer itself:

```
$ source activate pysurfer
$ pip install mayavi nibabel imageio
$ pip install pysurfer
```

Note that I've left out some other dependencies of interest (e.g., `ipython`) to keep things minimal for now. Once everything is installed, fire up your Python terminal and test out your installation (based on [this example](http://pysurfer.github.io/auto_examples/plot_basics.html#sphx-glr-auto-examples-plot-basics-py)):

```python
>>> from surfer import Brain
>>> brain = Brain(subject_id='fsaverage', hemi='lh', surf='inflated')
```

You should see a rendered brain open up in a new window. With that, you're set!