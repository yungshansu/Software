# Package `easy_node` {#easy_node}

`easy_node` is a framework to make it easier to create and document ROS nodes.

The main idea is to provide a *declarative approach* to describe:

- The node parameters;
- The node's subscriptions;
- The node's publishers;
- The node's assumptions (contracts).

The user describes subscriptions, publishers, and parameters in a YAML file.

The framework then automatically takes care of:

- Calling the necessary boilerplate ROS commands for subscribing and publishing topics.
- Loading and monitoring configuration.
- Create the Markdown documentation that describes the nodes.
- Provide a set of common functionality, such as benchmarking and monitoring latencies.

Using `easy_node` allows to cut 40%-50% of the code required for programming
a node. For an example, see the package [`line_detector2`](#line_detector2),
which contains a re-implementation of  `line_detector` using the new framework.

**Transition plan**: The plan is to first use `easy_node` just for
documenting the nodes. Then, later, convert all the nodes to use it.

## YAML file format

For a node with the name `![my node]`, implemented in the file
`src/![my node].py` you must create a file by
 the name `![my node].easy_node.yaml`somewhere in the package.

The YAML file must contain 4 sections, each of which is a dictionary.

This is the smallest example of an empty configuration:

    parameters: {}
    subscriptions: {}
    publishers: {}
    contracts: {}

### `parameters` section: configuring parameters

This is the syntax:

    parameters:
        ![name parameter]:
            type: ![type]
            desc: ![description]
            default: ![default value]

where:

- `![type]` is one of `float`, `int`, `bool`, `str`.
- `![description]` is a description that will appear in the documentation.
- The optional field `default` gives a default value for the parameter.

For example:

    parameters:
        k_d:
            type: float
            desc: The derivative gain.
            default: 1.02

### `publishers` and `subscriptions` section

The syntax for describing subscribers is:

    subscriptions:
        ![name subscription]:
            topic: ![topic name]
            type: ![message type]
            desc: ![description]

            queue_size: ![queue size]
            latch: ![latch]
            process: ![process]

where:

- `![topic name]` is the name of the topic to subscribe.
- `![message type]` is a ROS message type name, such as `sensor_msgs/Joy`.
- `![description]` is a Markdown description string.
- `![queue size]`, `![latch]` are optional parameters for
  ROS publishing/subscribing functions.
- The optional parameter `![process]`, one of `synchronous` (default) or `asynchronous` describes whether to process the message in a synchronous or asynchronous way (in a separated thread).
- The optional parameter `[timeout]` describes a timeout value. If no message is received for more than this value, the function `on_timeout_![subscription]()` is called.

TODO: implement this timeout functionality
y
The syntax for describing publishers is similar; it does not have the
the `process` and `timeout` value.

Example:


    subscriptions:
        segment_list:
            topic: ~segment_list
            type: duckietown_msgs/SegmentList
            desc: Line detections
            queue_size: 1
            timeout: 3

    publishers:
        lane_pose:
            topic: ~lane_pose
            type: duckietown_msgs/LanePose
            desc: Estimated pose
            queue_size: 1


### Describing contracts

Note: This is not implemented yet.

The idea is to have a place where we can describe constraints such as:

- "This topic must publish at least at 30 Hz."
- "Panic if you didn't receive a message for 2 seconds."
- "The maximum latency for this is 0.2 s"

Then, we can implement all these checks once and for all in a proper way,
instead of relying on multiple broken implementations

## Using the `easy_node` API


### Initialization

Here is a minimal example of a node that conforms with the API:

    from easy_node import EasyNode

    class MyNode(EasyNode):

        def __init__(self):
            EasyNode.__init__(self, 'my_package', 'my_node')
            self.info('Initialized.')

    if __name__ == '__main__':
        MyNode().spin()

- The node class must derive from `EasyNode`.
- You need to tell EasyNode what is the package name and the node name.
- To initialize, call the function `spin()`.

The `EasyNode` class provides the following functions:

    info()
    debug()
    error()

These are mapped to `rospy.loginfo()` etc.; they include the name of the node.



### Using configuration parameters

This next example shows how to use configuration parameters.

First, create a file `my_node.easy_node.yaml` containing:

    parameters:
        num_cells:
            desc: Number of cells.
            type: int
            default: 42
    subscriptions: {}
    contracts: {}
    publishers: {}

Then, implement the method `on_parameters_changed()`. It takes
two parameters:

- `first_time` is a boolean that tells whether this is the first time
  that the function is called (initialization time).
- `updated` is a set of strings that describe the set of parameters
  that changed. The first time, it contains the set of all parameters.

To access the parameter value, access `self.config.![parameter]`.

Example:

    class MyNode():

        def __init__(self):
            EasyNode.__init__(self, 'my_package', 'my_node')

    def on_parameters_changed(self, first_time, updated):
        if first_time:
            self.info('Initializing array for the first time.')
            self.cells = [0] * self.config.num_cells

        else:
            if 'num_cells' in updated:
                self.info('Number of cells changed.')
                self.cells = [0] * self.config.num_cells

    if __name__ == '__main__':
        Node().spin()

EasyNode will monitor the ROS parameter server, and will call the function
`on_parameters_changed` if the user changes any parameters.


### Using subscriptions

To automatically subscribe to topics, add an entry in the `subscriptions`
section of the YAML file.

For example:

    subscriptions:
        joy:
            desc: |
                The `Joy.msg` from `joy_node` of the `joy` package.
                The vertical axis of the left stick maps to speed.
                The horizontal axis of the right stick maps to steering.
            type: sensor_msgs/Joy
            topic: ~joy
            timeout: 3.0

Then, implement the function `on_received_![name]`.

This function will be passed 2 arguments:

- a `context` object; this can be used for benchmarking ([](#easy_node-benchmarking)).
- the message object.

Example:

    class MyNode():
        # ...

        def on_received_joy(self, context, msg):
            # This is called any time a message arrives
            self.info('Message received: %s' % msg)

### Time-out

TODO: to implement

The function `on_timeout_![subscription]()` is called when
there hasn't been a message for the specified timeout interval.

    class MyNode():
        # ...

        def on_timeout_joy(self, context, time_since):
            # This is called when we have not seen a message for a while
            self.error('No joystick received since %s s.' % time_since)


### Publishers

The publisher object can be accessed at `self.publishers.![name]`.
EasyNode has taken care of all the initialization for you.

For example, suppose we specify a publisher `command` using:

    publishers:
        command:
            desc: The control command.
            type: duckietown_msgs/Twist2DStamped
            topic: ~car_cmd

Then we can use it as follows.

    class MyNode():
        # ...

        def on_received_joy(self, context, msg):
            out = Twist2DStamped()
            out.header.stamp = 0
            out.v = 0
            out.omega = 0

            self.publishers.command.publish(out)

### `on_init()` and `on_shutdown()`

Define the two methods `on_init()` and `on_shutdown()` to c

    class MyNode(EasyNode):
        # ...
       def on_init(self):
            self.info('Step 1 - Initialized')

       def on_parameters_changed(self, first_time, changed):
            self.info('Step 2 - Parameters received')

       def on_shutdown(self):
            self.info('Step 3 - Preparing for shutdown.')

Note that `on_init()` is called before `on_parameters_changed()`.

## Configuration using `easy_node`: the user's point of view

So far, we have seen how to use parameters from the node, but we
did not talk about how to specify the parameters from the user's point of view.

EasyNode introduces lots of flexibility compared to the legacy system.

### Configuration file location

The user configuration is specified using files by the pattern

    ![package_name]-![node_name].![config name].config.yaml

where `![config name]` is a short string (e.g., `baseline`).

The files can be anywhere in:

- The directory `${DUCKIETOWN_ROOT}/catkin_ws/src`;
- The directory `${DUCKIEFLEET_ROOT}`.


Several config files can exist at the same time.
For example, we could have somewhere:

    line_detector-line_detector.baseline.config.yaml
    line_detector-line_detector.fall2017.config.yaml
    line_detector-line_detector.andrea.config.yaml

where the `baseline` versions are the baseline parameters,
`fall2017` are the parameters we are using for Fall 2017,
and `andrea` are temporary parameters that the user is using.

However, there cannot be two configurations with the same filename
e.g. two copies of `line_detector-line_detector.baseline.config.yaml`.
In this case, EasyNode will raise an error.

TODO: implement this functionality.

### Configuration file format

The format of the `*.config.yaml` file is as follows:

    description: |
        ![description of what this configuration accomplishes]
    extends: [![config name], [config name]]
    values:
        ![parameter name]: ![value]
        ![parameter  name]: ![value]

The `extends` field (optional) is a list of string. It allows to use the
specified configurations as the defaults for the current one.

For example, the file `line_detector-line_detector.baseline.config.yaml` could contain:

    description: |
        These are the standard values for the line detector.
    extends: []
    values:
        img_size: [120,160]
        top_cutoff: 40

### Configuration sequence

Which parameters are used depend on the **configuration sequence**.

The configuration sequence is a list of configuration names.

It can be specified by the environment variable `DUCKIETOWN_CONFIG`,
using a colon-separated list of strings. For example:

    $ export DUCKIETOWN_CONFIG=baseline:fall2017:andrea

The line above specifies that the configuration sequence is
`baseline`, `fall2017`, `andrea`.

The system loads the configuration in order. First, it loads the `baseline`
version. Then it loads the `fall2017` version. If a value  was already
specified in the `baseline` version, it is overwritten. If a version does not
exist, it is simply skipped.

If a parameter is not specified in any configuration, an error is raised.

Using this functionality, it is easy to have team-based customization and
user-based customization.

There are two special configuration names:

1. The configuration name "`defaults`" loads the defaults specified by the node.
   Note that the defaults are ignored otherwise.
2. The configuration name "`vehicle`" expands to the name of the vehicle being used.

TODO: the `vehicle` part is not implemented yet.

### Time-variant configuration

EasyNode allows to describe configuration that can change in time.

The use case for this is the configuration of calibration parameters:

- Calibration parameters change with time.
- We still want to access old calibration parameters, when processing logs.

The solution is to allow a date tag in the configuration name. The format for this is

    ![package_name]-![node_name].![config name].![date].config.yaml

For example, we could have the files:

    kinematics-kinematics.ferrari.20160404.config.yaml
    kinematics-kinematics.ferrari.20170101.config.yaml

Given this, EasyNode will select the configuration to use intelligently.
When reading from a bag file from 2016, the first configuration
is going to be used; for logs in 2017, the second is going to be used.

TODO: not implemented yet.

## Visualizing the configuration database

There are a few tolls used to visualize the configuration database.


### `easy_node desc`: Describing a node

The command

    $ rosrun easy_node desc ![package name] ![node name]

shows a description of the node, as specified in `![package name].![node name].easy_node.yaml`.

<div class='usage-example' markdown="1">

For example:

    $ rosrun easy_node desc line_detector2 line_detector2

shows the following:

TODO: output


</div>

### `easy_node eval`: Evaluating a configuration sequence

The program `eval` script allows to evaluate a certain configuration sequence.

The syntax is:

    $ rosrun easy_node eval ![package name] ![node name] ![config sequence]

The tool shows also which file is responsible for the value of each parameter.

<div class='usage-example' markdown="1">

For example, the command

    $ rosrun easy_node desc line_detector2 line_detector2 defaults:andrea

evaluates the configuration for the `line_detector2` node with the configuration
sequence `defaults:andrea`.

The result is:

TODO: output


Note how we can tell which configuration file set each parameter.

</div>

### `easy_node summary`: Describing and validating all configuration files

The program `summary` reads all the configuration files described in the repository
and validates them against the node description.

If a configuration file refers to parameters that do not exist,
the configuration file is established to be invalid.

The syntax is:

    $ rosrun easy_node summary

<div class='usage-example'>

For example, the output can be:

TODO: example

</div>

## Benchmarking {#easy_node-benchmarking}

EasyNode implements some simple timing statistics. These are accessed using
the `context` object passed to the message received callbacks.

Here's an example use, from [`line_detector2`](#line_detector2-line_detector_node2):

    def on_received_image(self, context, image_msg):

        with context.phase('decoding'):
            ...

        with context.phase('resizing'):
            # Resize and crop image
            ...

        stats = context.get_stats()
        self.info(stats)

The idea is to enclose the different phases of the computation
using the [context manager](#python-context-manager) `phase(![name])`.

A summary of the statistics can be accessed by using `context.get_stats()`.

For example, this will print:

TODO: example stats

## Automatic documentation generation

EasyNode generated the documentation automatically from the `*.easy_node.yaml`
files.

Note that this can be used independently of the fact that the node
implements the `EasyNode` API. So, we can use this EasyNode functionality
also to document the legacy nodes.

To generate the docs, use this command:

    $ rosrun easy_node generate_docs

For each node documented using a `*.easy_node.yaml`, this generates a Markdown
file called  "`![node name]-easy_node-autogenerated.md`" in the package
directory.

The contents are enclosed in a `div` with ID `#![package name]-![node name]-autogenerated`.

The intended use is that this can be used to move the contents
to the place in the documentation using [the `move-here` tag](#move-here).

For example, in the `README.md` of the `joy_mapper` package, we have:

    # Package `joy_mapper`

    ![...]

    ## Node `joy_mapper_node`

    <!-- include the automatically generated documenation here -->

    <move-here src="#joy_mapper-joy_mapper_node-autogenerated"/>

The result can be seen at [](#joy_mapper).


## Other ideas

(Add here other ideas that we can implement.)