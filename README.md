# EventRec_toolkit
This repositories contain various funcionalities:
1. How to do an intrinsic and extrinsic calibration of a camera system consisting of two event cameras and one RGB camera.
2. Eye-To-Hand calibration of the camera system with a vicon camera. This is esential in order to track objects using vicon camera. Subsequently transforming their positions to the camera system. Thereby, enabling one to get the ground truth in all camera frames. 
3. Computing translation and angular velocity of objects being tracked by Vicon.
4. Anootaion scripts

## Requirments
[e2calib package](https://github.com/uzh-rpg/e2calib) needs to be installed first to be able to
reconstruct the events to grey scale images. The scripts "convert.py" and "offline_reconstruction.py" will be used.
Install the package in directory:

     cd ~/event_camera/

Install [Kalibr toolbox](https://github.com/ethz-asl/kalibr). Create a workspace "~/kalibr_workspace"
Source its workspace:

     source ~/kalibr_workspace/devel/setup.bash

If using ids camera, then instal ids camera driver as mentioned in:

https://github.com/FLW-TUDO/ids_camera_driver

## event camera calibration

Launch one RGB and two event cameras with ROS from a launch file in this package:

     roslaunch RGB_event_cam_stereo.launch.launch

Record a ROS bag file by keeping the cameras stationtry and moving a checkerboard 
in front of the cameras. Make sure to cover the entire frame.

     mkdir ~/event_camera/calibration_data/events_only_calibration
     cd ~/event_camera/calibration_data/events_only_calibration
     rosbag record /dvxplorer_left/events /dvxplorer_right/events /rgb/image_raw --output-name events_only.bag

Then run the "convert.py" and finally the "offline_reconstruction.py" scripts from the e2calib package.

     cd ~/event_camera/e2calib
     python /home/eventcamera/e2calib/python/convert.py --output_file /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/events_left.h5 --topic '/dvxplorer_left/events' 
     --input_file /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/events_only.bag 

     python /home/eventcamera/e2calib/python/convert.py --output_file /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/events_right.h5 --topic '/dvxplorer_right/events' 
     --input_file /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/events_only.bag 

     python /home/eventcamera/e2calib/python/offline_reconstruction.py --h5file /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/events_right.h5 --freq_hz 90 
     --output_folder /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/reconstructed_event_images --use_gpu
     mv /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/reconstructed_event_images/e2calib/ /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/reconstructed_event_images/cam2 
 
     python /home/eventcamera/e2calib/python/offline_reconstruction.py --h5file /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/events_left.h5 --freq_hz 90 
     --output_folder /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/reconstructed_event_images --use_gpu
     mv /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/reconstructed_event_images/e2calib/ /home/eventcamera/Eventcamera/calibration_data/RGB_stereo_event/reconstructed_event_images/cam1 

Extract RGB image from bag file by executing the script in folder RGB_Event_cam_system/calibration/calibration_rgb_event_cameras_extrinsics/
     
     python RGB_Event_cam_system/calibration/calibration_rgb_event_cameras_extrinsics/extract_rgb_img_from_bag.py

The RGB images will be extracted to folder cam0

Source kalibr workspace:
     source /home/eventcamera/kalibr_ws/devel/setup.bash
Go to folder cd RGB_stereo_event

The calibration can be performed using kalibr. Kalibr reads the images from a ROS bag.
So the reconstructed event images have to converted to a ROS bag using this command:
     
     rosrun kalibr kalibr_bagcreater --folder reconstructed_event_images/ --output-bag images.bag
     
     rosrun kalibr kalibr_calibrate_cameras \
      --target ../checkerboard.yaml \
      --models pinhole-radtan pinhole-radtan pinhole-radtan \
      --topics /cam0/image_raw /cam1/image_raw /cam2/image_raw \
      --bag images.bag \
      --bag-freq 100.0 \
      --verbose
NOTE: For our case the model for dvxplorer event camera is omni radtan. But this can defer based on the lens that is used with camera.

## Common Errors

If you get an error sayng Optimization failed the please increase the timeOffsetPadding in line 97 of kalibr_calibrate_imu_camera.py file. timeOffsetPadding is the margin within which the temporal offset is allowed to vary. Varying this parameter is a trade-off between computation time and robustness. If the margin is large, it will result in increased computation time, if it is too low, the estimate might travel across knot boundaries in an unpredicted manner, violating the precomputed sparsity pattern. Temporal offset during optimization depend on trajectory of the motion of the checkerboard pattern during calibration.
