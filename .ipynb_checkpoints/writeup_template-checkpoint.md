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

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./pipeline.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function the result is located in "./output_images/undistorted.jpg": 

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like the one located in "./output_images/undistorted-car".
I used `cv2.undistort()` with the `img` argument and the `mtx` and `dist` arguments calculated previously using `cv2.calibrateCamera()`, `cv2.undistort()` returns the distortion corrected image.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used 3 different thresholds, first in the R color channel of the original RGB image, second was the S channel of the HLS image (used `cv2.cvtColor()` to convert the original image to an HLS image), finally I used the magnitude of the gradient on a grayscale image (again used `cv2.cvtColor` to convert the image to grayscale, sobel operators were used to find the magnitude of the gradient). I defined helper functions at cell 6 and the `getEdges()` function in cell 7 in the notebook located at `./pipeline.ipynb`. An example of my output for this step is located in "./output_images/binary_edges1".  

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `Warp()` and a function called `getTransforms()`, they appear in the 11th code cell (I didn't count markdown cells) in the notebook at "./pipeline.ipynb".  The `getTransforms()` function takes as inputs an image `img`, the function defines `src` and `dst` as seen below, It gets the transform to a bird's eye view as well as the inverse transform, it does this using `cv2.getPerspectiveTransform()` twice once from `src` to `dst` and once from `dst` to `src`.
The function returns `M`, the transformation to a bird's view and `Minv` the inverse transformation.
The `Warp()` function takes as input an image (`img`) this is the image to apply the transformation to, it also takes as input `M` the transformation to be applied. The function then uses `cv2.warpPerspective()` to apply the transformation to the image. I chose to hardcode the source and destination points in the following manner through experimenting with a quadrilateral's boundry points:

```python
src = np.float32([[img.shape[1] - 600, 450],
                  [img.shape[1] - 170, img.shape[0]],
                  [170, img.shape[0]],
                  [600, 450]])

dst = np.float32([[img.shape[1] - 300, 0],
                  [img.shape[1] - 300, img.shape[0]],
                  [300, img.shape[0]],
                  [300, 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 680, 450      | 980, 0        | 
| 1110, 720     | 980, 720      |
| 170, 720      | 300, 720      |
| 600, 450      | 300, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image. an example of my output is provided in "./output_images/birds-eye-view1".

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

In order to identify the lane lines I wrote a function called `find_lane_pixels()`, it is located at the 14th code cell lines 1 through 67, the notebook is in "./pipeline.ipynb". The function takes as input `warped` and `rect`. `warped` is a binary bird's eye view of the image and `rect` is a boolean which tells the function if it should draw the rectangles on the image or not. lines 2 - 6 find the middle of the image (in the X direction) and calculate the left and right bases for the lane lines, it finds the bases by a vertical sum over the bottom half on the image it then uses `np.argmax()` on the left half of the sum and on the right half of the sum. `np.argmax()` returns the index of each of those spots. The bases are where the search for the lane lines begins. From line 10 to line 14 the sizes of the windows are defined as well as the amount of windows and the minimal amount of nonzero pixels spotted in a window to recenter the next window. lines 15 - 17 get the X and Y coordinates of nonzero pixels in the image (nonzero pixels are pixels that have been identified as edges earlier). lines 19 - 20 define `leftx_current` and `rightx_current` these are used to store the current center for each of the windows, lines 22 - 23 create 2 lists called `left_lane_inds` and `right_lane_inds` to keep track of the indices of the left and right lanes. line 26 makes a 3D (3 channels) image so that later we will be able to color the lanes, in line 28 we enter a loop, it loops over the amount of windows. lines 30 - 37 find the bounds of the current windows in the loop (there is one window for each lane), lines 39 - 42 draw the rectangles if the `rect` argument is true, lines 45 - 46 get the coordinates of the nonzero pixels in the window for each of the windows. Lines 49 - 50 append the coordinates found in this iteration of the loop to `left_lane_inds` and `right_lane_inds` to store them for later use. Lines 53 - 56 recenter the windows if the amount of nonzero pixels found in the window was more then minpix (the minimal amount of pixels to recenter the window). At this point we are out of the loop. Lines 58 - 59 concatenate the `left_lane_inds` and `right_lane_inds` (flattening them). Lines 62 - 65 get the X and Y coordinates for each pixel identified as a part of the lane lines. Finally, the function returns 4 lists, 2 lists for the X and Y coordinates of the left lane and 2 more for the right lane. It also returns the image with the rectangles drawn and the bases computed earlier.

In order to color the lanes and fit a 2nd degree polynomial to them I wrote another function called `fit_poly()` it is located at the 14th code cell lines 69 - 96 it takes as input the X and Y coordinates for both lanes and the image returned from `find_lane_pixels()`. lines 72 - 73 call `np.polyfit()` twice with the X and Y coordinates for each of the lanes (note that a 2nd degree polynomial will be fit for the lanes). The returned coefficients are stored in variables called `left_fit` and `right_fit`. Line 85 uses `np.linspace()` to get the domain of the function (Y coordinates). Lines 88 - 89 get the X coordinates from the domain and the coeffients from each of the lines. At lines 92 - 93 the left lane is colored red and the right lane is colored blue. Finally the function returns the image, the X coordinates for both of the lanes, the domain and the coefficients for each of the polynomials.

An example of my output is provided in "./output_images/lanes_fit1.jpg".

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in code cell 17 at lines 1 through 21 in my code in `./pipeline.ipynb`. I defined a function called `getCurvatureAndPosition()` that takes as input the coefficients for both polynomials fit earlier, the domain for the functions (list of Y values), the image output from the `fit_poly()` function and finally the bases for both lanes calculated in `find_lane_pixels()`. On lines 8 - 9 `ym_per_pix` and `xm_per_pix` are defined, these are used to convert from pixel space to meters. On line 11 I get the max item from the domain (the lowest pixel in the image) to evaluate the curvature at that point. On lines 13 - 14 I use the equations given in the lesson to calculate the Curvature using the inputs. On lines 17 - 19 I calculate the position of the car with respect to the center of the lanes, first I take the mean of the bases to get the middle of the lanes then I get the middle of the picture, finally I compute the distance between them andd convert from pixel space to meters. The function returns the `min()` of the curvatures and the location of the car.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in code cell 19 and 21 the jupyter notrbook is in `./pipeline.ipynb`. I defined two functions `colorOriginal()` and `colorInsideLane()`, the first function colors the lanes in the original image, the second function colors all pixels between the lanes green. An example of my result on a test image is in "./output_images/final_output.jpg".

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

My final output video is in "./output_images/VIDEOOUTPUT.mp4".

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

Approach & techniques: 
For the most part I was using the techniques and ideas given in the lessons with some mild changes when necessary, some of the steps in the pipeline were not covered in the lessons so I came up with my own ideas and implementation for them.

Possible failures / shortcomings:
(1) The detection of the right lane is wobbly most of the time.
(2) At the parts in the video where the color of the road changes quickly the detection becomes less accurate.
(3) Detection at night time (low lighting) would be less accurate.

(hopefully) Addressing some of the failures:
addressing (1): Connecting the right lane edge pieces could make detecting it as consistent as the left lane.
addressing (3): Finding a color space that detects the edges well and is consistent in both daytime and nightime.

General ideas for improvement:
(1) As suggested in the rubric, since the frames have correlation between them, after detecting the lanes the first time using the rectangels we could use another method which already uses what's known instead of going blindy into a search again, for example we could search in a region around the polynomials fit earlier etc.