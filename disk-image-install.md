Raspberry Pi Disk Image Installation
------------------------------------

* Download the image from here. (coming soon)

* Unzip and mount the image to the memory card by following the instructions specific to your OS.
  * [Windows](https://www.raspberrypi.org/documentation/installation/installing-images/windows.md)
  * [Linux](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md)
  * [Mac](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md)

* After mounting the image to the SD card, open the memory card on your computer and navigate to the boot folder.

* Open and edit the wpa_supplicant.conf to include your wifi network's SSID and password.

* Open and edit the wifilogin.json file to include your wifi network's SSID and password.

* Insert the memory card into your Pi and power it on.

* You are now ready to proceed to the [Getting Started](https://github.com/jakeloggins/Clod#getting-started) section. 

### Defaults

Should you desire to do advanced configuration, here are some default settings:

* User: clod
* Pass: !ClodMQTT!
* User home directory: /home/clod
* Location of Mosquitto MQTT Server: 192.168.1.140, ports 1883 and 9001
* Location of Mosquitto MQTT Configuration file: etc/mosquitto/mosquitto.conf
* Location of Crouton: http://192.168.1.140:8080
* Platform Io Directory: /home/clod/.platformio
