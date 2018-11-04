# Self-Driving Car Engineer Nanodegree


## Project: **Advanced Lane Lines on the Road** 
***
This is a CV based lane-marker detection and tracking implementation using standard OpenCV packages and advanced image processing functions. 

The entire pipeline is structured as follows. 

---
>**Advanced Lane Finding Project**

>The goals / steps of this project are the following:

>1. Camera Calibration
>    1. Compute the camera calibration matrix and distortion coefficients given a set of chessboard images
>    2. Apply a distortion correction to raw images
>2. Color Space & Gradient Processing
>    1. Use color transforms in HLS to extract useful color/level data   
>    2. Use gradients with Sobel operator and thresholding for a binary image
>    3. Apply a perspective transform to rectify binary image ("birds-eye view").
>4. Detect lane pixels and fit to find the lane boundary.
>     1. Determine the curvature of the lane and vehicle position with respect to center.
>     2. Warp the detected lane boundaries back onto the original image.
>     3. Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.  
>5. Video Processing Pipeline   
>     1. Image undistort
>     2. Color transform (use S-channel)
>     3. Gradient detector for edges
>     4. Perspective transform to dewarp camera image
>     5. Lane detect using historical tracking data
>     6. Calcualte curvature metrics
>     7. Warp back to original image geometry
***


## 1. Camera Calibration

Use checkerboard images to calibrate camera intrinsics. 
* Load reference calibration images
* Detect checkerboard corners for _valid_ images & calibrate camera sensor
* Undistort images using calibration matrix (instrinsics)

- - - - 

### 1.1 Camera Calibration & Distortion Correction
A sample calibration data frame used for camera matrix calculation. 

<img src="./Resources/camera_cal/calibration3.jpg" alt="Calibration Data Frame" width=450/>

Checkerboard detection usign OpenCV library, the camera fundamental matrix and distortion transformations are derived. We test these on the sample images to get the undistorted frame. A sample is provided below. 

<img src="./Resources/test_images_undistort/straight_lines1.jpg" alt="Undistorted Test Image" width=450/>

## 2. Color & Gradient Processing
The main task for this process pipeline is to extract the relevant lane-markers from the test images, the dominant colors for this selection being white & yelow.

After a suitable regio-of-interest mask and edge gradient thresholding, a bird's eye view is obtained by a warping transformation. The reason for this is to make it easier to detect and track lane marker lines across a camera frame by transforming the geometrically warped image into a rectangular view.    

### 2.1 Color Thresholding & Gradient Detection
The undistorted frames are then processed to pick up the white & yellow colors dominant in lane markers. Various color spaces were evaluated with a threshold selector for `RGB`, `HSV` and `Lab` color components.

A alternative color thresholding using `R` and `S` channels was also tried but the more robust 'white-yellow' color selector using the following thresholds was found ot be most robust. 

Using white & yellow thresholding on test images with the following component thresholds results in the following binary image for the sample test frame. 

| Color | Space | Min Threshold | Max Threshold |
|-:|-:|-:|-:|
|White|RGB|[100, 100, 200]|[255, 255, 255]|
|Yellow|RGB|[225, 180, 0]|[255, 255, 170]|
|Yellow|HLS|[20, 120, 130]|[45, 200, 255]|
|Yellow|Lab|[0, 0, 154]|[0, 0, 255]|

<img src="./Resources/test_images_undistort_whiteyellow/straight_lines1.jpg" alt="Binary Thresholded Test Image" width=450/>

### 2.2 RoI Mask & Gradient Thresholding
The frames are then processed on the selected RoI that maximizes the probability of lane marker detection while reducing unnecessary processing on parts of the image that are not likely to contain any useful information. 

The gradient detectors used were evaluated from `sobel'` and `laplacian` operators with various kernel sizes. A separate absolute, magnitude and angular threshold selection was done across the thresholded image. 

The RoI selected and gradient thresholded image used for further processing is sampled below. 

<img src="./Resources/test_images_gradient_rhs_grad/straight_lines1.jpg" alt="Gradient Threhsolded Test Image" width=450/>

### 2.3 De-warping with Perspective Transform
To aid in the lane marker detection, the gradient thresholded image is then de-warped using the camera distortion matrix computed in the earlier steps and `pickled`. 

For this process, we select a test image that shows the largest span of straight lane lines as below. 

<img src="./Resources/test_images_warped_gradient_binary/straight_lines1_linemarked.jpg" alt="Straight Line Marked Test Image" width=450/>

These are then mapped to a rectangular pixel region to generate the de-warping transformation. Both the warp and inverse-warp (de-warp) transformations are `pickled` for further processing. This process is done just once per camera setup and hence not part of the pipeline process used for the video frames. 

<img src="./Resources/test_images_warped_gradient_binary/straight_lines1_warped.jpg" alt="Warped (Bird's-eye View) Test Image" width=450/>

The binary gradient thresholded frame is then warped using the same warp transformation computed above to provide a bird's-eye view of lane marker lines. 

<img src="./Resources/test_images_warped_gradient_binary/straight_lines1.jpg" alt="Warped (Bird's-eye View) Gradient Test Image" width=450/>

## 3. Lane Boundary Detector
A couple of lane detector and tracking algorithms were tested based on the theory & practice covered in the course material. 

The primary lane marker detector before applying the searcher-tracker is asimple historgram peak search that picks out the lanes by detecting the two dominant peaks in the gradient thresholded image as shown below.

<img src="./Resources/test_images_warped_gradient_binary/straight_lines1_warped_histogram.jpg" alt="Warped Lane Histogram (bottom 1/4th)" width=450/>


### 3.1 Sliding Window Tracker
The sliding window tracker first sets up a series of search window pairs, one each for left and the right lanes that split up the entire frame into a given number of horizontal slices. This limits the search region for each lane to a small rectangular area that can work robustly for curved lanes. 

All non-zero binary gradient thresholded pixels that lie within the search rectangle are marked as `lane` pixels. As the slices are scanned from bottom to top, the next slice window centers are adjusted to be the mean of the previous slice lane pixels, but only if the count exceed some minimum detection threshold. 

The number of slices, the thresholds and the search window dimensions are all tunable and have been chosen after some exploration of across the test image & video data set. 

A second order polynomial is fit through all detected left & right lane pixels to provide an anlytic expression of the lane markers. 

A sample window search with the rectangular search windows and the identified lane pixels and markers is shown below.

<img src="./Resources/test_images_slidingwindow_lanemarkers/straight_lines1.jpg" alt="Sliding Window Lane-marker Search" width=450/>

Using a search window around the previous frame's detected polynomial allows a much tighter and temporally correlated detection. The plot below shows in green the region of search which takes advantage of the temporal correlation between video frames and hence can be a whole lot tighter than a brute force blank search.

<img src="./Resources/test_images_slidingwindow_lanemarkers/straight_lines1_history.jpg" alt="Sliding Window Lane-marker Search (with history)" width=450/>


### 3.2 Convolutional Search Tracker
The convolutional search algorithm on the other hand uses a mask to convolve the thresholded binary image with a window mask that shows peak where the mask and lane marker pixels coincide. 

Various order polynomials were tested with weighted centroids (i.e. selecting centroids where normalized convolution peak was a least 30%-tile of the centroids in all the selected search windows for each of left & right lanes). A final selection was made with 2nd order polynomials as below.   

<img src="./Resources/test_images_slidingwindow_lanemarkers/straight_lines1_conv.jpg" alt="Convolutional Lane-marker Search" width=450/>

### 3.3 Lane Marker Annotation
Once the lane markers are detected and modelled, the drivable surface can then be idnetified as the surface encompassing the left & right polynomials. The lanes & the surface can then be warped back to the original space using the `pickled` geometry inversion matrix. 

With the left & right lane polynomials evaluated, the radius of curvature is then computed in pixel & `km` domain. Other useful metrics such as deviation from center track can also be then annotated onto the frame. 

<img src="./Resources/test_images_annotated_lanes/straight_lines1.jpg" alt="Annotated Lane-marker Detection" width=450/>

---

## 4. Video Pipeline Implementation
The video pipeline is then just a sequential function call of the various tuned implementations discussed above. These have been implemened in the jupyter notebook [here](.\VideoPipeline.ipynb)

The project test vidoe output processed from the pipeline is availabele [here](./Resources/test_videos_output/project_video.mp4)

---

## 5. Discussion

A few enhancements that can make the implementation more robust:

    1. A combined sliding window and convolution search dynamically selected when one fails to detect reliable lane markers. 
    2. Using a frame averaged lines from multiple previous frames. 