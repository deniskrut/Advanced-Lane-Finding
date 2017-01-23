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

[image1]: ./images/undistort_test.png "Undistorted"
[image2]: ./images/undistort.png "Road Transformed"
[image3]: ./images/binary.png "Binary Example"
[image4]: ./images/warped_straight_lines.png "Warp Example"
[image5]: ./images/fit_lines.png "Fit Visual"
[image6]: ./images/example_output.png "Output"
[video1]: ./project_video_solution.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
###Writeup / README

####1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!
###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in `./advanced-lane-finder.ipynb`.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
Using undistortion matrix obtained at the calibration step, I execute `cv2.undistort` function to undo the distortion. You can see result below:

![alt text][image2]

####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I’ve observed that detection works better on S channel of HLS on some set of images, while on others L works better. So I used OR combination to get both. I also noticed that some false positives can be masked by threshold on the S and L channel of HLS, so I applied that step as well.

The code for this step is contained in the second code cell of the IPython notebook located in `./advanced-lane-finder.ipynb`.

Here is what result looks like:

![alt text][image3]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for this step is contained in the third code cell of the IPython notebook located in `./advanced-lane-finder.ipynb`. Function `change_perspective` takes `image` as a parameter, and optional `reverse` flag that allows reverse transofrmation.

Some of the sample images have been of varying resolutions, so I have chosen source and destination points relative to the image. Here is how source and destination points are defined:

```
src = np.float32([
[image.shape[1]*0.4475, image.shape[0]*0.65],
[image.shape[1]*0.5525, image.shape[0]*0.65],
[image.shape[1]*0.175, image.shape[0]*0.95],
[image.shape[1]*0.825, image.shape[0]*0.95],
])

dst = np.float32([
[image.shape[1]*0.2,image.shape[0]*0.025],
[image.shape[1]*0.8,image.shape[0]*0.025],
[image.shape[1]*0.2,image.shape[0]*0.975],
[image.shape[1]*0.8,image.shape[0]*0.975],
])

```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 573, 468      | 256, 18       | 
| 707, 468      | 1024, 18      |
| 224, 684      | 256, 702      |
| 1056, 684     | 1024, 702     |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for this step is contained in the fourth code cell of the IPython notebook located in `./advanced-lane-finder.ipynb`.

If I don’t have prior assumptions about where the lanes are, I do a full search (`get_lane_points_bottom_up` function). First I take bottom quarter of the image and get a histogram and peaks on that histogram (`sorted_peaks` function). Form that peaks I form a pairs of peaks that are likely represent lanes - based on distance between the peaks (`get_lanes_candidates` function). I sort those pairs in a way that lanes containing highest peaks go first. Let’s call those pairs a candidates for lanes.

I iterate through candidates looking for ones that will yield lanes for me. I cut image to 8 pieces horizontally, and for each horizontal piece starting from the bottom I try to find pixels that are close to the candidate point (`get_points_around_x` function). Coming to the next piece I update my candidate point to be mean of all lane points for each lane from the previous piece. If there are no white pixels at given location, I ignore it.

The process above stops when I find good enough lane points. I have certain thresholds for what I believe good enough lanes are in terms of bend, slope, how close to a parallel they are and how much residual did I get from the polynomial fit (`is_good_points` function). If I cannot find good points, I return the best result I have been able to obtain. To define which lanes are better, I wrote a scoring function (`score_lanes`). That function favors lanes that are parallel, have about the same bend, slope, greater number of points and lower residual from the polynomial fit for the points associated with each lane.

If I have information about lane points found in the prior frame, I try to search for lanes near where they used to be (`get_lane_points_nearby` function). For that I divide image into 8 horizontal pieces, obtain lane location from previous polynomial fit for each piece (`get_x_for_line` function), and search for non-zero pixels in the vicinity of that location (`get_points_around_x` function).

Image below demonstrates polynomial fit for lane points. Left lane is in red, right lane is in blue. Polinomial fit lines are in green.

![alt text][image5]

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The code for this step is contained in the fourth code cell of the IPython notebook located in `./advanced-lane-finder.ipynb`. Function is called `get_curvature_and_distance_from_center`. It accepts left and right lane coordinates, image width and image height, and returns the curvature and offset from center in meters.

First step is to obtain polynomial for the lane lines in meters, not in pixels. For that, knowing the US government requirements for lane width and dashed line distance, I estimated the conversion coefficients and performed a fit. Second step is to find the line in between of two lane lines. This is done through evaluating polynomial fits for each lane at every vertical position, and finding the center. Third step is to find polynomial fit for the center line. Finally, I use first and second derivative to obtain a curvature.

To obtain center offset I evaluate X position of left and right lane polynomials, and find point in the middle (in meters). Then I find the distance from the center of the image (in meters) to that middle of the lane position. That distance is the center offset.

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

To warp lanes back to original image, I perform a drawing of lanes in the bird eye view image, and then transform it back to a normal view.

The code for this step is contained in the fifth code cell of the IPython notebook located in `./advanced-lane-finder.ipynb`. Function is called `draw_lane`.

Here is an example of my result on a test image:

![alt text][image6]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

In order to get video to work, I have declared a global variables to pass results from previous frame to the next frame. If 10 frames in a row did not fit the criteria for good lanes - e.g. parallel, low residual, etc. - I start search from scratch as described above. Otherwise I perform an informed search. See `write_clip` function.

Here's a [link to my video result](./project_video_solution.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Before trying the current approach, I've tried more complete searches that would not rely that we get a correct peak for the lanes in the bottom quarter of an image. For that initially I took all possible candidates for each 1/8th of an image, and iterated through all of them, with higher peaks first. That obviously took tremendous amount of time, and also did not work very well, because lanes might be matched with some artifacts on the image that would give lanes better score.

Other approach I've tried was successive refinement of set of candidates. Starting with some set of candidates for lanes for each of the 1/8th of an image, I would peak other candidate that would improve the score. I did it until lanes were "good", or until score for lanes stops improving. This did have better performance than previous approach, but still suffered from peaking random artifacts and declaring them lanes.

If I had more time, I would implement a better binary image processing. I think better results can be achieved by trying more color models and channels. This would give me better results on challenge videos, that can fail due to false positives in the binary image

I also could partially remove assumptions I gave for a good lanes to accommodate better for harder challenge.

Finally I can incorporate knowledge of the flow - e.g. which way we are moving. I should be able to tell which lanes roughly stay through out multiple frames, and which one are coming and going.

Current pipeline fails if assumptions are violated - e.g. turn is so steep that only one lane is visible, while we try to find both lanes. Certain road colors can add false positives to the picture, causing pipeline to fail.
