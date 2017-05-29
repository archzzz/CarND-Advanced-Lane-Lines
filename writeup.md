## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/undistort.jpg "Undistorted"
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./output_images/binary.jpg "Binary Example"
[image4]: ./output_images/perspective.jpg "Warp Example"
[image5]: ./output_images/color_fit.jpg "Fit Visual"
[image6]: ./output_images/result/image0000.jpg "Output"
[image7]: ./output_images/bad1.jpg "Bad Image"
[video1]: ./output_images/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./submit/advancedLane.ipynb"

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I tried magnitude and direction of gradient, RGB and HLS color space, finally choose to use S channel in HLS color space to find yellow line with threshold (160, 255), and gradient direction of (60, 120) to find white lanes. The code is in function `binary_image()` in step #4 in `advancedLane.ipynb`. Threshold is passed in pipeline in step #8. Here's an example of my output for this step.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `perspective_transform()`, which appears in step #5 in `advancedLane.ipynb`.  The `perspective_transform()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
    src = np.float32([[602,450], [270,720], [1080,720], [682,450]])
    dst = np.float32([[320,0], [320,720], [960,720], [960,0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 602, 450      | 320, 0        | 
| 270, 720      | 320, 720      |
| 1080, 720     | 960, 720      |
| 682, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I use sliding window to fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

Code in step #6 in `advancedLane.ipynb`
First i use histogram to find the starting point of the line, then i use sliding window with margin 100 to detect the line. I also applied sanity test that checks if the bottom line is roughly 3.7m ((650, 800) pixel). If past the test, i update the cache. Otherwise, this frame is consider as a bad frame and skiped. 

If the line is detected in previous frame, i uses look ahead to find the line, it saves the effort of searching blindly. And apply sanity test that checks the diff of every x on the curve is less than 30 pixel. If past the test, i update the cache. Otherwise, it falls back to sliding window.

I use a Line object to save caches, which as a boolean of line detected, recent_xfitted that stores fitted x values in last frame, and bestx used to plot on output image, new bestx is calculated by bestx*0.3 + recent_xfitted*0.3 + current_xfitted*0.4, and coefficient of current polynomial.
Note that i use bestx as the result so that the image is smooth, and use recent_xfitted to do look ahead search since it's more accurate.

Also, i didn't use curvature and parallel in sanity check, because image is distorted after perspective transformation, even if detected lines are correct, they don't seem to have similar curvature or parallel on warped image. Since a loose check is not useful, i removed curvature and parallel in my sanity check.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in step #7 in `advancedLane.ipynb`. 

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in step #8 in `advancedLane.ipynb`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_video.mp4)

I first write all images to output_images/result/ , then use video.py in behavior cloning project to generate a video.
I tried cv2.VideoWriter, it succeed in open the writer, but cannot write images to video.

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Two biggest issue i had are creating binary image and sanity check. 
I tried RGB and HLS, HLS is good in finding yellow line, but i don't know how to find white color only. And i take the union of all filters, which introduce noises. Instead of union, if i can use intersection and extract useful parts from different filter and add them to one image, it maybe better.

I also don't have an effective strategy in sanity test. First, it's difficult to use perspective transformation to get parallel lines, top area in the warped image is amplified and distorted. In some cases if two lines are parallel, result is like this:

![alt text][image7]

So i can just apply a very loose condition, which is not helpful. 
