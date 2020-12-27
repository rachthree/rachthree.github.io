---
layout: post
title: "Automating Labels with Simulators"
tags:
- AirSim
- NeurIPS
- Data Engineering
- Labelling
thumbnail_path: "blog/airsim-flyable-regions/processed_labels.png"
---

Towards the end of 2019, Microsoft hosted a drone competition at NeurIPS called [Game of Drones](https://www.microsoft.com/en-us/research/academic-program/game-of-drones-competition-at-neurips-2019/), aiming to advance autonomous systems through drone races. To focus on algorithms, it used [AirSim](https://github.com/microsoft/AirSim), an open source Unreal-based simulation platform for robots, supporting cars and drones to aid in autonomous systems research.
My team was unable to compete due to life obligations. However, I continued after the competition ended. I believe as graphics get more photorealistic, simulators may be used to get the large amount of data needed to train models.

Following the lessons learned from being a perception lead in Lockheed Martin's AlphaPilot competition, my approach was to utilize both traditional computer vision and deep learning.
Masks of objects and flyable regions can be found using convolutional neural networks, and additional simple computer vision algorithms can be used to further postprocess.
For AlphaPilot, this required hand labelling of all training images. Although labelling is an essential part for training neural networks, it can be a laborious process with risk of error and low fidelity.

With AirSim, you can get the pixel labelling of object IDs from a camera's point of view, allowing for segmentation. With being able to place a drone/camera anywhere, there is limitless potential to get the dataset you need in less time and with less resources.
For instance, object positioning, weather, environments, and time of day can be controlled. Below is an example scenery.

{% include figure.html path="blog/airsim-flyable-regions/drone_view.png" alt="Drone View" %}

The scene would be what the drone sees. The lower three views, from left to right, are segmentation, IR (not really functional), and depth. After some initial relabelling, this is the segmentation view:

{% include figure.html path="blog/airsim-flyable-regions/unprocessed_labels.png" alt="Unprocessed Labels" %}

Using a purely traditional computer vision method to identify gates and their flyable regions is difficult because each frame would need its own parameters. If a CNN can get flyable regions as well, then it would be a more robust method.
The easiest way would be to edit the gate assets to have invisible regions that can be ID'd with the segmentation camera. However, the training and evaluation binaries were not accessible by Unreal. So, how can this be solved?

### Additional Labelling using a Competition Simulator 
Thankfully, the training binary had access to the following information via the AirSim client:
* Drone location and orientation
* Drone camera information
* Gate scale
* Gate locations and orientation
* Object IDs (all in the level and those in view via the segmentation camera)
* Depth image from the depth perspective camera

This information allows us to do the following steps to get the flyable regions:
1. Grab all gate IDs that are in the segmentation image and get their position and orientation in global coordinates.
2. We define the flyable region as the plane at the front face of the gate. The four inner corners can be computed using gate scale, position, and orientation.
3. Convert the four corners from global coordinates to the drone/camera's coordinate system.
4. Project the points onto the camera's perspective using its projection matrix
5. Fill in the polygon formed by those points and account for occlusion.

The projected points are essentially indices in an image array, and OpenCV can be used to fill in the polygon to provide a flyable region mask.
However, some points may actually be beyond the array's dimensions, i.e the flyable region extends outside of the camera view. To safely account for all possible cases, the image can be padded to fit the points that are outside the array.
When the polygon is filled in, the image can be cropped back to its original size.

To account for occlusion, we can use the depth measurements. Anything with a distance measurement less than the distances of the four points to the camera is kept. 

### Final Label Result
Now we have the final processed labels. Now the flyable regions have been labeled. Note that the colors representing the IDs of the gates changed so that the numbering of the gates always start with 1.

{% include figure.html path="blog/airsim-flyable-regions/processed_labels.png" alt="Processed Labels" %}

Above, it can be seen that the third gate's flyable region does not bleed through the second gate. Occlusion has been taken into account.
Even with the limitations of not being able to edit the assets, we can still find ways to get the dataset we need.
