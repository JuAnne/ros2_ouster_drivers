# ROS2 Ouster Drivers

These are an implementation of ROS2 drivers for the Ouster OS-1 3D lidars. This includes all models of the OS-1 from 16 to 128 beams. 

You can find a few videos looking over the sensor below. They both introduce the ROS1 driver but are extremely useful references regardless:

OS-1 Networking Setup      |  OS-1 Data Overview
:-------------------------:|:-------------------------:
[![IMAGE ALT TEXT HERE](http://img.youtube.com/vi/92ajXjIxDGM/0.jpg)](http://www.youtube.com/watch?v=92ajXjIxDGM) | [![IMAGE ALT TEXT HERE](http://img.youtube.com/vi/4VgGG8Xe4IA/0.jpg)](http://www.youtube.com/watch?v=4VgGG8Xe4IA)


I also happen to run a YouTube channel, [Robots for Robots](https://www.youtube.com/channel/UCZT16dToD1ov6lnoEcPL6rw), focusing on robotics, sensors, and industry insights. If you like this work and want to hear about some other things I am working on, I'd appreciate if you watch, like, and subscribe! 

## Documentation

Documentation can be generated using Doxygen. 

Run `doxygen` in the root of this repository. It will generate a `/doc/*` directory containing the documentation. Entrypoint in a browser is `index.html`.

## Design

See design doc in `design/*` directory [here](ros2_ouster/design/design_doc.md).

## ROS Interfaces


| Topic             | Type                    |
|-------------------|-------------------------|
| `range_image`     | sensor_msgs/Image       |
| `intensity_image` | sensor_msgs/Image       |
| `noise_image`     | sensor_msgs/Image       |
| `points`          | sensor_msgs/PointCloud2 |
| `imu`             | sensor_msgs/Imu         |

| Service           | Type                    |
|-------------------|-------------------------|
| `reset`           | std_srvs/Empty          |
| `GetMetadata`     | ouster_msgs/GetMetadata |

| Parameter         | Type                    | Description                                         |
|-------------------|-------------------------|-----------------------------------------------------|
| `lidar_ip`        | String                  | IP of lidar (ex. 10.5.5.87)                         |
| `computer_ip`     | String                  | IP of computer to get data (ex. 10.5.5.1)           |
| `lidar_mode`      | String                  | Mode of data capture, default `512x10`              |
| `imu_port`        | int                     | Port of IMU data, default 7503                      |
| `lidar_port`      | int                     | Port of laser data, default 7502                    |
| `sensor_frame`    | String                  | TF frame of sensor, default `laser_sensor_frame`    |
| `laser_frame`     | String                  | TF frame of laser data, default `laser_data_frame`  |
| `imu_frame`       | String                  | TF frame of imu data, default `imu_data_frame`      |

Note: TF will provide you the transformations from the sensor frame to each of the data frames.

## Extensions

This package was intentionally designed for new capabilities to be added. Whether that being supporting new classes of Ouster lidars (OS1-custom, OS2, ...) or supporting new ways of processing the data packets.

### Additional Lidar Processing
It can be imagined that if you have a stream of lidar or IMU packets, you may want to process them differently. If you're working with a high speed vehicle, you may want the packets projected into a pointcloud and published with little batching inside the driver. If you're working with pointclouds for machine learning, you may only want the pointcloud to include the `XYZ` information and not the intensity, reflectivity, and noise information to reduce dimensionality. 

In any case, I provide a set of logical default processing implementations on the lidar and IMU packets. These are implementations of the `ros2_ouster::DataProcessorInterface` class in the `interfaces` directory. To create your own processor to change the pointcloud type, buffering methodology, or some new cool thing, you must create an implementation of a data processor.

After creating your implementation, that will take in a `uint8_t *` of a data packet and accomplish your task, you will need to create a factory method for it in `processor_factories.hpp` and add it to the list of processors to be created in the `createProcessors` method.

I encourage you to contribute back any new processor methods to this project! The default processors will buffer 1 full rotation of data of the pointcloud and publish the pointcloud with the X, Y, Z, range, intensity, reflectivity, ring, and noise information. It will also buffer a full rotation and publish the noise, intensity, and reflectivity images. Finally, it will publish the IMU data at transmission frequency.

Some examples:
- If you wanted the points at transmission frequency to reduce aliasing
- Different types of pointclouds published containing a subset or additional information.
- If you wanted the information in another format (ei 1 data image with 3 channels of the range, intensity, and noise)
- Downsample the data at a driver level to only take every `N`th ring.

### Additional Lidar Units
To create a new lidar for this driver, you only need to make an implementation of the `ros2_ouster::SensorInterface` class and include any required SDKs. Then, in the `driver_types.hpp` file, add your new interface as a template of the `OusterDriver` and you're good to go.

You may need to add an additional `main` method for the new templated program, depending if you're using components. If it uses another underlying SDK other than `OS1` you will also need to create new processors for it as the processors are bound to a specific unit as the data formatting may be different. If they are the same, you can reuse the `OS1` processors. 

## Lifecycle

This ROS2 driver makes use of Lifecycle nodes. If you're not familiar with lifecycle, or managed nodes, please familiarize yourself with the [ROS2 design](https://design.ros2.org/articles/node_lifecycle.html) document on it.

The lifecycle node allow for the driver to have a deterministic setup and tear down and is a new feature in ROS2. The launch script will need to use a lifecycle node launcher to transition into the active state to start processing data.

## Component

This ROS2 driver makes use of Component nodes. If you're not familiar with component nodes please familiarize yourself with the [ROS2 design](https://index.ros.org/doc/ros2/Tutorials/Composition) document on it.

The component node allow for the driver and its processing nodes to be launched into the same process and is a new feature in ROS2. This allows the sensor and its data clients to operate without serialization or copying between nodes sharing a memory pool.

There's a little work in ROS2 Eloquent to launch a component-lifecycle node using only the roslaunch API. It may be necessary to include the Ouster driver in your lifecycle manager to transition into the active state when loading the driver into a process as a component.

# Ouster Messages

A message `Metadata` was created to describe the metadata of the lidars. In addition the `GetMetadata` service type will be used to get the metadata from a running driver.
