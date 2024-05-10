---
layout: project
type: project
image: img/drone.jpg
title: "Drone Simulation"
date: 2023
published: true
labels:
  - Linux
  - C++
  - Python
  - Ardupilot
summary: "I was tasked to design and operate missions on a simulated drone using Ardupilot libraries"
---

# Drone Simulation Project

In the Fall 2023 semester, I chose drone simulation as my junior project. Dr. Yingfei Dong led the project, overseeing myself and a handful of other students. We used a virtual machine to run Ubuntu, which would be our working environment. The SITL simulator was used to simulate the flight paths of the drone. We would use Python scripts to run automated missions on the drone, where I successfully learned how to program a drone's flight pattern.

<div>
	<img src="https://ardupilot.org/dev/_images/sitl.jpg">
</div>

Here is the final Python script that I created to control the drone.

### finalmission.py
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from __future__ import print_function
from dronekit import connect, VehicleMode, LocationGlobalRelative, LocationGlobal, Command
import time
import math
from pymavlink import mavutil

# Establish connection to drone
master = mavutil.mavlink_connection('udp:0.0.0.0:14550')
master.wait_heartbeat()

###############################################################
# Establish drone controls
vehicle = connect('127.0.0.1:14550', wait_ready=True, heartbeat_timeout=99999)

'''cmds = vehicle.commands
cmds.download()
cmds.wait_ready()'''

home_location = vehicle.location.global_relative_frame

vehicle.mode = VehicleMode("GUIDED")

def arm_and_takeoff(aTargetAltitude):
    """
    Arms vehicle and fly to aTargetAltitude.
    """
    
    print ("Basic pre-arm checks")
    # Don't try to arm until autopilot is ready
    while not vehicle.is_armable:
        print (" Waiting for vehicle to initialise...")
        time.sleep(1)
    print ("armable= %s"% vehicle.is_armable)

    print ("Arming motors")
    # Copter should arm in GUIDED mode

    vehicle.mode = 'GUIDED'

    while vehicle.mode != VehicleMode("GUIDED"):
        print (" Waiting for changing mode...")
        vehicle.mode = 'GUIDED'
        print ("vehicle.mode= %s"% vehicle.mode)
        time.sleep(1)
    print ("vehicle.mode= %s"% vehicle.mode)

    vehicle.armed   = True
    
    time.sleep(1)
    # Confirm vehicle armed before attempting to take off

    while not vehicle.armed:
        print (" Waiting for arming...")
        time.sleep(1)

    print ("vehicle.armed= %s"% vehicle.armed)


    print ("Taking off!")

    time.sleep(2)

    vehicle.simple_takeoff(aTargetAltitude)
     # Take off to target altitude

    # Wait until the vehicle reaches a safe height before processing the goto (otherwise the command
    #  after Vehicle.simple_takeoff will execute immediately).
    while True:
        print (" Altitude: ", vehicle.location.global_relative_frame.alt)
        #Break and return from function just below target altitude.
        if vehicle.location.global_relative_frame.alt>=aTargetAltitude*0.95:
            print ("Reached target altitude")
            break
        time.sleep(1)

arm_and_takeoff(100)

def send_ned_velocity(velocity_x, velocity_y, velocity_z, duration):
    """
    Move vehicle in direction based on specified velocity vectors.
    """
    msg = vehicle.message_factory.set_position_target_local_ned_encode(
        0,       # time_boot_ms (not used)
        0, 0,    # target system, target component
        mavutil.mavlink.MAV_FRAME_LOCAL_NED, # frame
        0b0000111111000111, # type_mask (only speeds enabled)
        0, 0, 0, # x, y, z positions (not used)
        velocity_x, velocity_y, velocity_z, # x, y, z velocity in m/s
        0, 0, 0, # x, y, z acceleration (not supported yet, ignored in GCS_Mavlink)
        0, 0)    # yaw, yaw_rate (not supported yet, ignored in GCS_Mavlink)

    # send command to vehicle on 1 Hz cycle
    for x in range(0,duration):
        vehicle.send_mavlink(msg)
        time.sleep(1)

# Set up velocity mappings
# velocity_x > 0 => fly North
# velocity_x < 0 => fly South
# velocity_y > 0 => fly East
# velocity_y < 0 => fly West
# velocity_z < 0 => ascend
# velocity_z > 0 => descend
print('going to waypoint A')
send_ned_velocity(-2,2,0,15)

def get_location_metres(original_location, dNorth, dEast):

	earth_radius=6378137.0 #Radius of &quot;spherical&quot; earth
	#Coordinate offsets in radians
	dLat = dNorth/earth_radius
	dLon = dEast/(earth_radius*math.cos(math.pi*original_location.lat/180))

	#New position in decimal degrees
	newlat = original_location.lat + (dLat * 180/math.pi)
	newlon = original_location.lon + (dLon * 180/math.pi)
	if type(original_location) is LocationGlobal:
		targetlocation=LocationGlobal(newlat, newlon,original_location.alt)
	elif type(original_location) is LocationGlobalRelative:
		targetlocation=LocationGlobalRelative(newlat, newlon,original_location.alt)
	else:
		raise Exception("Invalid Location object passed")

	return targetlocation;

waypoint_B = get_location_metres(vehicle.location.global_relative_frame, -50, 0)

vehicle.groundspeed = 7

print('going to waypoint B')
vehicle.simple_goto(waypoint_B)

def distance_to_current_waypoint():
	nextwaypoint = vehicle.commands.next
	if nextwaypoint==0:
		return None
	missionitem=vehicle.commands[nextwaypoint-1] #commands are zero indexed
	lat = missionitem.x
	lon = missionitem.y
	alt = missionitem.z
	targetWaypointLocation = LocationGlobalRelative(lat,lon,alt)
	distancetopoint = get_distance_metres(vehicle.location.global_frame,
	targetWaypointLocation)
	return distancetopoint

def download_mission():
	cmds = vehicle.commands

	cmds.download()
	cmds.wait_ready() # wait until download is complete.

def adds_waypoint_mission(aLocation):
	cmds = vehicle.commands

	print("Clear any existing commands")
	cmds.clear()

	print("Define/add new commands")
	# Add new commands. The meaning/order of the parameters is documented in the Command class.

	#Add MAV_CMD_NAV_TAKEOFF command. This is ignored if the vehicle is already in the air.
	cmds.add(Command( 0, 0, 0, mavutil.mavlink.MAV_FRAME_GLOBAL_RELATIVE_ALT, mavutil.mavlink.MAV_CMD_NAV_TAKEOFF, 0, 0, 0, 0, 0, 0, 0, 0, 10))

	#Define the four MAV_CMD_NAV_WAYPOINT locations and add the commands
	point1 = get_location_metres(aLocation, -100, 100)
	point2 = get_location_metres(point1, -100, 0)
	point3 = get_location_metres(point2, 0, 200)
	point4 = get_location_metres(point3, -30, 0)

	cmds.add(Command( 0, 0, 0, mavutil.mavlink.MAV_FRAME_GLOBAL_RELATIVE_ALT, mavutil.mavlink.MAV_CMD_NAV_WAYPOINT, 0, 0, 0, 0, 0, 0, point1.lat, point1.lon, 11))
	cmds.add(Command( 0, 0, 0, mavutil.mavlink.MAV_FRAME_GLOBAL_RELATIVE_ALT, mavutil.mavlink.MAV_CMD_NAV_WAYPOINT, 0, 0, 0, 0, 0, 0, point2.lat, point2.lon, 12))
	cmds.add(Command( 0, 0, 0, mavutil.mavlink.MAV_FRAME_GLOBAL_RELATIVE_ALT, mavutil.mavlink.MAV_CMD_NAV_WAYPOINT, 0, 0, 0, 0, 0, 0, point3.lat, point3.lon, 13))
	cmds.add(Command( 0, 0, 0, mavutil.mavlink.MAV_FRAME_GLOBAL_RELATIVE_ALT, mavutil.mavlink.MAV_CMD_NAV_WAYPOINT, 0, 0, 0, 0, 0, 0, point4.lat, point4.lon, 14))
	#add dummy waypoint &quot;5&quot; at point 4 (lets us know when have reached destination)
	cmds.add(Command( 0, 0, 0, mavutil.mavlink.MAV_FRAME_GLOBAL_RELATIVE_ALT, mavutil.mavlink.MAV_CMD_NAV_WAYPOINT, 0, 0, 0, 0, 0, 0, point4.lat, point4.lon, 14))

	print("Upload new commands to vehicle")
	cmds.upload()

def get_distance_metres(aLocation1, aLocation2):
    dlat = aLocation2.lat - aLocation1.lat
    dlong = aLocation2.lon - aLocation1.lon
    return math.sqrt((dlat*dlat) + (dlong*dlong)) * 1.113195e5

def get_bearing(aLocation1, aLocation2):
    off_x = aLocation2.lon - aLocation1.lon
    off_y = aLocation2.lat - aLocation1.lat
    bearing = 90.00 + math.atan2(-off_y, off_x) * 57.2957795
    if bearing < 0:
        bearing += 360.00
    return bearing;

adds_waypoint_mission(vehicle.location.global_relative_frame)

print("Starting mission")
vehicle.commands.next=0

vehicle.mode = VehicleMode("AUTO")

while True:
	nextwaypoint=vehicle.commands.next

	print('Distance to waypoint (%s): %s' % (nextwaypoint, distance_to_current_waypoint()))

	if nextwaypoint==3: #Skip to next waypoint
		print('Skipping to Waypoint 4 when reach waypoint 2')
		vehicle.commands.next = 4
	if nextwaypoint==5: #Dummy waypoint - as soon as we reach waypoint 4 this is true and we exit.
		print("Exit mission when start heading to final waypoint (5)")
		break;
	time.sleep(1)

print('Distance: %s, Bearing: %s' % (get_distance_metres(home_location, vehicle.location.global_relative_frame), get_bearing(home_location, vehicle.location.global_relative_frame)))

print('Return to launch')
vehicle.mode = VehicleMode("RTL")

time.sleep(5)
print('Finished Final')
```

Here is the [Ardupilot Repository](https://github.com/Jaguda01/ardupilot) that I used to write my missions.
