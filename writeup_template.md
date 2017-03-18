## Advanced Lane Finding Project

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

[image1]: ./output_images/undistort_output.png "Undistorted"
[image2]: ./output_images/undistort_test.png "Road Transformed"
[image3]: ./output_images/binary_combo_example.png "Binary Example"
[image4.1]: ./output_images/warped_straight_lines.png "Warp Example"
[image4.2]: ./output_images/warped_straight_lines2.png "Warp Example 2"
[image5]: ./output_images/color_fit_lines.png "Fit Visual"
[image6]: ./output_images/example_output.png "Output"
[video1]: ./easy.mp4 "Video"

## [Rubric Points](https://review.udacity.com/#!/rubrics/571/view)
Here I will consider the rubric points individually and describe how I addressed each point in my implementation.

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  
You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./P4.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the real world space. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

### Pipeline (single image

#### 1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will show how I apply the distortion correction to one of the test images like this one:
![alt text][image2]
#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I used a combination of color and gradient thresholds to generate a binary image. The code is in the third code cell of the Jupyter notebook, functions `sobel_filter()` and `color_filter()`.

I created an interact widget in the Jupyter notebook to be able to play with different values and instantly see the results, so I could fine-tune the parameters much faster than changing the code and rerun the cell. These are the widget options to adjust color transforms and gradients, and their final selected values:
1. Sobel X range (50-255)
2. Sobel Y range (250-255)
3. Sobel Magnitude range (60-220)
4. Sobel Direction range (0.7-1.2)
5. Kernel size (3)
6. Hue (HLS) range (0-70)
7. Lightness (HLS) range (0-195)
8. Saturation (HLS) range (85-195)
9. White (HSV) sensitivity (35)
10. Yellow (HSV) sensitivity (35)

Sobel filter: ((X range) and (Y range)) or ((Magnitude range) and (Direction range))
Color filter: (white or yellow or HLS range)

To extract white and yellow in the color filter I transformed the image to the HSV color space.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp_image()`, which appears in the second code cell of the Jupyter notebook. The `warp_image()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points. I also used an interact widget to choose the values of the apex and the base offset in the source points and x offset in the destination points in the following manner:

```
apex_loff, apex_roff, apex_yoff, base_loff, base_roff, base_yoff, warp_xoff = region_params

apex_left = (width / 2 - apex_loff, height / 2 + apex_yoff)
apex_right =(width / 2 + apex_roff, height / 2 + apex_yoff)
base_left = (base_loff, height - base_yoff)
base_right = (width - base_roff, height - base_yoff)

vertices = np.int32([base_left, apex_left, apex_right, base_right])
warped_vertices = np.int32([[warp_xoff, height], [warp_xoff, 0], [width - warp_xoff, 0], [width - warp_xoff, height]])

```
This resulted in the following source and destination points in (X,Y) coordinates:

| Source        | Destination   |
|:-------------:|:-------------:|
| 250, 690      | 380, 720      |
| 582, 460      | 380, 0        |
| 702, 460      | 900, 0        |
| 1062, 690     | 900, 720      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto test images with straight lanes and its warped counterpart to verify that the lane lines appear parallel in the warped images.

![alt text][image4.1]
![alt text][image4.2]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I decided to use convolutions to find the lane-line pixels from the warped binary images, because I thought it is a more elegant solution than the other proposed sliding window algorithm. However, I struggled fine-tuning the convolution algorithm to detect the lanes accurately. For example, I found the squared window to apply convolution as proposed in the course lessons caused an accuracy problem depending on the warped lane width. If the lane width was small compared to the window size, the detected point was always on the left of the lane center, because the `argmax()` function only returns the first convolution maximum. I then decided to use a triangular window, leading to more accurate results when finding the maximum convolution. I also discarded points with very low convolution value, to avoid sobel/color noise when the lane is discontinuous.

The parameters to fine-tune the lane finding algorithm and their final selected values are the following:
1. Window X size (80)
2. Window Y size (60)
3. Margin to look for new points left and right (80)
4. Convolution threshold value to consider a point part of the lane (30)

After detecting the lane points, I fit them to a 2nd-grade polynomial and plot the resulting curve:
![alt text][image5]
<span style="color:#0000FF; background:#000000">Blue: Color Binary Filter</span>
<span style="color:#00FF00; background:#000000">Green: Gradient Binary Filter</span>
<span style="color:#FFFF00; background:#000000">Yellow: Found lane points</span>
<span style="color:#FF0000; background:#000000">Red: 2nd-grade polynomial curve</span>

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in the third code cell of the Jupyter notebook, in the `get_lane_curvature()` function, which receives  the lane points and returns the radius of each lane and the lanes offset from the center of the image. First, it gets a 2nd-grade polynomial fit from the lane points translated to meter scale. Then it calculates the radius of the curvature by applying this formula to the bottom point (max y):
f(y)=Ay^2+By+C
Radius = (1+(2Ay+B)^2)^1.5 / ​∣2A∣

To calculate the offset from the center of the image, I obtained the middle of the lane based on the polynomial evaluation at the bottom point (max y) and subtract the center of the image (max x / 2).

In order to translate lane points from pixels to meters, I figured out the meters per pixel based on the warped road image in the following way:
 - x axis: 3.7 meters (standard US road width) divided by 520 pixels (measured in the warped road image)
 - y axis: 3 meters (standard US dash line length) divided by 74 pixels (measured in the warped road image)


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step the third code cell of the Jupyter notebook, in the function `pipeline()`.  Here is an example of my result on a test image:
![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./easy.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.

After the camera calibration, my next step was to find out the region of interest to warp the image to get the birds-eye view. I used an interact widget to facilitate this task (Code cell 2). The challenge here was to select the right points so that the lanes of the first 2 images look completely straight in the warped image. This was particularly difficult for the right lane.

Then I implemented all the remaining steps of the final pipeline at the third code cell. In the 4th code cell, I sent each test image through the pipeline and plotted several intermediate images along with an interact widget to fine-tune the parameters for each step of the pipeline. In particular, I plotted the final output image, a debug warped image to show the found lane points and the polynomial fit, and the color and gradient binary images also for fine-tuning purposes.

Regarding color and gradient parameters, the challenge was to find the ones giving the most accurate lane lines without adding too much noise.

Regarding lane finding parameters, the challenge was to find the ones giving the most accurate points within the lanes while ensuring there were enough of them to fit the polynomial. Thanks to the triangular window, the accuracy of the points within the lane was good enough.

After fine-tuning with the test images, I tried the video and saved the frames where the pipeline would fail to find the lanes properly. I fine-tuned again with those problematic images to make the pipeline more robust.

**Challenge Videos**
Once I got a stable pipeline with the project video, I tested with the challenge ones and found really disappointing results.

In the first challenge video, the lines are blurred at the far end of the road, thus making my model to fail identifying the end of each lane.

In the second challenge video, the high curvature of the lanes along with very different lightning conditions make my model completely useless.

To pass the challenge videos, I believe I should implement the following:
1. Adaptive region of interest, ideally based on car speed for example.
2. Increase image contrast, using CLAHE (Contrast Limited Adaptive Histogram Equalizer) for example.
3. Improve lane detection based on history, discarding outliers and smoothing transitions between frames.
