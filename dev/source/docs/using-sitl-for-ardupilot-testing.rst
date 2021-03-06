.. _using-sitl-for-ardupilot-testing:

================================
Using SITL for ArduPilot Testing
================================

This article describes how to preform a number of common ArduPilot
testing tasks in :ref:`SITL <sitl-simulator-software-in-the-loop>` using
:ref:`MAVProxy <mavproxy-developer-gcs>`.

Overview
========

The :ref:`SITL (Software In The Loop) <sitl-simulator-software-in-the-loop>` simulator is a build of
the ArduPilot code which allows you to run
:ref:`Plane <plane:home>`,
:ref:`Copter <copter:home>` or
:ref:`Rover <copter:home>` without any hardware. It can be
built and run on :ref:`Windows <sitl-native-on-windows>` or
:ref:`Linux <setting-up-sitl-on-linux>`, and can be run on Mac OSX (or the
other platforms) in a virtual machine with Linux installed.

Using SITL is just like using a real vehicle: you can connect to the
(simulated) vehicle using the Ground Control Station (GCS) of your
choice (or even multiple ground stations), take off, change flight
modes, make guided or automatic missions, and land. The main difference
is that in addition to being able to configure the vehicle,
*simulator-specific* parameters allow you to configure the *physical
environment* (for example wind speed and direction) and also to simulate
failure of different components. This means that SITL is the perfect
environment to test bug fixes and other changes to the autopilot,
failure modes, and DroneKit-Python apps.

This article explains some of the more important parameters that you can
set to change the environment, simulate failure modes, and configure the
vehicle with optional components. It also explains how to :ref:`connect to different GSCs <using-sitl-for-ardupilot-testing_connecting_otheradditional_ground_stations>`.

.. tip::

   If you're just getting started with MAVProxy and SITL you may wish
   to start by reading the :ref:`Copter SITL/MAVProxy Tutorial <copter-sitl-mavproxy-tutorial>`
   (or equivalent tutorials for the other vehicles).

.. note::

   These instructions generally use
   :ref:`MAVProxy <mavproxy-developer-gcs>` to
   describe operations (e.g. setting parameters) because it presents a
   simple and consistent command-line interface (removing the need to
   describe a GSC-specific UI layout). There is no reason the same
   operations cannot be performed in *Mission Planner* (through the *Full
   Parameters List*) or any other GSC.

Setting vehicle start location
==============================

You can start the simulator with the vehicle at a particular location by
calling **sim_vehicle.sh** with the ``-L`` parameter and a named
location in the
`ardupilot/Tools/autotest/locations.txt <https://github.com/diydrones/ardupilot/blob/master/Tools/autotest/locations.txt>`__
file.

For example, to start Copter in *Ballarat* (a named location in
**locations.txt**) call:

::

    cd ArduCopter 
    sim_vehicle.sh -j4 -L Ballarat --console --map

.. note::

   You can add your own locations to the file. If you need to use a
   location regularly then consider adding it to the project via a pull
   request.

   
.. _using-sitl-for-ardupilot-testing_loading_a_parameter_set:

Loading a parameter set
=======================

When starting SITL the first time, the device may be configured with
"unforgiving" parameters. Typically you will want to replace these with
values that simulate more realistic vehicle and environment conditions.
Useful parameter sets are provided in the autotest source for
`Copter <https://github.com/diydrones/ardupilot/blob/master/Tools/autotest/copter_params.parm>`__,
`Plane <https://github.com/diydrones/ardupilot/blob/master/Tools/autotest/ArduPlane.parm>`__
and
`Rover <https://github.com/diydrones/ardupilot/blob/master/Tools/autotest/Rover.parm>`__.

.. tip::

   This only needs to be done once. After loading the parameters are
   stored in the simulated EEPROM.

The MAVProxy commands to load the parameters for Copter, Rover and Plane
(assuming the present working directory is a vehicle directory like
**/ardupilot/ArduCopter/**) are shown below:

::

    param load ..\Tools\autotest\copter_params.parm

::

    param load ..\Tools\autotest\ArduPlane.parm

::

    param load ..\Tools\autotest\Rover.parm

You can re-load the parameters later if you choose, or revert to the
default parameters by starting SITL (**sim_vehicle.sh**) with the
``-w`` flag.

Parameters can also be saved. For example, to save the parameters into
the present working directory you might do:

::

    param save ./myparams.parm

Setting parameters
==================

Many of the following tasks involve setting parameter values over
MAVLink, which you do using the ``param set`` command as shown:

::

    param set PARAMETERNAME VALUE

All available parameters can be listed using ``param show``. The
SITL-specific parameters start with ``SIM_``, and can be obtained using:

::

    param show SIM_*

.. tip::

   When you change a parameter the value remains in the virtual EEPROM
   after you restart SITL. Remember to change it back if you don't want it
   any more (or :ref:`reload/reset the parameters <using-sitl-for-ardupilot-testing_loading_a_parameter_set>`). 

Testing RC failsafe
===================

To test the behaviour of ArduPilot when you lose remote control (RC),
set the parameter ``SIM_RC_FAIL=1``, as shown:

::

    param set SIM_RC_FAIL 1

This simulates the complete loss of RC input. If you just want to
simulate low throttle (below throttle failsafe level) then you can do
that with the RC command:

::

    rc 3 900

Testing GPS failure
===================

To test losing GPS lock, use ``SIM_GPS_DISABLE``:

::

    param set SIM_GPS_DISABLE 1

You can also enable/disable a 2nd GPS using ``SIM_GPS2_ENABLE``.

Testing the effects of vibration
================================

To test the vehicle's reaction to vibration, use ``SIM_ACC_RND``. The
example below adds 3 m/s/s acceleration noise:

::

    param set SIM_ACC_RND 3

Testing the effects of wind
===========================

The wind direction, speed and turbulence can be changed to test their
effect on flight behaviour. The following settings changes the wind so
that it blows towards the South at a speed of 10 m/s.

::

    param set SIM_WIND_DIR 180
    param set SIM_WIND_SPD 10

To see other wind parameters do:

::

    param show sim_wind*

.. note::

   At time of writing the wind parameters only appear to work for
   Plane.

Adding a virtual gimbal
=======================

SITL can simulate a virtual gimbal.

.. note::

   Gimbal simulation causes SITL to start sending
   `MOUNT_STATUS <http://mavlink.org/messages/ardupilotmega#MOUNT_STATUS>`__
   messages. These messages contain the orientation according to the last
   commands sent to the gimbal, not actual measured values. As a result, it
   is possible that the true gimbal position will not match - i.e. a
   command might be ignored or the gimbal might be moved manually. Changes
   are not visible in Mission Planner.

First start the simulator and use the following commands to set up the
gimbal mount:

::

    # Specify a servo-based mount:
    param set MNT_TYPE 1

    # Set RC output 6 as pan servo:
    param set RC6_FUNCTION 6

    # Set RC output 8 as roll servo:
    param set RC7_FUNCTION 8

Then stop and re-launch SITL with the ``-M`` flag:

::

    sim_vehicle.sh -M

Adding a virtual rangefinder
============================

SITL can simulate an analog rangefinder, which is very useful for
developing flight modes that can use a rangefinder. To set it up use the
following commands:

::

    param set SIM_SONAR_SCALE 10
    param set RNGFND_TYPE 1
    param set RNGFND_SCALING 10
    param set RNGFND_PIN 0
    param set RNGFND_MAX_CM 5000

    # Enable rangefinder for landing (Plane only!)
    param set RNGFND_LANDING 1

The above commands will setup an analog rangefinder with a maximum range
of 50 meters (the 50m comes from an analog voltage range of 0 to 5V, and
a scaling of 10). After making the above changes you need to restart
SITL.

Then to test it try this:

::

    module load graph
    graph RANGEFINDER.distance

Then try a flight and see if the graph shows you the rangefinder
distance.

.. tip::

   You can also use the following commands to graph rangefinder
   information (defined as *MAVProxy* aliases):

   -  ``grangealt`` - graph rangefinder distance and relative altitude.
   -  ``grangev`` - rangefinder voltage
   -  ``grange`` - graph "rangefinder_roll"

Adding a virtual optical flow sensor
====================================

You can add a virtual optical flow sensor like this:

::

    param set SIM_FLOW_ENABLE 1
    param set FLOW_ENABLE 1

Then restart SITL. After setting it up try this:

::

    module load graph
    graph OPTICAL_FLOW.flow_x OPTICAL_FLOW.flow_y

Go for a flight and see if you get reasonable data.

Accessing log files
===================

SITL supports both tlogs and DF logs (same as other types of ArduPilot
ports). The DF logs are stored in a "logs" subdirectory in the directory
where you start SITL. You can also access the DF logs via MAVLink using
a GCS, but directly accessing them in the logs/ directory is usually
more convenient.

To keep your tlogs organised it is recommended you start SITL using the
"--aircraft NAME" option. That will create a subdirectory called NAME
which will have flight logs organised by date. Each flight will get its
own directory, and will include the parameters for the flight plus any
downloaded waypoints and rally points.

Graphing vehicle state
======================

*MAVProxy* allows you to create graphs of vehicle state. Numerous
aliases have been created for useful graph types in the *MAVProxy*
initialisation file (**mavinit.scr**). These all start with "g" and
include ``gtrackerror``, ``gaccel`` etc.

Using a joystick
================

It can be useful to use a joystick for input in SITL. The joystick can
be a real RC transmitter with a USB dongle for the trainer port, or
something like the RealFlight interlink controller or a wide range of
other joystick devices.

Before you use the joystick support you need to remove a debug
statements from the python-pygame joystick driver on Linux. If you don't
then you will see lots of debug output like this:

::

    SDL_JoystickGetAxis value:-32768:

To remove this debug line run this command:

::

    sudo sed -i 's/SDL_JoystickGetAxis value/\x00DL_JoystickGetAxis value/g' /usr/lib/python2.7/dist-packages/pygame/joystick.so

note that this needs to be one long command line. Ignore the line
wrapping in the wiki.

Then to use the joystick run:

::

    module load joystick

If you want to add support for a new joystick type then you need to edit
the `mavproxy_joystick module <https://github.com/tridge/MAVProxy/blob/master/MAVProxy/modules/mavproxy_joystick.py>`__

Using real serial devices
=========================

Sometimes it is useful to use a real serial device in SITL. This makes
it possible to connect SITL to a real GPS for GPS device driver
development, or connect it to a real OSD device for testing an OSD.

To use a real serial device you can use a command like this:

::

    sim_vehicle.sh -A "--uartB=uart:/dev/ttyUSB0" --console --map

what that does it pass the --uartB argument to the ardupilot code,
telling it to use /dev/ttyUSB0 instead of the normal internal simulated
GPS for the 2nd UART.

Any of the 5 UARTs can be configured in this way, using uartA to uartE.

.. _using-sitl-for-ardupilot-testing_connecting_otheradditional_ground_stations:

Connecting other/additional ground stations
===========================================

SITL can connect to multiple ground stations by using *MAVProxy* to
forward UDP packets to the GCSs network address. Alternatively SITL can
connect to a GCS over TCP/IP without using *MAVProxy*.

.. _using-sitl-for-ardupilot-testing_sitl_with_mavproxy_udp:

SITL with MAVProxy (UDP)
------------------------

SITL can connect to multiple ground stations by using *MAVProxy* to
forward UDP packets to the GCSs network address (for example, forwarding
to another Windows box or Android tablet on your local network). The
simulated vehicle can then be controlled and viewed through any attached
GCS.

First find the IP address of the machine running the GCS. How you get
the address is platform dependent (on Windows you can use the 'ipconfig'
command to find the computer's address).

Assuming the IP address of the GCS is 192.168.14.82, you would add this
address/port as a *MAVProxy* output using:

::

    output add 192.168.14.82:14550

The GCS would then connect to SITL by listening on that UDP port. The
method for connecting will be GCS specific (we show :ref:`how to connect for Mission Planner <using-sitl-for-ardupilot-testing_connecting_mission_planner_udp>` below).

.. tip::

   If you're running the GCS on the **same machine** as SITL then an
   appropriate output may already exist. Check this by calling ``output``
   on the *MAVProxy command prompt*:

   ::

       GUIDED> output
       GUIDED> 2 outputs
       0: 127.0.0.1:14550
       1: 127.0.0.1:14551

   In this case we can connect a GCS running on the same machine to UDP
   port 14550 or 14551. We can choose to connect another GCS to the
   remaining port, and add more ports if needed. 

   
.. _using-sitl-for-ardupilot-testing_sitl_without_mavproxy_tcp:

SITL without MAVProxy (TCP)
---------------------------

It is also possible to interact with SITL over TCP/IP by starting it
using \ *vehicle_name*.\ **elf** (e.g. **/ArduCopter/ArduCopter.elf**).
*MAVProxy* is not needed when using this method.

Run the file in the *Cygwin Terminal*, specifying a home position and
vehicle model as shown below:

::

    $ ./ArduCopter.elf --home -35,149,584,270 --model quad
    Started model quad at -35,149,584,270 at speed 1.0
    Starting sketch 'ArduCopter'
    Starting SITL input
    bind port 5760 for 0
    Serial port 0 on TCP port 5760
    Waiting for connection ....

The command output shows that you can connect to SITL using TCP/IP at
the network address of the **machine SITL is running on** at port 5760.

.. tip::

   **ArduCopter.elf** has other startup options, which you can use
   using the -h command line parameter:

   ::

       ./ArduCopter.elf -h

.. _using-sitl-for-ardupilot-testing_connecting_mission_planner_udp:

Connecting Mission Planner (UDP)
--------------------------------

First set up SITL to :ref:`output UDP packets to the address/port of the computer running *Mission Planner* <using-sitl-for-ardupilot-testing_sitl_with_mavproxy_udp>`.

In *Mission Planner* listen to the specific UDP port by selecting
**UDP** and then the **Connect** button. Enter the port to listen on
(the default port number of 14550 should be correct if SITL is running
on the same computer).

.. figure:: ../../../images/MissionPlanner_Connect_UDP.jpg
   :target: ../_images/MissionPlanner_Connect_UDP.jpg

   Mission Planner: Connecting to a UDPPort

Connecting to Mission Planner (TCP)
-----------------------------------

First set up SITL :ref:`for use with TCP <using-sitl-for-ardupilot-testing_sitl_without_mavproxy_tcp>`.

In *Mission Planner* connect to SITL by selecting **TCP** and then the
**Connect** button. Enter the \ *remote host* and *remote Port* of the
machine running SITL. *Mission Planner* will then connect and can be
used just as before.

.. tip::

   If SITL is running on the same machine as *Mission Planner* you can
   click through the \ *remote host* and *remote Port* prompts as these
   default to the correct values.

.. figure:: ../../../images/MissionPlanner_ConnectTCP.jpg
   :target: ../_images/MissionPlanner_ConnectTCP.jpg

   Mission Planner: Connecting toSITL using TCP
