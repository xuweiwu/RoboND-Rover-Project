## Project: Search and Sample Return
### Writeup Template: You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---


**The goals / steps of this project are the following:**  

**Training / Calibration**  

* Download the simulator and take data in "Training Mode"
* Test out the functions in the Jupyter Notebook provided
* Add functions to detect obstacles and samples of interest (golden rocks)
* Fill in the `process_image()` function with the appropriate image processing steps (perspective transform, color threshold etc.) to get from raw images to a map.  The `output_image` you create in this s
* Use `moviepy` to process the images in your saved dataset with the `process_image()` function.  Include the video you produce as part of your submission.

**Autonomous Navigation / Mapping**

* Fill in the `perception_step()` function within the `perception.py` script with the appropriate image processing functions to create a map and update `Rover()` data (similar to what you did with `process_image()` in the notebook). 
* Fill in the `decision_step()` function within the `decision.py` script with conditional statements that take into consideration the outputs of the `perception_step()` in deciding how to issue throttle, brake and steering commands. 
* Iterate on your perception and decision function until your rover does a reasonable (n

[image1]: ./calibration_images/example_rock1.jpg
[image2]: ./output/warped_threshed_provided_data.jpg
[image3]: ./calibration_images/example_rock_recorded.jpg
[image4]: ./output/warped_threshed_recorded_data.jpg
[image5]: ./warped_threshed_rock.jpg 

## [Rubric](https://review.udacity.com/#!/rubrics/916/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it!

### Notebook Analysis
#### 1. Run the functions provided in the notebook on test images (first with the test data provided, next on data you have recorded). Add/modify functions to allow for color selection of obstacles and rock samples.
The color selection of obstacles is achieved by inverting the binary image of the navigable terrain `threshed_navigable`:
```python
threshed_obstacle = np.ones_like(threshed_navigable) - threshed_navigable
```
Here `threshed_obstacle` is the obtained binary image of obstacles.

For color selection of rock samples a new function `color_thresh_rock()` is added:
```python
def color_thresh_rock(img, rgb_thresh=([130, 210], [110, 190], [0, 80])):
    # Create an empty array the same size in x and y as the image 
    # but just a single channel
    color_select_rock = np.zeros_like(img[:,:,0])
    # Apply the ranges for RGB and assign 1's 
    # where RGB was in the range
    # Return the single-channel binary image
    in_range = (img[:,:,0] >= rgb_thresh[0][0]) & (img[:,:,0] <= rgb_thresh[0][1]) \
               & (img[:,:,1] >= rgb_thresh[1][0]) & (img[:,:,1] <= rgb_thresh[1][1]) \
               & (img[:,:,2] >= rgb_thresh[2][0]) & (img[:,:,2] <= rgb_thresh[2][1])
    color_select_rock[in_range] = 1
    return color_select_rock
```
The function sets a value range for each channel of RGB individually to specify the yellow/gold color of rock samples. Those ranges are obtained through investigations of self-recorded example images in an interactive matplotlib window.

The following two figures show the resulsts of the color selection with provided and recorded images, respectively.
Provided image:
![alt text][image1]
Color selection with the provided image:
![alt text][image2]
Recorded image:
![alt text][image3]
Color selection with the recorded image:
![alt text][image4]

#### 2. Populate the `process_image()` function with the appropriate analysis steps to map pixels identifying navigable terrain, obstacles and rock samples into a worldmap.  Run `process_image()` on your test data using the `moviepy` functions provided to create video output of your result. 
The main steps in `process_image()` include:
* The perspective transformation from rover camera's view into top-down view. First, the source and destination points are determined. After that, the function `perspect_transform()` performs the transformation by invoking the OpenCV functions `cv2.getPerspectiveTransform()` and `cv2.warpPerspective()`.

* The color selection for navigable terrain/obstacles/rock samples, implemented as explained above.

* The conversion of pixel positions into rover-centric coordinates. The function `rover_coords()` is applied to perform the conversion, which takes the binary images of navigable terrain/obstacles/rock samples as input and returns the corresponding rover-centric coordinates.

* The trnsformation of rover-centric coordinates into world coordinates. The coordinate transformation is further divided into a rotation and a following translation. The function `rotate_pix()` performs the 2D rotation by using the values of the rover's yaw angle. The function `translate_pix()` performs the translation by adding the values of the rover's position in world space to the rotated coordinates. These two functions are called internally in the function `pix_to_world()` to accomplish the whole transformation step.

* The generation of a worldmap. The woldmap, saved as `data.worldmap`, is initialized as a color image with 200 x 200 pixels (each pixel represents 1 x 1 m) and all its RGB channels are filled with zeros, i.e. the worldmap is completely black at the beginning. As the rover explores the environment, the values of channel R, G, B accumulate at the pixels (described with world coordinates) of obstacles, rock samples, and navigable terrain, respectively. As a result, the navigable terrain will be marked with blue color and the obstacles with red color. Since rock samples are also seen as a kind of obstacles, the positions of the rock samples in the map will be marked by red and green at the same time and the combined color is yellow.

### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (at the bottom of the `perception.py` script) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts and an explanation is provided in the writeup of how and why these functions were modified as they were.
Modifications in `perception_step()`:

* The most part of the modifications in `perception_step()` are similar to those in `process_image()` in the notebook, including the perspective transformation, the color selection, and the coordinate conversion and transformation. 

* An additional vision image, saved as `Rover.vision_image`, is updated to show the instantaneous pixel positions of navigable terrain/obstacles/rock samples in the rover camera's view. After each update the vision image is overlaid with the black area of a warped white image to show the actual vision area. 

* To enhence the map fidelity, `Rover.worldmap` will only be updated if the roll and pitch angles are smaller than 0.5 degree, since the performed coordinate transformation is only valid for 2D situations and dosen't take the roll and pitch angles into account.

* The rover-centric polar coordinates of the navigable terrain pixels `(Rover.nav_dists, Rover.nav_angles)` and rock sample pixels `(Rover.rock_dists, Rover.rock_angles)` are calculated by using the function `to_polar_coords()`. These additional variables are to be utilized in `decision_step()` to steer the rover.

Modifications in `decision_step()`:
* The rover has two basic driving modes: `forward` and `stop`. The conversion between the modes are determined by the extent of the navigable terrain. Two thresholds, `Rover.go_forward` and `Rover.go_forward`, are set for the mode conversion.

* If the navigable terrain is sufficient (`len(Rover.nav_angles) >= Rover.stop_forward`) and no rock sampels are detected (np.size(Rover.rock_angles) == 0), the rover will keep going forward with its velocity limited to a preset maximum value. The steering angel is set to the average angle of the navigable terrain polar coordinates.

* If during the forward mode any pixels of rock samples are detected (`np.size(Rover.rock_dists) > 0`), then the rover will be turned towards the position of the nearest sample. As the rover approaches the sample, its velocity is limited, first a


#### 2. Launching in autonomous mode your rover can navigate and map autonomously.  Explain your results and how you might improve them in your writeup.  

**Note: running the simulator with different choices of resolution and graphics quality may produce different results, particularly on different machines!  Make a note of your simulator settings (resolution and graphics quality set on launch) and frames per second (FPS output to terminal by `drive_rover.py`) in your writeup when you submit the project so your reviewer can reproduce your results.**

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  



![alt text][image3]


