Clod
======

Clod is a system for easily deploying IoT devices based on espressif chips like the ESP8266. It is intended to be self-contained within the user's network, so that data and controls are private and isolated from the internet. Its components are a disorganized mess of other great open source software projects that together form a coherent user experience. A clod is a clump of dirt, which is the opposite of a stack, and as far as one can get from a cloud.

The main components of Clod are:

* Mosquitto MQTT server - intended to run on a Raspberry Pi, this facilitates communication with the devices via the MQTT protocol.

* [Clod MQTT standard](https://github.com/jakeloggins/clod-MQTT-standard) - an intuitive location-based syntax for MQTT messages.

* [Clod Scripts](https://github.com/jakeloggins/clod-scripts) - monitors the messages in order to add functionality such as persistence, scheduling, and uploading.

* Crouton - a dashboard for controlling devices and viewing their output.

* Platform Io - manages OTA software updates and changes to the espressif chips.

* Arduino core for ESP8266.

* [ESP8266 WiFI Manager](https://github.com/tzapu/WiFiManager)

* [Clod Sketch Library](https://github.com/jakeloggins/clod-sketch-library) - Arduino sketches that have been modified for use with Clod and ESP8266 chips.



Raspberry Pi Installation
--------------------------

This section will cover the initial setup of the Raspberry Pi, which will manage device communication, monitoring, and maintenance. There are two options for completing it:

* A [step-by-step guide](pi-install.md):
  * Approximately 1 hour to complete all of the steps.
  * Requires the use of a monitor connected to the Pi throughout the setup process.

* A [disk image](disk-image-install.md) with Raspbian and all of the steps in the guide already completed.
  * Significantly faster process, just mount the disk image to an SD card and insert it into the Pi.
  * Does not require the use of a monitor.



ESP8266 Installation
--------------------



Getting Started
----------------

* **Prerequisites**:
  * All of the components installed, configured and running properly on your Raspberry Pi. 
  * An esp8266 chip loaded with the initial configuration script.


* Point your browser to the Crouton landing page (default: http://192.168.1.140:8080).

* Turn on your esp8266 and connect to the wifi network "Clod" with a mobile device.

* Select your home wifi network's SSID from the list and enter the password.

* Make a note of the information displayed on the screen.

* In Crouton, select the "Connections" tab.

* In the "Upload" section, select the device from the available devices drop down and [upload your desired sketch][]

* Once successfully uploaded, select the device from the "Devices" section and click add.

* View information and controls from your device on the "Dashboard" tab.

* You're done. Enjoy!

Data Structure
--------------



Create Your Own Sketch
----------------------
