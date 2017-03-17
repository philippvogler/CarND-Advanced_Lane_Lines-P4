
# Advanced Lane Finding Project

The goals of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image0]: ./output_images/dist/original "Original"
[image1]: ./output_images/dist/undist "Undistorted"
[image2]: ./test_images/test6.jpg "Road Transformed"
[image3]: ./output_images/binary/5.jpg "Binary Example"
[image4]: ./output_images/warped/5.jpg "Warp Example"
[image5]: ./output_images/road/5.jpg "Fit Visual"
[image6]: ./output_images/overlay/5.jpg "Output"


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second and third code cell of the IPython notebook located in "./Advanced_Lane_Finding.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection utilizing the `cv2.findChessboardCorners`. I plot the images for illustration with `cv2.drawChessboardCorners`.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![Original][image0]
![Undistorted][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:

![Road Transformed][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at code blocks five to seven in `Advanced_Lane_Finding.ipynb`).  Here's an example of my output for this step.

![Binary Example][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform uses `cv2.getPerspectiveTransform` and `cv2.warpPerspective` which appear in code block seven of the IPython notebook.  The transformation of the image uses the preprocessed binary image (`preprocessImage`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```
src = np.float32([[img.shape[1]*(0.5-upper_width/2), img.shape[0]*height],
                  [img.shape[1]*(0.5+upper_width/2), img.shape[0]*height],
                  [img.shape[1]*(0.5+lower_width/2), img.shape[0]*bottom_trim],
                  [img.shape[1]*(0.5-lower_width/2), img.shape[0]*bottom_trim]
                  ])

dst = np.float32([[offset, 0],
                  [img_size[0] - offset, 0],
                  [img_size[0] - offset, img_size[1]],
                  [offset, img_size[1]]
                  ])
```
This resulted in the following source and destination points:

| Source        | Destination   |
|:-------------:|:-------------:|
| 588, 446      | 320, 0        |
| 691, 446      | 960, 0        |
| 1126, 673     | 960, 720      |
| 153, 673      | 320, 720      |

I verified that my perspective transform was working as expected by inspecting the  warped image to verify that the lines appear parallel in the warped image.

![Warp Example][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I stick close to the approach from Brian in his Q&A session. I designed a tracker class that tries to find the lane centroids in one half and one layer of the binary image.
To do so a convolution signal is calculated by using `np.convolve` in code block eight for each search window. These centroids are traced for the left and right lane / side of the image in the list `window_centroids`.
In code block nine the tracker class is used to track these curve centers to fit polynomials for the left and right lane using `np.polyfit`.


![Fit Visual][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in code block nine of the iPython notebook where I define the loop for image processing. Again I use `np.polyfit`to create `curve_fit` and than calculate the curve radius in meter:

```
curve_fit = np.polyfit(np.array(res_yvals, np.float32) * ym_per_pix, np.array(leftx, np.float32) * xm_per_pix, 2)


curverad = ((1 + (2 * curve_fit[0] * yvals[-1] * ym_per_pix + curve_fit[1]) **2) **1.5) / np.absolute(2*curve_fit[0])

```

xm and ym_per_pix are used to convert from pixel to meter.

The information is put on the image by using `cv2.putText`at the end of code block nine.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Here is an example of my result on a test image:

![Overlay][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_tracked.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

As mentioned above I sticked very close to Brians approach from his Q&A session. It was a very tough project and it took much more time than I had.

The pipeline likely fails in cases, where there are no line markings to detect. Very steep curves are also a problem, when the markings extend to the other half of the camera image. Heavy traffic with a lot of vehicles that are obscuring the line markings would make the detection difficult as well as snow, puddles or sand. Double lane markings in construction sides or additional old lane markings are problematic too.

There are several opportunities to improve the results. All the hard coded parts like source and destination points as well as the search window width and hight and the shape of the trapezoid could be optimized in iterative functions that measures the fit.
Especially in for the video processing a smoothing opportunities over multiple frames would probably increase the results.
Furthermore I did not investigate all the possible color spaces for creating the binaries. There might by improvement opportunities there as well.
