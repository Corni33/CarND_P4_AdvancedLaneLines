# Adavanced Lane Line Detection

The goal of this project was to detect the lane line markings of the ego-lane using camera images.
To achieve this, the following steps were performed:

* Camera calibration (distortion correction) 
* Perspective transformation to achiev a "birds-eye view" of the road
* Lane pixel detection and polynomial fitting to find the lane boundary
* Determining the curvature of the lane and vehicle position with respect to the lane center
* Overlaying the detected lane markings onto the original camera image 

All of the code for completing the project is contained in [this jupyter notebook](https://github.com/Corni33/CarND_P4_AdvancedLaneLines/blob/master/advanced_lane_lines.ipynb).


[//]: # (Image References)
[images_orig]: ./images_orig.png "Recorded images (center, left and right camera)"
[images_cropped]: ./images_cropped.png "Images cropped to exclude unnecessary data"


## Camera Calibration

To compensate for the distortion effects (e.g. radial or tangential distortion) introduced by the camera lense a series of calibration images showing a calibration pattern (chessboard pattern) was has been provided.
After detecting the corners of the chessboard pattern in multiple images, I used OpenCV's "calibrateCamera(...)" function to calculate the distorition coefficients of the lense (notebook cell 3).
These coefficients then get used to define a function for undistorting camera images (notebook cell 4) like in the following example:

![alt-text-1](./readme_images/chessboard_dist.png "Distorted Image") ![alt-text-1](./readme_images/chessboard_undist.png "Undistorted Image") 

The left image has not been edited while the right image has been corrected for lense distortion.


## Perspective Transformation

The detection of lane lines is much easier when looking at the road from a birds-eye-view.
To achieve this view a perspective transformation gets applied to the camera image.
Cells 5 and 6 of the notebook provide the user with the opportunity to draw a transformed rectangle onto a camera image which corresponds to a standard rectangle in the birds-eye-view.
The coordinates of the rectangle corners then are used to calculate a perspective transformation matrix (cell 6)
Images can now be transformed ('warped') to and back from a birds-eye-view using the functions defined in cell 7 of the notebook. 

![alt-text-1](./readme_images/perspective_normal.png "Normal image") ![alt-text-1](./readme_images/perspective_top.png "Perspective transformed image") 


## Lane Line Detection

The warped camera image serves as a basis for detecting pixels which are likely part of a lane line before fitting a polynomial to get a functional description of each lane line. 
The following image serves as an example for visualizing the lane line detection pipeline (left: distortion corrected image, right: image warped into birds-eye-view):

![alt-text-1](./readme_images/input.png "distortion corrected imag") ![alt-text-1](./readme_images/input_warped.png "warped into birds-eye-view") 


### Edge Detection

After converting the image to grayscale, edge detection is performed separately in x- and y-direction by always convolving the image with a sobel kernel that simultaneously smooths the image.
Afterwards a sobel binary image is created via thresholding, that only contains ones (shown as white areas in the images) at the locations where there is a large gradient in x-direction and a small gradient in y-direction (cell 12):

![alt-text-1](./readme_images/gray.png "grayscaled image") ![alt-text-1](./readme_images/sobel.png "thresholded sobel image") 


### Color Channel Thresholding

The warped (birds-eye-view) color image gets converted to HLS color space where a threshold is applied to the S-channel (left image).
Another color threshold is applied to the R-channel of the warped color image in BGR color space (middle image). 
The two resulting binary images get combined via logical "AND" operation to yield a common binary image (right image):

![alt-text-1](./readme_images/s_binary.png "threshold on s-channel") ![alt-text-1](./readme_images/r_binary.png "threshold on r-channel") ![alt-text-1](./readme_images/s_r_binary.png "combined binary image") 




### Combination of Edge and Color Information

The binary image from color thresholding now gets combined with the binary image from edge detection via logical "OR".
The result of this operation is a single binary image containing ones where we expect either lane line edges or the lane line marking itself to be.
Unfortunately most images contain other features that get misclassified as being part of the lane lines (a.g. road border, parts of other vehicles, ...) when applying the above thresholding logic. 

#TODO example images

A first to improve on this situation is to remove small isolated spots by applying a morphological opening operation:

#TODO example images

### Line Area Masking

To get an estimate of which pixels belong to the left and right lane line, the binary image now gets masked (= applying logical "AND") with a distinct lane line mask for each each lane line.
A lane line mask contains an area estimate for where the lane line might be now given that we know where it was in the last frame that has been processed.

#TODO example images

### Polynomial Fitting

All the white pixels (ones) of each lane line binary image now get used as data points for fitting a parabola (cell ...). 
To make this fit more robust against outliers I used the RANSAC algorithm instead of standard least squares polynomial fitting.
The results of this step are two sets of polynomial coefficients describing each lane line in a functional form.

#TODO example images

From the polynomial coefficients the curvature of the lane at the vehicles current position as well as the vehicles lateral offset to the lane center can now be caluclated (cell ...).

The polynomial additionally gets used to generate the lane mask for the next frame:

#TODO example images

### Lane Overlay 

After the lane lines have been identified in the warped image the lane in the original image gets highlighting by 'unwarping' the polynomials and using them as boundary points for a lane polygon (cell ...). Cusrvature and lateral offset are also displayed in the image.









### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./examples/example.ipynb" (or in lines # through # of the file called `some_file.py`).  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines # through # in `another_file.py`).  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()`, which appears in lines 1 through 8 in the file `example.py` (output_images/examples/example.py) (or, for example, in the 3rd code cell of the IPython notebook).  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
