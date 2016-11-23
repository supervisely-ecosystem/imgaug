# About

This library helps you with augmenting images for your machine learning projects.
It converts a set of input images into a new, much larger set of slightly altered images.

![64 quokkas](examples_grid.jpg?raw=true "64 quokkas")

The library currently support most commonly used augmentation techniques and can apply them to both images and landmarks.
The image below shows each of these techniques.

![Available augmenters](examples.jpg?raw=true "Effects of all available augmenters")

# Requirements and installation

Required dependencies:
* numpy
* scipy
* scikit-image
* OpenCV

Currently only tested in python2.7.

# Examples

A standard machine learning situation.
Train on batches of images and augment each batch via crop, horizontal flip ("Fliplr") and gaussian blur:
```python
from imgaug import augmenters as iaa

seq = iaa.Sequential([
    iaa.Crop(px=(0, 16)), # crop images from each side by 0 to 16px (randomly chosen)
    iaa.Fliplr(0.5), # horizontally flip 50% of the images
    iaa.GaussianBlur(sigma=(0, 3.0)) # blur images with a sigma of 0 to 3.0
])

for batch_idx in range(1000):
    # 'images' should be either a 4D numpy array of shape (N, height, width, channels)
    # or a list of 3D numpy arrays, each having shape (height, width, channels).
    # Grayscale images must have shape (height, width, 1) each.
    # All images must have numpy's dtype uint8. Values are expected to be in
    # range 0-255.
    images = load_batch(batch_idx)
    images_aug = seq.augment_images(images)
    train_on_images(images_aug)
```

Apply heavy augmentations to images (used to create the image at the very top of this readme):
```python
from imgaug import augmenters as iaa

# random example images
images = np.random.randint(0, 255, (16, 128, 128, 3), dtype=np.uint8)

# Sometimes(0.5, ...) applies the given augmenter only in 50% of all cases,
# e.g. Sometimes(0.5, GaussianBlur(0.3)) would blur every second image.
sometimes = lambda aug: iaa.Sometimes(0.5, aug)

seq = iaa.Sequential([
          sometimes(iaa.Crop(percent=(0, 0.1))), # crop images by 0-10% of their height/width
          iaa.Fliplr(0.5), # horizontally flip 50% of all images
          iaa.Flipud(0.5), # vertically flip 50% of all images
          sometimes(iaa.GaussianBlur((0, 3.0))), # blur images with a sigma between 0 and 3.0
          sometimes(iaa.AdditiveGaussianNoise(loc=0, scale=(0.0, 0.2))), # add gaussian noise to images
          sometimes(iaa.Dropout((0.0, 0.1))), # randomly remove up to 10% of the pixels
          sometimes(iaa.Multiply((0.5, 1.5))), # change brightness of images (50-150% of original value)
          sometimes(iaa.Affine(
              scale={"x": (0.8, 1.2), "y": (0.8, 1.2)}, # scale images to 80-120% of their size, individually per axis
              translate_px={"x": (-16, 16), "y": (-16, 16)}, # translate by -16 to +16 pixels (per axis)
              rotate=(-45, 45), # rotate by -45 to +45 degrees
              shear=(-16, 16), # shear by -16 to +16 degrees
              order=ia.ALL, # use any of scikit-image's interpolation methods
              cval=(0, 1.0), # if mode is constant, use a cval between 0 and 1.0
              mode=ia.ALL # use any of scikit-image's warping modes (see 2nd image from the top for examples)
          )),
          sometimes(iaa.ElasticTransformation(alpha=(0.5, 3.5), sigma=0.25)) # apply elastic transformations with random strengths
      ],
      random_order=True # do all of the above in random order
)

images_aug = seq.augment_images(images)
```

Quickly show example results of your augmentation sequence:
```python
from imgaug import augmenters as iaa

images = np.random.randint(0, 255, (16, 128, 128, 3), dtype=np.uint8)
seq = iaa.Sequential([iaa.Fliplr(0.5), iaa.GaussianBlur((0, 3.0))])

# show an image with 8*8 augmented versions of image 0
seq.show_grid(images[0], cols=8, rows=8)

# Show an image with 8*8 augmented versions of image 0 and 8*8 augmented
# versions of image 1. The identical augmentations will be applied to
# image 0 and 1.
seq.show_grid([images[0], images[1]], cols=8, rows=8)
```

Augment grayscale images:
```python
from imgaug import augmenters as iaa
images = np.random.randint(0, 255, (16, 128, 128), dtype=np.uint8)
seq = iaa.Sequential([iaa.Fliplr(0.5), iaa.GaussianBlur((0, 3.0))])
# The library expects a list of images (3D inputs) or a single array (4D inputs).
# So we add an axis to our grayscale array to convert it to shape (16, 128, 128, 1).
images_aug = seq.augment_images(images[:, :, :, np.newaxis])
```

Augment two batches of images in *exactly the same way* (e.g. horizontally flip 1st, 2nd and 5th images in both batches, but do not alter 3rd and 4th images):
```python
from imgaug import augmenters as iaa

# Standard scenario: You have N RGB-images and additionally 21 heatmaps per image.
# You want to augment each image and its heatmaps identically.
images = np.random.randint(0, 255, (16, 128, 128, 3), dtype=np.uint8)
heatmaps = np.random.randint(0, 255, (16, 128, 128, 21), dtype=np.uint8)

seq = iaa.Sequential([iaa.Fliplr(0.5), iaa.GaussianBlur((0, 3.0))])

# Convert the stochastic sequence of augmenters to a deterministic one.
# The deterministic sequence will always apply the exactly same effects to the images.
seq_det = seq.to_deterministic() # call this for each batch again, NOT only once at the start
images_aug = seq_det.augment_images(images)
heatmaps_aug = seq_det.augment_images(heatmaps)
```

Augment images *and* landmarks on these images:
```python
import imgaug as ia
from imgaug import augmenters as iaa
from scipy import misc
images = np.random.randint(0, 255, (16, 128, 128, 3), dtype=np.uint8)

# Generate random keypoints.
# The augmenters expect a list of imgaug.KeypointsOnImage.
keypoints_on_images = []
for _ in range(images.shape[0]):
    keypoints = []
    for _ in range(images.shape[1]):
        x = random.randint(0, width)
        y = random.randint(0, height)
        keypoints.append(ia.Keypoint(x=x, y=y))
    keypoints_on_images.append(ia.KeypointsOnImage(keypoints, shape=images[i].shape))

seq = iaa.Sequential([iaa.Fliplr(0.5), iaa.Affine(scale=(0.8, 1.2))])
seq_det = seq.to_deterministic() # call this for each batch again, NOT only once at the start

# augment keypoints and images
images_aug = seq_det.augment_images(images)
keypoints_aug = seq_det.augment_keypoints(keypoints)

# Example code to show each image and print the new keypoints coordinates
for img_idx, (image, keypoints_on_image) in enumerate(zip(images_aug, keypoints_aug)):
    misc.imshow(image)
    for kp_idx, keypoint in enumerate(keypoints_on_image.keypoints):
        keypoint_old = keypoints_on_images[img_idx].keypoints[kp_idx]
        x_old, y_old = keypoint_old.x, keypoint_old.y
        x_new, y_new = keypoint.x, keypoint.y
        print("[Keypoint %d] before aug: x=%d y=%d | after aug: x=%d y=%d" % (i, x_old, y_old, x_new, y_new))
```

Apply single augmentations to images:
```python
from imgaug import augmenters as iaa
images = np.random.randint(0, 255, (16, 128, 128, 3), dtype=np.uint8)

flipper = iaa.Fliplr(1.0) # always horizontally flip each input image
images[0] = flipper.augment_images([images[0]])[0] # horizontally flip image 0

vflipper = iaa.Flipud(0.9) # vertically flip each input image with 90% probability
images[1] = vflipper.augment_images([images[1]])[0] # probably vertically flip image 1

blurer = iaa.GaussianBlur(3.0)
images[2] = blurer.augment_images([images[2]])[0] # blur image 2 by a sigma of 3.0
images[3] = blurer.augment_images([images[3]])[0] # blur image 3 by a sigma of 3.0 too

translater = iaa.Affine(translate_px={"x": -16}) # move each input image by 16px to the left
images[4] = translater.augment_images([images[4]])[0] # move image 4 to the left

scaler = iaa.Affine(scale={"y": (0.8, 1.2)}) # scale each input image to 80-120% on the y axis
images[5] = scaler.augment_images([images[5]])[0] # scale image 5 by 80-120% on the y axis
```

Use unusual distributions for the stochastic parameters of each augmenter:
```python
from imgaug import augmenters as iaa
from imgaug import parameters as iap
images = np.random.randint(0, 255, (16, 128, 128, 3), dtype=np.uint8)

# Blur by a value sigma which is sampled from a uniform distribution
# of range 0.1 <= x < 3.0.
# The convenience shortcut for this is: iaa.GaussianBlur((0.1, 3.0))
blurer = iaa.GaussianBlur(iap.Uniform(0.1, 3.0))
images_aug = blurer.augment_images(images)

# Blur by a value sigma which is sampled from a normal distribution N(1.0, 0.1),
# i.e. sample a value that is usually around 1.0.
# Clip the resulting value so that it never gets below 0.1 or above 3.0.
blurer = iaa.GaussianBlur(iap.Clip(iap.Normal(1.0, 0.1), 0.1, 3.0))
images_aug = blurer.augment_images(images)

# Same again, but this time the mean of the normal distribution is not constant,
# but comes itself from a uniform distribution between 0.5 and 1.5.
blurer = iaa.GaussianBlur(iap.Clip(iap.Normal(iap.Uniform(0.5, 1.5), 0.1), 0.1, 3.0))
images_aug = blurer.augment_images(images)

# Use for sigma one of exactly three allowed values: 0.5, 1.0 or 1.5.
blurer = iaa.GaussianBlur(iap.Choice([0.5, 1.0, 1.5]))
images_aug = blurer.augment_images(images)

# Sample sigma from a discrete uniform distribution of range 1 <= sigma <= 5,
# i.e. sigma will have any of the following values: 1, 2, 3, 4, 5.
blurer = iaa.GaussianBlur(iap.DiscreteUniform(1, 5))
images_aug = blurer.augment_images(images)
```