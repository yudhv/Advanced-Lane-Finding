#Advanced Lane Finding Project

## Brief Summary
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

[image1]: ./examples/undistort_output.png "Undistorted"
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./examples/binary_combo_example.jpg "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## Detailed Summary
####Here I will describe how I addressed each of the above-mentioned points in my implementation.  

---
###Camera Calibration and Distortion Correction

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the third code cell of the IPython notebook "main.ipynb."

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

###Color and Gradient

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

I computed the Sobel derivative of the test image in the X direction. A Sobel operator computes the gradient of the "image intensity," thus emphasizing the edges of an image. This was particularly helpful as it emphasized on the lane lines more than other features in the test images. To compound this, I also converted the image to HLS (Hue, Lightness, Saturation) color mode. I found that the lane features were most augmented in the Saturation channel, and hence, I added a thresholded Saturation channel to the Sobel image created earlier and finally produced the resultant binary image as output. This code can be found in the `color_n_gradient()` function in the fourth cell.

The resultant image looks like this:

![alt text][image3]

###Perspective Transformation

The fifth and sixth cells in the IPython notebook are tasked with creating a Perspective transform of the image to make the lanes appear as if we are standing right "above" them. Theoretically, this transformation should produce parallel lines, as is true for the lanes on a road. 

In-order to accomplish this, I first crop the upper half of the image and form a trapezoid area of interest that is most likely to comprise of the lane lines. Next, I use the HoughLinesP function on the previously generated bitwise image to get all the lines. The HoughLinesP transform is a much faster version of the Hough Transform, as it uses a Probabilistic approach to finding lines. Coupling these lines with a slope filter, I get a vague idea for where the lane lines are, and use it to form the four points for a trapezoid that will later be used for the Perspective Transform. 

In-order to effect a Perspective Transform from right above, I use parallel lane lines fact and choose four destination points that coincide with the 1280 * 720 image corners (as our image's sides are also parallel). Next, the `cv2.getPerspectiveTransform()` function is fed both the 'src' (original points identified from the lanes) and the 'dst' (destination points selected hypothetically) points, and a transform matrix is generated. This matrix is then fed to the `cv2.warpPerspective()` function to generate the final warped image. 

Here is an example of the source and destination points generated from the above image:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto the test image:

![alt text][image4]

And here is what the resultant warped image looks like:

![alt text][image4]

###Polynomial fitting

In-order to fit a polynomial to the lanes, I first had to identify the lines on a pixel-by-pixel basis. For this, I first found the base points of the lane lines by looking at the histogram summations of X columns in the image. The column with the highest sum was bound to be closest to the lane lines (as lanes are parallel and biased towards the y-direction). This code can be found in the 7th cell in the `line_detect()` function. 

Here is the histogram image:



Next, a hypothetical window is created and moved in the upward direction along the previously detected X column. During its upward journey, I find the x values for columns with the most non-zero points inside the window. I reiterate this method for the other lane as well. This process allows for the window's height level accuracy of detecting the lane lines. 

Here is what the windowed image looks like, in order to better understand the process:

![alt text][image5]

Finally, a polynomial is fit to the above-mentioned x-points for the lane lines. The function `np.polyfit()` is used for this. It returns the values of the parameters 'A', 'B', and 'C' for the second order polynomial equation -
####Ax^2 + Bx + C
A smoother line generated using the polynomial looks like this:

![alt text][image5]

###Radius of Curvature

According to the [US traffic manual](http://onlinemanuals.txdot.gov/txdotmanuals/rdw/horizontal_alignment.htm#BGBHGEGC), the normal road is 3.7m wide and a single line is 30m long. Using these assumptions, I measured a meter-to-pixel ratio in the x and y directions:
```
ym_per_pix = 30/720 # meters per pixel in y dimension
xm_per_pix = 3.7/700 # meters per pixel in x dimension
```
The code to find radius of curvature can be found in 9th shell of main.ipynb.

In addition to the radius of curvature, the distance of the car from the center of the road is also computed. This provides an easy and efficient way to guide the car back to the center of the lanes.

The logic for distance from the center of the lane can be found in cell 10.

###Visualize output

I implemented this step in the 13th cell under the function `show_finished()`. This function takes as arguments the original image, the left lane polynomial, the right lane polynomial, the binary warped image, and the inverse matrix for the previously performed perspective transform. The inverse matrix was the 'Minv' array that was outputted by the `get_warped()` function. This matrix allows the warped plotting of the lanes to be converted back to the original image, thus giving us the final output.

Here is an example of my result on a test image:

![alt text][image6]

---

##Pipeline (video)

In-order to make this code work for videos, a few incremental changes were needed:

* Using an iterative method of lane finding after the first lane was found so as to reduce computational complexity
* Using a sanity check mechanism to avoid plotting any lines that weren't parallel, had vastly different curvatures, or had vastly different slopes. 
* Using previous 5 polynomail fit instances to smooth out the plotting in the video, and make it look more elegant.

###Iterative Lane Finding
Here, the previously found polynomial fits are used along with a margin window of 100 pixels. This allowed me to avoid going through the entire process of finding the lane lines from the start for each new frame. Generally, road lanes tend to remain non-abrupt, more so when each frame represents 1/30th of a second. Hence, a margin of 100 pixels allowed me to note any partial changes in the lane curvatures across each frame. This process helped in heavily reducing the burden on the host machine. This logic has been implemented in the `lane_detect_after()` function in cell number 8. 

###Sanity Check
Due to the nominal changes that occur in each frame, we can easily use the previous frame's computation to guide lane line plotting. Here, three simple checks are made; first, checking if the lines are within a certain slope range (done right at the source in the `process_image()` function), second, checking if the left and right lanes have similary radius of curvatures, and third, checking if the lanes are at least 700 pixels (3.7 meters) wide apart. The second and the third checks can be found in the `sanity_check()` function in cell number 11. Whenever, a check fails, a red colored 'RESET' text is displayed on the frame, and the global 'attempted' boolean is turened to 'False.' This kicks in the original lane finding function `line_detect()`, thus making the pipeline more robust to false positives. 

To use the previous frame's computations, I created a ```Line``` class that stored the last 'n' number of polynomial fits, or lane lines for both the right and the left lane. This code can be found in the 2nd cell of the IPython notebook.

###Smoothening
Finally, the last 'n' values stored in the Line class are averaged out to give a smoother, less wobbly lane area plotting. I found 3-5 to be the ideal range for n. Anything above that made the plotting lag behind the actual curve on the road, and anything smaller would make the plotting more wobbly. 

Here's a [link to my final video result](./output_video.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

The pipeline works well on the given video. However, with videos like [this](./challenge_video.mp4) and [this](./harder_challenge_video.mp4) the pipeline fails. In general, roads with too much traffic, imperfect roads with bad patching artifacts, and roads without clear lane lines would pose a problem for the given pipeline. 

TO-DO: 
* To make the pipeline more robust. 
* To include a 3.7m wide clause at the initial stages of the pipeline (while finding lanes). 
* Use Machine Learning to tackle the harder challenges in this project. 
