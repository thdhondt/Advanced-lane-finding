# Advanced lane finding

The aim of the present project is to determine the position of lane lines around the vehicle. For this purpose, a front-facing camera is used to capture a video of the road on which the car drives. Those images are then processed in order to extract the information related to the lane lines. This is done by combining thresholds applied in different color spaces and on the gradients in the image. This binary image is then transformed to remove the camera distortion and to obtain a bird's eye view on the road. Histogram methods are used to extract the pixels associated to a given lane line. Finally, the fitting of polynomials to those pixels can be used to extract the curvature of the road and the distance of the vehicle with respect to the lines.

This lane finding method is considered "advanced", since:
- it should work in varying lighting conditions
- it considers curving lane lines 
- it works for different colors of lane lines
- it computes the lateral position of the vehicle in the lane
- it estimates the curvature of the lane

The different steps used in order to attain this goal are summarized in the following figure. Those points will be further described in the next sections.

![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/process.jpg?raw=true)

## Camera distortion
Cameras are using lenses in order to focus light on their sensors, hereby reducing the time required in order to form an image. They also allow the user to adapt the field-of-view of the camera and the point that is in focus. However, lenses tend to induce distortions in the captured image, which are most noticeable at the borders of the image. 

Those deformations can be modeled through a radial and tangential model, which link the position of the pixel in the undistorted image to their position in the distorted one. Hence, knowing the coefficients in those models allows to undistort images taken by a given pair of camera and lens. This is a required step, if one wants to obtain exact geometrical information on the scene from the capture images.

For this purpose, the camera is used to take images of an object with a known geometry. By comparing the captured geometry with the known real geometry, the deformation induced by the lens can be assessed. This is usually realized using checkerboards. This computation is done considering several calibration images, in order to obtain a better estimate of those parameters. The resulting workflow is resumed in the following diagram.

![](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/calibration.PNG?raw=true)

The estimated camera parameters can then be used in order to undistort images captured while driving on the road.

![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/undistort.PNG?raw=true)

## Perspective transformation

The detection of lane lines and the estimation of the corresponding geometrical parameters can be facilitated by looking at the scene from a bird's eye point of view. Changing the initial point of view to this top down image is called a perspective change.  

This step is achieved through an affine transformation. It is using four points from the initial image which should form a rectangle in the scene. Due to the effect of perspective, those points usually end-up having the shape of a trapeze in the end image. Those four corners are then matched to their position if this rectangle was viewed from above. Those two sets of points are linked by an affine transform, which can be applied to any subsequent image, assuming that the position of the camera on the vehicle didn't change. The process of a perspective transform is resumed in the following image.

![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/bird%20process.PNG?raw=true)

As an example, this function is applied to several other images taken by the camera mounted on the vehicle.

![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/input.PNG?raw=true)
![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/bird.PNG?raw=true)


## Image thresholding

In order to fit a polynomial to the lane lines, one should first identify which pixels belong to the left and the right lane lines. For this purpose, the input image should be filtered in order to extract those pixels. Since lane lines tend to be brighter than their surroundings, this can usually be achieved through thresholding of the image. However, one should be careful that this filter should work in varying light conditions around the vehicle and for different lane line colors.

In order to make the filter more robust to those variations, it is good to make it rely on a set of filters, whose output is then combined. Those thresholds can be applied on the color intensities in several colorspaces and on the gradients in the image. The following diagram resumes all the filters and their combinations used in this project.

![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/thresholding.PNG?raw=true)

### Gradient filtering

In order to compute the gradient of the image, the sobel operator is used in different combinations:
- Absolute sobel x : Absolute value of the gradient in the image along the X-axis
- Absoluted sobel y : Absolute value of the gradient in the image along the Y-axis
- Sobel magnitude : Magnitude of the X and Y components of the sobel operator for each pixel
- Sobel direction : Angle of the sobel vector 

### Color filtering

Different colorspaces are more or less influenced by the color of the lane line and by the brightness information. Therefore, information from:
- V channel in HSV
- S channel in HLS
- B channel in LAB
- R channel in RGB
was combined in order to capture all the possible variations.

### Bump filter

The last filter exploits both the color information and the gradient. It is based on the idea that the pixels around a lane line are less bright than the pixels in the lane. For this purpose, the filter takes a input a kernel width and is going to iterate over all the columns of the image. It replaces each column in the image by its intensity minus the average of the intensities one kernel width before and after the considered column. This means that lane lines whose width matches that of the kernel will respond very strongly in this filter. 

## Polynomial fitting

Once the image is thresholded and its perspective is transformed, one should identify the pixels belonging to each lane. Two methods are used for this purpose. The first is based on histogram methods is can be used to do a cold-start, without prior knowledge of the lane position. The second takes less time to compute but requires the position of the lane line from the previous frame. It is therefore used to update the lane position at later iterations.

### Histogram fitting

As a first step, the start of the lanes at the bottom is detected. For this purpose, the image is split in four quadrants, of which only the two lower ones are used. In those quadrants, the lane pixels are summed per column of the image. By taking the maximum of this sum, the most likely initial position is computed.

![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/hist.PNG?raw=true)

This is then taken as a start position for a first window. All pixels inside this window are considered as belonging to the corresponding lane line. Windows are stacked one by one and their center position is adapted at each step, in order to match the average position of the detected pixels. In the end, all the pixels belonging to a given lane are fitted using a RANSAC fitting method and considering a second order polynomial.

![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/histfit.PNG?raw=true)

### Fitting update

This method defines a band centered on the fitted polynomials of the previous frame. All pixels included in this band are considered as part of the lane. A new polynomial can be fitted to them, as shown in the following image. 

![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/fitupdate.PNG?raw=true)

The pixels used for the fitting are computed for each lane and are used to determine the quality of the fit. It this value is too low, the algorithm falls back to the histogram fitting by default.

## Output lane update

Those raw polynomials are not used as the output of the algorithm, since they tend to be quite noisy. Instead, they are first assessed in terms of parallelism and curvature. Outliers are thrown away and the output of the previous slide is used instead. If a new correct output is computed, it is finally averaged over the last previous frames in order to reduce the jitter.

## Position and curvature computation

Assuming that the vehicle is located at the center of the image, the position of the lanes at the bottom of the image can be used in order to assess the position of the vehicle in the lane. Similarly, the higher order coefficients of the polynomials can be used to assess the curvature of the road, which is a very useful information for the control algorithm. Finally, one should not forget to convert those values from pixels to meters.

## Information plotting

Firstly, the two polynomials corresponding to the present frame are used to plot the area corresponding to the lane in which the car drives. The inverse perspective transformation can then be used to project this image back into the camera point of view. 

![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/roadzone.PNG?raw=true)

This image can then be overlayed on top of the input image. The information on the curvature and the position of the vehicle are also plotted on top of this image. Finally, some debug information was added, which includes the thresholded image and the output of the fitting algorithm. This allows the quick visualization of the important steps in the algorithm and to assess the situation if something goes wrong.

![enter image description here](https://github.com/thdhondt/Advanced-lane-finding/blob/master/examples/output.PNG?raw=true)

## Video results

The output of the algorithm is available in the videos "white.mp4" and "white2.mp4", which corresponds to the project video and the challenge video respectively. The algorithm performs very well on both and is able to filter out the noise, detect lanes in all lighting conditions.

The third video still requires some work, as the light variations in a single frame are very strong. The curvature of the road is also extremely high, which might require an alternative to the initialization by histogram. The smoothing over ten frames should also be revisited in order to allow for faster variations of the output.
