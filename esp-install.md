ESP8266 Installation
====================

Wiring and Flashing
-------------------

This guide assumes you have some knowledge of how to flash microprocessors using FTDI serial communication. This includes basic knowledge of electrical circuits, resistors, and voltage regulators.

Follow [this excellent MAKE guide](http://makezine.com/2015/04/01/installing-building-arduino-sketch-5-microcontroller/) for the wiring section only, then come back here for the easiest way to flash it using Platform IO.


Using Platform IO
-----------------

Install Platform IO

---link here

Download the Clod-sketch-library to your computer

Open the Initial_Config project folder

edit the conditional definition file (ESPBoardDefs.h) based on the board type you have, here is the list

hit the upload button

close the serial terminal, or remove the usb cable from your chip

power on the chip

follow the rest of this guide


conditional definition file (based on board type) - just have to edit it yourself to match platformio

upload the initial config script

follow the rest of the guide
