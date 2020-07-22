# Creating a New Device

In this tutorial, we will add a device to the Intprim ROS framework. In order to add a device, you must have a device driver (as mentioned below).

## 1.1 Add Device

In this tutorial, we will add a robot to the Intprim ros framework named Robot1. Robot1 can represent any type of robot (Kuka, Universal Robotics, Willow Garage, etc.) but the finer details will depend on your specific robot and its driver, which are a seperate from the interface. Feel free to replace the keyword Robot1 with the name of the robot you are using.

The devices have the following inheritance structure:
- *DeviceInterface -> RobotInterface -> Robot1*
- *DeviceState -> Robot1State*

To add a new robot, we will inherit from the RobotInterface class as well as the DeviceState Class.

Create a new file under `intprim_framework_ros_dev/include/inprim_framework_ros/devices` named `Robot1.h`. In this file, add a class that inherits from DeviceState:

```c++
#pragma once
#include "device_interface.h"
#include "robot_interface.h"
#include <vector>
#include "ros/ros.h"
#include <your_robots/robot1Control.h>
#include <your_robots/robot1Joints.h>

class Robot1State : public DeviceState
{
    public:
        static constexpr unsigned int           NUM_JOINTS = 7;
        static double                           JOINT_THRESHOLD;

        Robot1State();

        void to_matrix(std::vector<float>& trajectory) const;

        bool valid() const;

        void set_message(const your_robots::robot1Joints::ConstPtr& message);

        bool within_threshold(const std::vector<float>& trajectory, std::size_t trajectory_idx) const;

    private:
        your_robots::robot1Joints::ConstPtr         m_message;
        bool                                        m_valid;
};
```

In this implementation, two ros messages are used for communication between your robot and the Intprim ros framework:
- The first type, `your_robots::robot1Joints`, is published from the robot driver to the framework. This message contains the current state of Robot (Joint Positions and Velocities). 
- The second type, `your_robots::Robot1Control`, is published from the framework to the robot driver. This message contains the inference output generated by the framework as well as other useful data.

Notice the namespace `your_robots::`. This should be the namespace of your robot driver(s), and it is where the .msg files for the two types of ros messages are located. It is not necessary to use custom messages from the robot driver's namespace; you may use native ros messages instead. Additionally, you may use the same message for both control and joint state input.

In order to support the Intprim ros implementation, your robot driver will need to subscribe to the ros messages (of type `your_robots::robot1Control`) published by the framework. The driver should then use the information inside of these messages to command the robot. Additionally, your robot driver must publish the robot's current state as a rostopic for the framework to use. The following object from the class above is a message from the robot device driver publisher:

`your_robots::robot1Joints::ConstPtr         m_message;`

This message object may contain, for example, an array of positons and an array of velocities. The topic name that joint states are published on can be found inside the constructor for the Robot Interface (i.e '/robot1/joints'). The subscriber for this topic and its callback are specified in the Robot Interface (which contains an instance of the Robot1State class above). The callback sets the data inside of the `m_message` object of the Robot1State instance to the data inside the incoming message/parameter.

Next, we will create the Robot1Interface class by inheriting from RobotInterface. Add this class to the file above, `Robot1.h`:

```c++
class Robot1Interface : public RobotInterface
{
    public:
        Robot1Interface(ros::NodeHandle handle);

        // Covariant return type.
        const Robot1State& get_state();

        void publish_state(const std::vector<float>& trajectory, std::size_t trajectory_idx);
    private:
        static double                               CONTROL_TIME_BUFFER;
        static int                                  CONTROL_FREQUENCY;
        static double                               MAX_ACCELERATION;

        ros::Subscriber                             m_stateSubscriber;
        ros::Publisher                              m_statePublisher;

        Robot1State                                 m_currentState;
        your_robots::Robot1Control                  m_publishMessage;

        unsigned int                                m_control_frequency;
        float                                       m_messageTime;

        void state_callback(const your_robots::robot1Joints::ConstPtr& message);
};
```

As mentioned, the Robot1Interface will contain an instance of the Robot1State class, `m_currentState`. You can also see the publisher and subscriber, `m_stateSubscriber` and `m_statePublisher`. The `ros::NodeHandle` passed into the constructor allows the Interface to easily communicate with ROS. After creating the header file for your robot, you will need to fill in the function definitions. Make an new file named `Robot1.cpp`:

```c++
#include "devices/Robot1.h"
#include <boost/bind.hpp>
#include <chrono>
#include <regex>
#include <sstream>
#include <string>
#include <thread>

double Robot1State::JOINT_THRESHOLD = 0.0;
double Robot1Interface::CONTROL_TIME_BUFFER = 0.0;
int    Robot1Interface::CONTROL_FREQUENCY = 0;
double Robot1Interface::MAX_ACCELERATION = 0.0;

Robot1State::Robot1State() :
    m_message(),
    m_valid(false)
{

}

void Robot1State::to_matrix(std::vector<float>& trajectory) const
{
    if(m_message)
    {
        for(std::size_t joint_idx = 0; joint_idx < NUM_JOINTS; ++joint_idx)
        {
            trajectory.push_back(m_message->positions[joint_idx]);
        }
    }
    else
    {
        for(std::size_t joint_idx = 0; joint_idx < NUM_JOINTS; ++joint_idx)
        {
            trajectory.push_back(0.0);
        }
    }
}

const your_robots::robot1Joints::ConstPtr& Robot1State::get_message()
{
    return m_message;
}

void Robot1State::set_message(const your_robots::robot1Joints::ConstPtr& message)
{
    m_message = message;
    m_valid = true;
}

bool Robot1State::valid() const
{
    return m_valid;
}

bool Robot1State::within_threshold(const std::vector<float>& trajectory, std::size_t trajectory_idx) const
{
    if(m_message)
    {
        for(std::size_t joint_idx = 0; joint_idx < NUM_JOINTS; ++joint_idx)
        {
            if(std::abs(m_message->positions[joint_idx] - trajectory[joint_idx + trajectory_idx]) > JOINT_THRESHOLD)
            {
                return false;
            }
        }
        return true;
    }
    return false;
}
```

The incoming `trajectory` parameter for the to_matrix() function sets the to the positions in the most recent robot state. You may have to modify this depending on the data contained in your ros message for your robot. For example, here the positions attribute of the message contains the state information about the robot that will be used for inference. If you are using a different variable than position, you will have to specify that in this function.

In the following code block, we will write out the definitions for the Robot Interface:


```c++
Robot1Interface::Robot1Interface(ros::NodeHandle handle) :
    m_stateSubscriber(),
    m_statePublisher(),
    m_currentState(),
    m_publishMessage()
{
    handle.getParam("control/robot1/control_time_buffer", CONTROL_TIME_BUFFER);
    handle.getParam("control/robot1/control_frequency", CONTROL_FREQUENCY);
    handle.getParam("control/robot1/max_acceleration", MAX_ACCELERATION);
    handle.getParam("control/robot1/joint_distance_threshold", Robot1State::JOINT_THRESHOLD);

    std::cout << "control time buffer: " << CONTROL_TIME_BUFFER << ", control frequency: " << CONTROL_FREQUENCY << ", max accel: " << MAX_ACCELERATION << ", joint thresh: " << Robot1State::JOINT_THRESHOLD << std::endl;

    m_messageTime = (1.0 / CONTROL_FREQUENCY) * CONTROL_TIME_BUFFER;

    m_publishMessage.acceleration = MAX_ACCELERATION;
    m_publishMessage.blend = 0;
    m_publishMessage.command = "speed";
    m_publishMessage.gain = 0;
    m_publishMessage.jointcontrol = true;
    m_publishMessage.lookahead = 0;
    m_publishMessage.time = m_messageTime;
    m_publishMessage.velocity = 0;

    m_statePublisher = handle.advertise<your_robots::robot1Control>("/robot1/control", 1);
    m_stateSubscriber = handle.subscribe("/robot1/joints", 1, &Robot1Interface::state_callback, this);
}

const Robot1State& Robot1Interface::get_state()
{
    return m_currentState;
}

void Robot1Interface::publish_state(const std::vector<float>& trajectory, std::size_t trajectory_idx)
{
    for(std::size_t joint_idx = 0; joint_idx < Robot1State::NUM_JOINTS; ++joint_idx)
    {
        m_publishMessage.values[joint_idx] = trajectory[trajectory_idx + joint_idx];
    }

    m_statePublisher.publish(m_publishMessage);
}

void Robot1Interface::state_callback(const your_robots::robot1Joints::ConstPtr& message)
{
    m_currentState.set_message(message);
}
```

In the constructor, the Robot Interface reads the parameters from the ros server. These parameters are specified in the .yaml files that are discussed in the Introduction Section. To set up the desired frequency and thresholds, the following lines are used:

`handle.getParam("control/robot1/control_time_buffer", CONTROL_TIME_BUFFER);
handle.getParam("control/robot1/control_frequency", CONTROL_FREQUENCY);
handle.getParam("control/robot1/max_acceleration", MAX_ACCELERATION);
handle.getParam("control/robot1/joint_distance_threshold", Robot1State::JOINT_THRESHOLD);`

The following lines initialize the variables of the m_publishMessage, which is the message that will be used for commanding the robot. As you can see, the parameters from the ros server are used to set variables in this message:

`m_publishMessage.acceleration = MAX_ACCELERATION;
m_publishMessage.blend = 0;
m_publishMessage.command = "speed";
m_publishMessage.gain = 0;
m_publishMessage.jointcontrol = true;
m_publishMessage.lookahead = 0;
m_publishMessage.time = m_messageTime;
m_publishMessage.velocity = 0;`


The `respond()` thread that runs inside of Interaction Core calls `publish_state()`, which fills the values of the `m_publishMessage` object. The parameters of `publish_state()`, `trajectory` and `trajectory_size` are effectively the inference output controls from Intprim. As you can see, the Robot Interface is primarily responsible for commanding the robot. Through communicating with the DeviceState, the Robot Interface can maintain proper synchronization and control of the robot. Note that many of the methods above are related to the synchronization and frequency settings that are desired by the user.


## 1.2 Adding Robot to Factory Pattern Design
The framework has the following general structure: devices are used to publish data to the framework, and robots (which are a type of device) are primarily used for sending control output from the framework. The objcetive of using the Factory Pattern Design for Device Interfaces is for the framework to be as extensible and user-friendly as possible. The framework enumerates the device interfaces based on the strings read from the parameter file. For this reason, we will have to add an enum for `robot1`, which we have just created above. 

In the `interaction_application.h` file, add the `robot1` enum:

```c++
enum class DeviceInterfaceTypes
{
    ur5,
    robot1
};
```

This enum uniquely identifies the Robot Interface for `Robot1` (which inherits from DeviceInterface). In order to  create these devices at runtime, this information is needed in the `create_device()` method inside of `interaction_application.cpp`. If `robot1` is specified as a device in the parameter files, then when looping through these parameter files, intprim will create the correct robot device for your experiment:


```c++
std::unique_ptr<DeviceInterface> InteractionApplication::create_device(DeviceInterfaceTypes interface_type)
{
    std::unique_ptr<DeviceInterface> device_interface;

    switch(interface_type)
    {
        case InteractionApplication::DeviceInterfaceTypes::robot1:
            device_interface = create_robot(interface_type);
            break;

        case InteractionApplication::DeviceInterfaceTypes::ur5:
            device_interface = create_robot(interface_type);
            break;
    }

    return device_interface;
}
```

Likewise, If `robot1` is specified as a controller in the parameter files, then when looping through these parameter files, intprim will create the correct robot controller for your experiment. In order for this to happen, you must add code to the `create_robot()` method in `interaction_application.cpp`:

```c++
std::unique_ptr<RobotInterface> InteractionApplication::create_robot(DeviceInterfaceTypes interface_type)
{
    std::unique_ptr<RobotInterface> robot_interface;

    switch(interface_type)
    {
       case InteractionApplication::DeviceInterfaceTypes::robot1:
            #ifdef YOUR_ROBOTS_AVAILABLE
                robot_interface = std::unique_ptr<Robot1Interface>(new Robot1Interface(*m_handle));
            #else
                throw std::runtime_error("Experiment uses Robot1Interface but your robots unavailable.");
            #endif
            break;

        case InteractionApplication::DeviceInterfaceTypes::ur5:
            #ifdef IRL_ROBOTS_AVAILABLE
                robot_interface = std::unique_ptr<UR5Interface>(new UR5Interface(*m_handle));
            #else
                throw std::runtime_error("Experiment uses UR5Interface but IRL robots unavailable.");
            #endif
            break;
    }

    return robot_interface;
}
```

## 1.3 Adding the robot to CMakeLists.txt

The following changes will need to be added to `CMakeLists.txt` so that your new Robot Interface can be found and used. Additionally, you will have to make sure that your new Robot Interface, which depends on having a device driver in the `your_robots` namespace, is able to be found by CMake:

```c++
if(your_robots_FOUND)
    list(APPEND CORE_SRC
        src/devices/Robot1.cpp
    )

    list(APPEND CONTROLLER_SRC
        src/devices/Robot1.cpp
    )
endif()

add_executable(
    interaction_application
    ${CORE_SRC}
)

add_executable(
    robot_controller
    ${CONTROLLER_SRC}
)

if(your_robots_FOUND)
    target_compile_definitions(interaction_application PRIVATE YOUR_ROBOTS_AVAILABLE)
    target_compile_definitions(robot_controller PRIVATE YOUR_ROBOTS_AVAILABLE)
endif()
```