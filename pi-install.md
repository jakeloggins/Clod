Raspberry Pi Manual Installation
--------------------------

This section will cover the initial setup of the Raspberry Pi, which will manage device communication, monitoring, and maintenance. **If you would like to skip the section**, this [disk image](disk-image-install.md) contains a Raspbian environment with all of the necessary components installed and properly configured.

**Notes**: 
* This guide assumes you're using a Raspberry Pi 3 and will be connecting to your network over the built-in WiFi.
* You will need to connect a monitor to the Pi to follow this guide, but you won't need it after that.
* If you just use the [disk image](disk-image-install.md), you can avoid connecting a monitor to the Pi.
* All the username, password, and IP values listed in this guide are the default settings as found in the [disk image](disk-image-install.md). Feel free to change them, Clod will still work fine as long as you remember the changes.


## Raspbian and Localisation Settings

* Download the latest [Raspbian release](https://www.raspberrypi.org/downloads/raspbian/) and follow the [installation guide](https://www.raspberrypi.org/documentation/installation/installing-images/README.md).

* After booting to desktop, set the appropriate location by clicking Menu > Preferences > Raspberry Pi Configuration > Localisation

* In that same section, also set the appropriate keyboard layout. Verify by pressing the "#" key and making sure a "#" symbol is displayed.

* Still in that section, set the appropriate timezone.

* Click ok and agree to the reboot.

* Now open a terminal window. Menu > Accessories > Terminal. All future steps will be from the terminal window unless otherwise noted.


### Configure WiFi

**From the root directory, verify the interfaces file refers to wpa_supplicant:**
```
cd /
sudo nano etc/network/interfaces
```

**Assuming the reference to wpa_supplicant is there, exit the file by pressing CTRL+X**

**Enter your ssid and password in wpa_supplicant:**
```
sudo nano etc/wpa_supplicant/wpa_supplicant.conf
```
```
network={
   ssid="yourssid"
   psk="yourpassword"
   key_mgmt=WPA-PSK
}
```

**Press CTRL+X, Y to save the file, and ENTER**

**From the root directory and create a boot/wifilogin.json file**
```
sudo nano boot/wifilogin.json
```
```
{
	"ssid": "yourWifiSSIDHere",
	"password": "yourWifiPASSWORDHere"
}
```
**Note the comma after the SSID line**


**Press CTRL+X, Y to save the file, and ENTER**

**Edit etc/dhcpcd.conf:**
```
sudo nano etc/dhcpcd.conf
```
**Append the following to the bottom of the file:**

```
interface eth0
static ip_address=192.168.1.160/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1 8.8.8.8

interface wlan0
static ip_address=192.168.1.160/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1 8.8.8.8
```

**Press CTRL+X, Y to save the file, and ENTER**

**Now restart dhcpcd**

```
sudo service dhcpcd restart
```

**Reboot**
```
sudo reboot
```

**Verify ip address was assigned correctly**
```
ifconfig wlan0
```

**Output should match what you entered in the dhcpcd.conf file (default: 192.168.1.160)**
```
inet addr:192.168.1.160 ...
```

### Change the username to something other than pi (default: clod)

* ```sudo passwd root ```

* enter a good password (default: "!ClodMQTT!")

* From the desktop rather than the terminal window, go to Menu > Preferences > Raspberry Pi Configuration

* uncheck `Auto login: Login as user 'pi'`

* Click OK and agree to reboot

* Login by selecting 'Other' then enter user 'root' and the password you entered earlier (default: "!ClodMQTT!")

* Open a terminal window (Menu > Accessories > Terminal) and enter the following:

* ``` usermod -l clod pi ```

* ``` usermod -m -d /home/clod clod

* From the desktop, click Menu > Shutdown > Logout

* Enter your new username (default: clod) and "raspberry" as the password

* Open a terminal window: Menu > Accessories > Terminal

* ``` passwd ``` - change the password to something better, the default used in the disk image is "!ClodMQTT!"

* ``` sudo passwd -l root ``` - follow the prompts to disable the root password

* ``` sudo groupmod -n clod pi ``` - change the group from pi to clod

### Check that SSHD is enabled

```sudo service sshd status ``` 
The word "active" should be in green along with other information.

If you're going to want to access your pi with SSH later or other advanced fiddling, please consider disabling SSH passwords and using a public/private key pair. Consult the guides [here](http://raspi.tv/2012/how-to-set-up-keys-and-disable-password-login-for-ssh-on-your-raspberry-pi) and [here](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md).

### Components and Dependencies

**Node**

* Open a terminal window: Menu > Accessories > Terminal

* Go to the root directory `cd /`

* ```curl -sL https://deb.nodesource.com/setup_4.x | sudo -E bash -```

* ```sudo apt-get install -y nodejs```

**Clod**

* Go to the user home directory (default: home/clod) `cd home/clod`

* ``` git clone https://github.com/jakeloggins/Clod-scripts.git ```

* ``` git clone https://github.com/jakeloggins/Clod-sketch-library.git ```

* ``` git clone https://github.com/jakeloggins/crouton-new.git ```

**NPM**

* Go to the Crouton folder `cd crouton-new`

* ``` sudo npm update ```

**Bower**

* ``` sudo npm install -g bower ```

* ``` bower install ```

**Later**

* ``` npm install later ```

**Platform Io**

* Go to the user home directory (default: home/clod) `cd ..`

* ``` sudo pip install -U platformio ```

* ``` platformio platforms install espressif8266 ```



### Mosquitto MQTT Broker

**Install**

* Go to the root directory `cd /`

* ```sudo wget http://repo.mosquitto.org/debian/mosquitto-repo.gpg.key ```

* ``` sudo apt-key add mosquitto-repo.gpg.key ```

* ``` cd /etc/apt/sources.list.d/ ```

* ``` sudo wget http://repo.mosquitto.org/debian/mosquitto-jessie.list ```

* ``` sudo apt-get update ```

* ``` sudo apt-get install mosquitto ```

* ``` sudo apt-get install mosquitto-clients python-mosquitto ```

**Edit the configuration file**

* ``` cd / ```

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

**Press CTRL+X, Y to save the file, and ENTER**

**Verify mosquitto is running properly**

* ``` sudo service mosquitto restart ```

* Open another terminal window: Menu > Accessories > Terminal

* enter ``` mosquitto_sub -t /# -v ``` in the new terminal window

* enter ``` mosquitto_pub -t /whatever -m "test" ``` in the other terminal window

* you should now see ``` /whatever test ``` in the first window

* close the new terminal window


### Setup Systemd
	
Systemd will make all of the Clod-scripts and the crouton dashboard run automatically on boot.

* Navigate to `/etc/systemd/system`

* Create the following files, naming them ClodUploader.service, ClodScheduler.service, ClodPersistence.service, and ClodCrouton.service:

	* `sudo nano ClodUploader.service` 
	* `sudo nano ClodScheduler.service`
	* `sudo nano ClodPersistence.service` 
	* `sudo nano ClodCrouton.service`

Example files:

```
[Unit]
Description=Clod Uploader

[Service]
WorkingDirectory=/home/clod/Clod-scripts
ExecStart=/usr/bin/node /home/clod/Clod-scripts/uploader.js
Restart=always
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=ClodUploader
User=root
Group=root
Environment=NODE_ENV=production

[Install]
WantedBy=default.target

---

[Unit]
Description=Clod Scheduler

[Service]
WorkingDirectory=/home/clod/Clod-scripts
ExecStart=/usr/bin/node /home/clod/Clod-scripts/scheduler.js
Restart=always
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=ClodScheduler
User=root
Group=root
Environment=NODE_ENV=production

[Install]
WantedBy=default.target

---

[Unit]
Description=Clod Persistence

[Service]
WorkingDirectory=/home/clod/Clod-scripts
ExecStart=/usr/bin/node /home/clod/Clod-scripts/persistence.js
Restart=always
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=ClodPersistence
User=root
Group=root
Environment=NODE_ENV=production

[Install]
WantedBy=default.target

--

[Unit]
Description=Clod Crouton

[Service]
WorkingDirectory=/home/clod/crouton-new
ExecStart=/usr/bin/node /home/clod/crouton-new/app.js
Restart=always
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=ClodCrouton
User=root
Group=root
Environment=NODE_ENV=production

[Install]
WantedBy=default.target
```

* `sudo chmod u+rwx` to all the files

	* `sudo chmod u+rwx ClodUploader.service` 
	* `sudo chmod u+rwx ClodScheduler.service`
	* `sudo chmod u+rwx ClodPersistence.service` 
	* `sudo chmod u+rwx ClodCrouton.service`

* `sudo systemctl daemon-reload` - make systemd aware of changed files


* Install Clod-scripts node modules
	* Navigate to Clod-scripts directory
	* `npm install jsonfile`
	* `npm install mqtt`
	* `npm install fs`
	* `npm install later`
	* `npm install util`
	
* `sudo systemctl enable ClodUploader.service`

* `sudo systemctl enable ClodScheduler.service`

* `sudo systemctl enable ClodPersistence.service`

* `sudo systemctl enable ClodCrouton.service`

* `sudo reboot`

* `sudo systemctl` - check the list to see if all of the Clod Services are running.