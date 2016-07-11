Raspberry Pi Installation
--------------------------

This section will cover the initial setup of the Raspberry Pi, which will manage device communication, monitoring, and maintenance. **If you would like to skip the section**, this [disk image]() contains a Raspbian environment with all of the necessary components installed and properly configured.

**Notes**: 
* This guide assumes you're using a Raspberry Pi 3 and will be connecting to your network over the built-in WiFi.
* You will need to connect a monitor to the Pi to follow this guide, but you won't need it after that.
* If you just use the [disk image](), you can avoid connecting a monitor to the Pi.
* All the username, password, and IP values listed in this guide are the default settings as found in the [disk image](). Feel free to change them, Clod will still work fine as long as you remember the changes.


## Raspbian and Localisation Settings

* Download the latest [Raspbian release](https://www.raspberrypi.org/downloads/raspbian/) and follow the [installation guide](https://www.raspberrypi.org/documentation/installation/installing-images/README.md).

* After booting to desktop, set the appropriate location by clicking Menu > Raspberry Pi Configuration > Localisation

* In that same section, also set the appropriate keyboard layout. Verify by pressing the "#" key and making sure a "#" symbol is displayed.

* Now open a terminal window. Menu > Accessories > Terminal. All future steps will be from the terminal window unless otherwise noted.


### Configure WiFi

**From the root directory, check to make sure interfaces refers to wpa_supplicant:**
```
sudo nano/etc/network/interfaces
```

**Enter your ssid and password in wpa_supplicant:**
```
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```
```
network={
   ssid="yourssid"
   psk="yourpassword"
   key_mgmt=WPA-PSK
}
```

**Edit etc/dhcpcd.conf:**
```
sudo nano /etc/dhcpcd.conf
```

```
interface eth0
static ip_address=192.168.1.140/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1 8.8.8.8

interface wlan0
static ip_address=192.168.1.140/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1 8.8.8.8
```
**Now restart dhcpcd**
```
sudo service dhcpcd restart
```

### Change the username to something other than pi (default: clod)

* ```sudo passwd root ```

* enter a good password (default: "!ClodMQTT!")

* From the desktop rather than the terminal window, go to Menu > Preferences > Raspberry Pi Configuration

* uncheck "auto login as pi"

* Menu > Reboot

* Open a terminal window: Menu > Accessories > Terminal

* ``` usermod -l clod pi ```

* ``` usermod -m -d /home/clod clod

* From the desktop, click Menu > Logout

* Enter your new username (default: clod) and "raspberry" as the password

* Open a terminal window: Menu > Accessories > Terminal

* ``` passwd ``` - change the password to something better, the default used in the disk image is "!ClodMQTT!"

* ``` sudo passwd -l root ``` - disables the root password

* ``` sudo groupmod -n clod pi ``` - change the group from pi to clod

### Check that SSHD is enabled

```sudo service sshd status ``` 
The word "active" should be in green along with other information.

If you're going to want to access your pi with SSH later or other advanced fiddling, please consider disabling SSH passwords and using a public/private key pair. Consult the guides [here](http://raspi.tv/2012/how-to-set-up-keys-and-disable-password-login-for-ssh-on-your-raspberry-pi) and [here](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md).

### Components and Dependencies

**Node**

* Open a terminal window: Menu > Accessories > Terminal

* Go to the root directory

* ```curl -sL https://deb.nodesource.com/setup_4.x | sudo -E bash -```

* ```sudo apt-get install -y nodejs```

**Crouton**

* Go to the user home directory (default: home/clod)

* ``` git clone https://github.com/jakeloggins/crouton-new.git ```

**NPM**

* Go to the Crouton folder

* ``` sudo npm update ```

**Bower**

* ``` sudo npm install -g bower ```

* ``` bower install ```

**Later**

* ``` npm install later ```

**Platform Io**

* Go to the user home directory (default: home/clod)

* ``` sudo pip install -U platformio

* ``` platformio platforms install espressif ```

* Go to the .platformio directory (default: home/clod/.platformio)

* ``` mkdir -m 777 lib ```

* ``` cp -a ~/crouton-new/sketch_libraries/. ~/.platformio/lib/ ```


### Mosquitto MQTT Broker

**Install**

* Go to the root directory

* ```sudo wget http://repo.mosquitto.org/debian/mosquitto-repo.gpg.key ```

* ``` sudo apt-key add mosquitto-repo.gpg.key ```

* ``` cd /etc/apt/sources.list.d/ ```

* ``` sudo wget http://repo.mosquitto.org/debian/mosquitto-jessie.list ```

* ``` sudo apt-get update ```

* ``` sudo apt-get install mosquitto ```

* ``` sudo apt-get install mosquitto-clients python-mosquitto ```

**Edit the configuration file**

* ``` sudo nano etc/mosquitto/mosquitto.conf ```

* Make the configuration file look like this:

```
# Place your local configuration in /etc/mosquitto/conf.d/
#
# A full description of the configuration file is at
# /usr/share/doc/mosquitto/examples/mosquitto.conf.example

pid_file /var/run/mosquitto.pid

persistence true
persistence_location /var/lib/mosquitto/

log_dest file /var/log/mosquitto/mosquitto.log

#include_dir /etc/mosquitto/conf.d

listener 1883

listener 9001
protocol websockets
```

**Verify mosquitto is running properly**

* ``` sudo service mosquitto restart ```

* Open another terminal window: Menu > Accessories > Terminal

* type ``` mosquitto_sub -t /# -v ``` in one window

* type ``` mosquitto_pub -t /whatever -m "test" ``` in the other window

* you should now see ``` /whatever test ``` in the first window


### Start Crouton and the Node Scripts
	
**Note**: This step will be eliminated when everything is programmed to run on startup.

* Open a new terminal window and go to the Crouton folder before each of the subsequent steps.
* ``` node app.js ```
* ``` node scheduler.js ```
* ``` node uploader.js ```
* ``` node persistence.js ```
