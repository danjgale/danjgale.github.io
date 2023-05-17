---
layout: post
title: "Within-mask spatial smoothing for MRI"
date: 2023-03-19
---

Spatial smoothing is an often-necessary step in the analysis of MRI data that improves the signal-to-noise ratio and spatial correspondence across heterogenous individuals. Recently, I found myself wanting to apply spatial smoothing for a set of mass-univariate analyses restricted to within the cortical mantle. I had my own set of constraints: it was best to implement this within Python (rather than use something like FSL or AFNI), and I really didn't want to blur in voxels lying outside the gray-matter mask that I was using to define cortical mantle. Unfortunately, [nilearn's `smooth_img` function](https://nilearn.github.io/dev/modules/generated/nilearn.image.smooth_img.html) (which I normally love) applys uniform smoothing across the entire image.

So I needed to roll out my own implementation. I came across [this post on Stack Overflow](https://stackoverflow.com/questions/59685140/python-perform-blur-only-within-a-mask-of-image), in which the answer gave me everything I needed. In brief, applying a normalized convolution approach, wherein a filter is convolved with a set of weights in order to downweight-out irrelevant data, properly restricts spatial smoothing to in-mask data only. This solution is remarkably simple and works well. 

Below is an implementation that borrows from the Stack Overflow answer in order to smooth voxels within a mask without contamination from outside voxels. `smooth_in_mask_img` is a function that performs the operation directly on the image, much like you'd expect from nilearn.  

```python
import numpy as np
from scipy import ndimage
import nibabel as nib
from nilearn._utils import check_niimg


def smooth_in_mask_img(img, mask, fwhm):
    """Smooth only data within mask using a normalized convolution approach

    Args:
        img (niimg-like): Data volume
        mask (niimg-like): Binary mask volume
        fwhm (float): Smoothing full-width half-maximum in mm

    Returns:
        nib.Nifti1Image: Smoothed volume
    """
    img = check_niimg(img)
    mask = check_niimg(mask)

    # scale FWHM to mm and convert to sigma
    affine = img.affine[:3, :3]
    vox_size = np.sqrt(np.sum(affine**2, axis=0))
    sigma = fwhm / (np.sqrt(8 * np.log(2)) * vox_size)
    
    img_vox = img.get_fdata()
    mask_vox = mask.get_fdata()

    # apply smoothing with weight convolution
    smoothed = ndimage.filters.gaussian_filter(img_vox * mask_vox, sigma=sigma)
    weights = ndimage.filters.gaussian_filter(mask_vox, sigma=sigma)
    smoothed /= weights

    # re-mask output
    smoothed = np.nan_to_num(smoothed * mask_vox)
    return nib.Nifti1Image(smoothed, img.affine)
```