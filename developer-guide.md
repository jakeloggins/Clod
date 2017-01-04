Developer Guide
===============
	


















































Overall Flow and Purpose
------------------------

Clod is an open source control system for espressif chip projects. Clod does not connect to the cloud and is not meant to interface with commerical IoT devices. Commercial, cloud-dependent IoT devices are too expensive and generally a [bad idea](https://twitter.com/internetofshit). The system is designed for makers who want to quickly create something and control it, and for those without programming experience who want to follow a simple tutorial. This advanced guide is to help developers learn how to create and share projects.

The intended use case of Clod is some espressif chips, programmed in arduino, communicating over MQTT to a Raspberry Pi. The user will send commands or view data from the chips through MQTT messages or an optional web dashboard. The MQTT broker (mosquitto), web dashboard, and other administrative node scripts are all contained on the Raspberry Pi. Clod is intended to stay behind a LAN and not connect to the broader internet. 

However, not everyone will follow the intended use. Thus, the project is split into separate components to make it easy to customize and understand. These are:

* Clod MQTT Standard - a common syntax designed for console readability

* Crouton Dashboard - a web dashboard featuring various buttons, toggles, graphs, and menus. Helps with additional functionality such as OTA uploading and scheduling.

* Clod Scripts - a series of node scripts that listen to MQTT traffic and assist with features such as initial configuration, device persistence, OTA uploading, and scheduling.

* Clod Sketch Library - Arduino sketch templates that allow users to easily customize, replicate, and share projects.

Here's a chart showing how everything is intended to fit together:

![User-flow-chart](https://raw.githubusercontent.com/jakeloggins/Clod/master/img/complete_flow.png)

































Clod MQTT Standard
==================

MQTT is a messaging protocol that is perfect for the Internet of Things. Messages are sent to topics, which is a string separated by forward slashes ` /just/like/this ` and contain payloads that can be a string ` "like this" ` or an object `{ "like": "this" }" `. 

A typical message might look something like this: ` /this/is/the/topic "and this is the payload" `. A device will receive the payload (`"and this is the payload"`) only if it is *subscribed* to `/this/is/the/topic `. 

An MQTT standard is just an agreed way to format the topic and payload so that users and devices can easily understand each other. Development around IoT, and espressif chips in particular, is constantly changing. Since Clod is a disorganized mess of other great open source software projects, the Clod MQTT Standard is designed to display information intuitively for users and allow multiple languages to understand it.

* Intutive when viewed in a continuous output stream

* Allow user customization

* Able to be parsed by multiple programming languages

* Easy for developers to grasp when developing new GUI or device sketches

* Can integrate seamlessly with third party services (slack chat, text message, twitter, etc)


#### Topic Format

The Clod MQTT topic format is:

` /[location based path]/[command]/[device name]/[endpoint] `

Examples:

```
[location based path] > /house/upstairs/guestroom 
[command] > /control
[device name] > /myEspDevice
[endpoint] > /temperature
```

All together, that would look like this:

` /house/upstairs/guestroom/control/myEspDevice/temperature `


##### Location based path

If a user has only one device, any topic format will do. But with multiple devices, location based topic paths offer the following advantages:

* A more intuitive experience when viewing a raw stream of messages on the broker. The path tells you the location and the command describes what is happening. The output gets more detailed as you read from left to right.

* The ability to easily view a subset of your network. For example, a subscription to /house/upstairs/# will show you everything that is going on within upstairs including devices at /upstairs/guestroom and upstairs/bathroom.

* A better foundation for the addition of global commands. In the python client examples, the placement of command in the middle of the path string allows the device to parse whether a global command applies to it's location. Eventually, Clod scripts will allow you to send a message like ` /global/house/upstairs/control/lights "off" ` to turn off all the upstairs lights. 

Any number of user-defined combinations are allowed. For example, `/house/upstairs/ ` or ` /house/upstairs/guestroom/closet/storagebox/shoebox/russian-nesting-dolls/large/medium/small ` are both perfectly fine location paths. First, they are descriptive and helpful to the user. Second, they get more specific and limiting as read from left to right. 

**Note**: The user is free to not use location based paths by simply assigning the same path to each each device, like `/default` or `/house`. 


##### Command

There are four commands: control, confirm, errors, and log. Clod first uses the command to parse the topic. Once Clod recognizes the command, it reads everything to the left of it as part of the `[path]`, and everything to the right as the `[name]` and `[endpoint]`.

* A control command tells the device to do something. `[path]/control/[name]/[endpoint]`

* A confirm command informs Clod that the control message has been executed or updates Clod with new data. `[path]/confirm/[name]/[endpoint]` 

* When a device loses power or is disconnected from the MQTT broker, it sends a message to `[path]/errors/[name]/` to notify Clod. 

* The log command currently does nothing, but is reserved for future use. `[path]/log/[name]` 


##### Name

Device names are provided by the user during the upload process and/or hardcoded into the sketch. Spaces in the name are allowed, but will be converted to [camelCase](https://en.wikipedia.org/wiki/CamelCase) and assigned to device_name_key in the device object. More information about device objects are provided below.


##### Endpoint

A device can have multiple endpoints. Each endpoint represents a dashboard card that will be displayed for the user to control or view data from the device.

The device must subscribe to the `/[path]/control/[name][endpoint]` of each endpoint which may receive values from the user. For example, a toggle switch may change a value on the device therefore a subscription is necessary; however, an alert button which sends only messages from device to to the dashboard may not need a subscription because the device does not expect any values from the user.

```
Subscription address for endPoints on device:
/[path]/control/[name]/[endpoint name]
```

Upon receiving a new value from the user, the device **must** send back the new value or the appropriate value back to the dashboard on [path]/confirm/[name]/[endpoint]. This is because the dashboard will not reflect the new value change *unless* it is coming from the device.


```
Address: /[path]/confirm/[name]/[endpoint name]
Payload: {"value": "some new value here"}
```


##### Startup and Scripts

Most routine communication between the user and devices are covered by the proceeding sections. However, a device's initial connection and use of the Clod Scripts require special topic formatting. This section simply notes the topic syntax for these processes. A more in-depth guide is featured below.

 * deviceInfo is where information about the device is exchanged between the user, devices, and Clod. 
   * ` /deviceInfo/[command]/[name] `

 * init is used by esp chips when they have first connected to Clod but have not yet gone through the upload process. 
   * ` /init/[command]/[chipID] `

 * the persistence script maintains information about all devices within Clod and makes it available to other devices and scripts 
   * ` /persistence/[command]/name `

 * the uploader script allows a user to customize and upload a sketch from the [Clod Sketch Library](https://github.com/jakeloggins/Clod) to an esp chip. 
   * ` /uploader/[command]/name `

 * the scheduler sends normal control commands to endpoints at specified times. 
   * ` /scheduler/[path]/[action type]/[name]/[endpoint]/[value] `



#### Payload Format

The payload format is simply an object, depending on the following contexts. 

##### Device Objects

All of a device's information that is required by Clod can be found in its device object. The device object is sent to `/deviceInfo/` whenever a user adds a device to the Crouton dashboard, a new sketch is uploaded, or after loss of connection to the MQTT broker. The device object is the primary method for Crouton to understand the device. It is the first message the device will send to Crouton to establish connection and also the message that describes the device and values to Crouton. 

Device object example:

```json
{
  "deviceInfo": {
    "current_ip": "192.168.1.141",
    "type": "esp",
    "espInfo": {
        "chipID": "16019999",
        "board_type": "esp01_1m",
        "platform": "espressif",
        "flash_size": 1048576,
        "real_size": 1048576,
        "boot_version": 4,
        "upload_sketch": "basic_esp8266",
    },
    "device_name": "Floodlight Monitor",
    "device_name_key": "floodlightMonitor",
    "device_status": "good",
    "path": "/backyard/floodlight/",
    "card_display_choice": "custom",
    "endPoints": {
      "lastTimeTriggered": {
        "title": "Last Time Triggered",
        "card-type": "crouton-simple-input",
        "static_endpoint_id": "time_input",
        "values": {
          "value": "n/a"
        }
      },
      "alertEmail": {
        "title": "Alert Email",
        "card-type": "crouton-simple-input",
        "static_endpoint_id": "alert_input",
        "values": {
          "value": "your_email_address@gmail.com"
        }
      }
    }
  } 
}
```

* *type*: Specifies whether the device is an esp chip. Required by the uploader.
* *espInfo*: Information about the esp chip. Required by the uploader.
* *device_name*: A string that is the name for the device.
* *device_name_key*: A camelized version of the device_name so that the MQTT topic will not include a space.
* *device_status*: A string that describes the status of the device.
* *path*: Specifies the location based topic path.
* *card_display_choice*: Notes whether the user chose to specify custom endpoints during the upload process.
* *endPoints*: An object that configures each dashboard element Crouton will show. There can be more than one endPoint which would be key/object pairs within *endPoints*
* *card-type*: How data should be displayed from the endpoint on the dashboard. See next section.
* *description*: A string that describes the device (for display to user only)

**Note**: Both *device_name* and *endPoints* are required and must be unique to other *device_names* or *endPoints* respectively

<!--
### Card Types

Since Clod evolved from the Crouton dashboard, each endpoint must follow a Crouton card's payload format. For a full description of the available dashboard cards, go [here](https://github.com/jakeloggins/crouton-new#dashboard-cards). Most cards require sending a `values` object containing a single `value` key to the endpoint's topic. Some cards, such as the [Line Chart](https://github.com/jakeloggins/crouton-new#line-chart), have a more complicated `values` object.
-->

##### Updating Endpoints

Updates to endpoints can come from user or the device. The message payload will be a object that updates the value. This object will be equivalent to the object of the key `values` within each endpoint. 

```json
Payload: {"value": 35}

An entry in endPoints:
"barDoor": {
  "title": "Bar Main Door",
  "card-type": "crouton-simple-text",
  "units": "people entered",
  "values": {
      "value": 34
  }
}
```

##### From User

The user has the ability to update the value of the device's endpoints via certain dashboard cards. Therefore the device needs to be subscribe to all of its endpoint topics. The payload from the user is in the same format as the one coming from the device.

```
Address: /[path]/control/Kroobar/barDoor
Payload: {"value": 35}
```

##### From Device

To update values on the dashboard from the device, simply publish messages to the `confirm` of the endPoint which Crouton is already subscribed to. The payload is just the same as the one coming from Crouton.

```
Address: /[path]/confirm/Kroobar/barDoor
Payload: {"value": 35}
```

##### Last will and testament (LWT)

In order for Crouton to know when the device has unexpectedly disconnected, the device must create a LWT with the MQTT Broker. This a predefined broadcast that the broker will publish on the device's behalf when the device disconnects. The payload in this case can be anything as long as the address is correct.

```
Address: /[path]/errors/Kroobar
Payload: anything
```



























































[Crouton Dashboard](https://github.com/jakeloggins/crouton-new)
================

Crouton is a dashboard that lets you visualize and control your IOT devices with minimal setup. Essentially, it is the easiest dashboard to setup for any IoT hardware enthusiast using only MQTT and JSON.



Connecting to Crouton
--------------

First, have Crouton and the device connected to the same MQTT Broker. The connection between the device and Crouton will be initiated from Crouton, therefore the device needs to subscribe to its own path.

```
Device should subscribe to the following:
/deviceInfo/[the device name]/control
```

Every time the device successfully connects to the MQTT Broker, it should publish its *deviceInfo*. This is needed for auto-reconnection. If Crouton is waiting for the device to connect, it will listen for the *deviceInfo* when the device comes back online.

```
Device should publish deviceInfo JSON once connected
/deviceInfo/[the device name]/confirm
```

### DeviceInfo

<!-- This might not be true anymore with persistence -->

The deviceInfo is the primary method for Crouton to understand the device. It is the first message the device will send to Crouton to establish connection and also the message that describes the device and values to Crouton. The primary object is *deviceInfo*. Within *deviceInfo* there are several keys as follows:

```json
{
  "deviceInfo": {
    "device_name": "kroo bar",
    "device_name_key": "krooBar",
    "device_status": "connected",
    "current_ip": "192.xxx.xxx.xxx",
    "type": "esp",
    "path": "/bar/front/entrance",
    "card_display_choice": "default",
    "description": "Kroobar's IOT devices",
    "espInfo": {
      ...
    },
    "endPoints": {
      "barDoor": {
        "title": "Bar Main Door",
        "card-type": "crouton-simple-text",
        "units": "people entered",
        "values": {
            "value": 34
        }
      }
    }
  }
}
```

* *device_name*: A string that is the name for the device. This is same name you would use to add the device
* *device_name_key*: a camelized version of device_name. Necessary for the MQTT topic.
* *device_status*: A string that describes the status of the device 
* *current_ip*: the IP address assigned during the upload process
* *type*: required for the persistence script.
* *path*: Specifies the location to publish and subscribe on the mqtt broker
* *card_display_choice*: used during the upload process. options are default or custom
* *espInfo*: an object with data about the espressif chip. Used during initial configuration, upload, and persistence.
* *description*: A string that describes the device (for display to user only)
* *endPoints*: An object that configures each dashboard element Crouton will show. There can be more than one endPoint which would be key/object pairs within *endPoints*

<!--
  * *function*: A string within an endpoint that is used to group together endpoints for global commands
-->


**Note**: Both *device_name* and *endPoints* are required and must be unique to other names or *endPoints* respectively

<!--
**Note**: There is now an additional method for adding and altering single card devices, discussed below. If you have multiple cards and store the deviceInfo JSON in your script, simply select "Auto Import" and type in the device name to add to the dashboard.
-->

### Addresses

Addresses are what Crouton and the device will publish and subscribe to. They are also critical in making the communication between Crouton and the devices accurate therefore there is a structure they should follow.

```
/[path]/[command type]/[device name]/[endPoint name]
```

*command type*: Helps the device and crouton understand the purpose of a message. Generally, *control* is for messages going *to* the device and *confirm* is for messages *from* the device. Last will and testament messages are sent to *errors*. A final command type, *log*, is reserved for future use.

```
control, confirm, errors, log
```

*path*: a custom prefix where all messages will be published. Using location names is recommended. Command type words may not be used within the path.

```
ex: /house/downstairs/kitchen
```

*device name*: The name of the device the are targeting; from *name* key/value pair of *deviceInfo*

*endPoint name*: The name of the endPoint the are targeting; from the key used in the key/object pair in *endPoints*

**Note**: All addresses must be unique to one MQTT Broker. Therefore issues could be encounter when using public brokers where there are naming conflicts.

### Updating device values

Updating device values can come from Crouton or the device. The message payload will be a JSON that updates the value. This JSON will be equivalent to the object of the key *values* within each endPoint. However, only values that are being updated needs to be updated. All other values must be updated by the deviceInfo JSON.

```json
Payload: {"value": 35}

An entry in endPoints:
"barDoor": {
  "title": "Bar Main Door",
  "card-type": "crouton-simple-text",
  "static_endpoint_id": "door",
  "units": "people entered",
  "values": {
      "value": 34
  }
}
```

##### From Crouton

Crouton has the ability to update the value of the device's endPoints via certain dashboard cards. Therefore the device needs to be subscribe to certain addresses detailed in the Endpoints section below. The payload from Crouton is in the same format as the one coming from the device.

```
Address: /[path]/control/Kroobar/barDoor
Payload: {"value": 35}
```

##### From Device

To update values on Crouton from the device, simply publish messages to the outbox of the endPoint which Crouton is already subscribed to. The payload is just the same as the one coming from Crouton.

```
Address: /[path]/confirm/Kroobar/barDoor
Payload: {"value": 35}
```

### Last will and testament (LWT)

In order for Crouton to know when the device has unexpectedly disconnected, the device must create a LWT with the MQTT Broker. This a predefined broadcast that the broker will publish on the device's behalf when the device disconnects. The payload in this case can be anything as long as the address is correct.

```
Address: /[path]/errors/Kroobar
Payload: anything
```

### Endpoints

A device can have multiple endPoints. Each endPoint represents a dashboard card that will be displayed on the dashboard.

The device must subscribe to the [path]/control of each endPoint which may receive values from Crouton. For example, a toggle switch on Crouton may change a value on the device therefore a subscription is necessary; however, an alert button which sends only messages from device to Crouton may not need a subscription because the device does not expect any values from Crouton.

```
Subscription address for endPoints on device:
/[path]/control/[device name]/[endpoint name]
```

Upon receiving a new value from Crouton, the device **must** send back the new value or the appropriate value back to Crouton on [path]/confirm/[device name]/[endpoint name]. This is because Crouton will not reflect the new value change *unless* it is coming from the device.

Therefore the value shown on Crouton more accurately reflects the value on the device.

```
Address: /[path]/confirm/[device name]/[endpoint name]
Payload: {"value": "some new value here"}
```
























Dashboard Cards
==============

Dashboard cards are visual and control elements which allow interaction with connected devices. By simply adding a new endPoint in the deviceInfo JSON, a new dashboard card will be automatically generated upon connection. They strive to do several things:

* Provide a visual experience for IOT devices
* Modularity allows different combinations of different devices
* Simplicity of define functionality for each type of card
* Reflect the latest real value of the device

The type of dashboard card for each endPoint is specified in the object of the endPoint under the key *card-type*. The *values* object is the object that is sent between Crouton and the device to keep values up-to-date. Both of these fields are required for each endPoint.

```json
"barDoor": {
  ...
  "card-type": "crouton-simple-text",
  "values": {
    ...
  }
  ...
}
```

Any icons used come from [Font Awesome](https://fortawesome.github.io/Font-Awesome/icons/) and utilizes the same name for the icons.

Also, new cards are always coming so keep the suggestions coming!

## Simple cards

Simple cards are basic cards that have only one value. Therefore if they are buttons, inputs, etc, there will only be one button, inputs, etc per card. Their functionality are fairly limited but by using multiple cards together, they can still be powerful. All simple card have the same prefix.

```
Simple card prefix:
crouton-simple-[card type]
```

### Simple Text

![Crouton-simple-text](https://raw.githubusercontent.com/jakeloggins/crouton-new/master/public/common/images/crouton-simple-text.png) </br> Simple text is used to display a value (text or number) on the dashboard from the device to Crouton.

```json
Device -> Crouton
Name: crouton-simple-text

Example:
"barDoor": {
  "units": "people entered", [optional]
  "values": {
    "value": 34 [required]
  },
  "card-type": "crouton-simple-text", [required]
  "title": "Bar Main Door" [optional]
}
```

### Simple Dropdown

A dropdown menu is used to select a value from the `choices` array.

```json
Device <- Crouton
Name: crouton-simple-dropdown

Example:
"treeAnimation": {
  "card-type": "crouton-simple-dropdown",
  "static_endpoint_id": "animationMenu",
  "title": "Tree Animation",
  "values": {
    "choices": [
      "Fun Random", 
      "Random Sparkle",
      "White Sparkle",
      "Fade Strip In Out",
      "Fade Strip In",
      "Clear and Fade Strip In",
      "Alternate Fade In",
      "Alternate Fade In Out",
      "Flare",
      "Flare Reverse",
      "Color Wipe",
      "Color Wipe Reverse",
      "Random Color",
      "Rainbow"
    ]
  }
}
```

### Simple Input

![Crouton-simple-text](https://raw.githubusercontent.com/jakeloggins/crouton-new/master/public/common/images/crouton-simple-input.png) </br> Simple input is similar to simple text except the user can update the value on the device from Crouton. There is no length restriction of the value by Crouton.

```json
Device <-> Crouton
Name: crouton-simple-input

Example:
"customMessage": {
  "values": {
    "value": "Happy Hour is NOW!" [required]
  },
  "card-type": "crouton-simple-input", [required]
  "title": "Billboard Message" [optional]
}
```

### Simple Slider

![Crouton-simple-text](https://raw.githubusercontent.com/jakeloggins/crouton-new/master/public/common/images/crouton-simple-slider.png) </br> Simple slider allows the user to select continuous values within a given range. Both the large number and the slider will attempt the give the real device value at all times except when the user is sliding.

```json
Device <-> Crouton
Name: crouton-simple-slider

Example:
"barLightLevel": {
  "values": {
    "value": 30 [required]
  },
  "min": 0, [required]
  "max": 100, [required]
  "units": "percent", [optional]
  "card-type": "crouton-simple-slider", [required]
  "title": "Bar Light Brightness" [optional]
}
```

### Simple Button

![Crouton-simple-text](https://raw.githubusercontent.com/jakeloggins/crouton-new/master/public/common/images/crouton-simple-button.png) </br> Simple button is one directional, sending a signal (with no meaningful value) from Crouton to the device. However, this is still a bi-directional card because the button is only enable if value is *true*. If the device updates the value of the card to *false*, the button will be disabled.

```json
Device <-> Crouton
Name: crouton-simple-button

Example:
"lastCall": {
  "values": {
    "value": true [required]
  },
  "icons": {
    "icon": "bell" [required]
  },
  "card-type": "crouton-simple-button", [required]
  "title": "Last Call Bell" [optional]
}
```

### Simple Toggle

![Crouton-simple-text](https://raw.githubusercontent.com/jakeloggins/crouton-new/master/public/common/images/crouton-simple-toggle.png) </br> Simple toggle allows a boolean value to be changed by both Crouton and the device. In the larger value display, priority for display is icon, labels, boolean text. If no labels or icons are given, the words true and false will be used. The labels around the toggle is only defined by *labels* object.

```json
Device <-> Crouton
Name: crouton-simple-toggle

Example:
"backDoorLock": {
  "values": {
    "value": false [required]
  },
  "labels":{ [optional]
    "true": "Locked",
    "false": "Unlocked"
  },
  "icons": { [optional]
    "true": "lock",
    "false": "lock"
  },
  "card-type": "crouton-simple-toggle", [required]
  "title": "Employee Door" [optional]
}
```


## Chart cards

These cards are for charts!

### Donut Chart

![Crouton-chart-donut-1](https://raw.githubusercontent.com/jakeloggins/crouton-new/master/public/common/images/crouton-chart-donut-1.png)

![Crouton-chart-donut-1](https://raw.githubusercontent.com/jakeloggins/crouton-new/master/public/common/images/crouton-chart-donut-2.png)

</br> A fairly flexible pie chart. The *labels* and *series* (values) are in arrays. The labels are optional (must have at least an empty array) and will not show if empty. *message* is displayed in the center of the donut. *centerSum* (defualt is false) sums up all of the values and replaces *message*. *total* is the value that will fill up the complete circle. If sum of *series* is beyond *total*, the extra parts will be truncated.

```json
Device -> Crouton
Name: crouton-chart-donut

Example:
"drinksOrdered": {
  "values": {
    "labels": ["Scotch","Rum & Coke","Shiner","Margarita", "Other"], [required]
    "series": [10,20,30,10,30], [required]
    "message": "" [optional]
  },
  "total": 100, [required]
  "card-type": "crouton-chart-donut",
  "title": "Drinks Ordered" [optional]
},
"occupancy": {
  "values": {
    "labels": [], [required]
    "series": [76] [required]
  },
  "total": 100, [required]
  "centerSum": true, [optional]
  "units": "%", [optional]
  "card-type": "crouton-chart-donut", [required]
  "title": "Occupancy" [optional]
}
```

### Line Chart

![Crouton-chart-line](https://raw.githubusercontent.com/jakeloggins/crouton-new/master/public/common/images/crouton-chart-line.png)

</br> A simple line chart with multiple lines available. The *labels* corresponds to the x axis values and the *series* corresponds to the y axis values. Multiple sets of (x,y) values can be passed at once as long as the array length of labels and series are matched. The reason why series is multidimensional is so that multiple lines can be drawn where each array in series corresponds to a line. *It is suggested that labels and series be prepopulated with one set of (x,y) value for each line* The *update* parameter is expected on update and holds a copy of *values* with the new *labels* and *series* within. *Max* refers to the maximum number of data points based on the x axis is shown. *low* and *high* refers to the maximum y values expected.

*It is suggested that labels and series be prepopulated with one set of (x,y) value for each line*

```json
Device -> Crouton
Name: crouton-chart-line

Example:
"temperature": {
    "values": {
        "labels": [1],  [required]
        "series": [[60]],  [required]
        "update": null  [required]  
    },
    "max": 11, [required]
    "low": 58,  [required]
    "high": 73,  [required]
    "card-type": "crouton-chart-line",  [required]
    "title": "Temperature (F)" [optional]
}

3 lines and 1 value:
"values": {
    "labels": [1],
    "series": [[60],[2],[543]],
    "update": null  
}

3 lines and 2 values:
"values": {
    "labels": [1,2],
    "series": [[60,62],[2,4],[543,545]],
    "update": null  
}

To update temperature:
"values": {
    "labels": null,
    "series": null,
    "update": {
      "labels" : [2],
      "series": [[62]]
    }
}
```

## Advanced cards

These cards are a little bit more specific to certain applications.

### RGB Slider

![Crouton-rgb-slider](https://raw.githubusercontent.com/jakeloggins/crouton-new/master/public/common/images/crouton-rgb-slider.png) </br> RGB slider is three combined slider for the specific application of controlling a RGB led. Prepopulate the values for red, green and blue by setting in values.

```json
Device <-> Crouton
Name: crouton-rgb-slider

Example:
"discoLights": {
  "values": {
    "red": 0, [required]
    "green": 0, [required]
    "blue": 0 [required]
  },
  "min": 0, [required]
  "max": 255, [required]
  "card-type": "crouton-rgb-slider", [required]
  "title": "RGB Lights" [optional]
}
```




















































<!--Transition here between here is the interface and how it works, but you can also make your own, here's the overview with Clod Scripts walkthrough-->



Clod Scripts Walkthrough
==============

This section explains the behavior of the Clod scripts as an esp chip is added to the system and a user performs typical interactions with it.


Init_Config
-------------- 

* To start, you need to know what kind of esp chip you have and hard code the values into the `ESPBoardDefs.h` file within the Init_Config's `src` folder. Platformio Io needs these values within the uploader script later on in the process. To find out what you should enter in this file, go [here](http://docs.platformio.org/en/latest/platforms/espressif.html#espressif).

* After verifying the values, flash the Init_Config sketch to the chip. The sketch opens an access point with the SSID "Clod." When connected, users are prompted to enter the user/pass of their home WiFi network.

* Clod scripts requires an MQTT broker to be running at the location specified in `mqtt_broker_config.json`. [Here's](https://github.com/jakeloggins/Clod/blob/master/pi-install.md#mosquitto-mqtt-broker) a quick guide to setting up a mosquitto MQTT broker.

* The sketch gets more information about the chip, formats it into an object, and sends it to ` /init/control/[chipID] `

* The persistence script grabs the object and adds it to the active_init_list.json file.

active_init_list.json example:
```json
{
  "16019999": {
	"current_ip": "192.168.1.141",
	"type": "esp",
    "espInfo": {
   		"chipID": "16019999",
		"board_type": "esp01_1m",
		"platform": "espressif",
		"flash_size": 1048576,
		"real_size": 1048576,
		"boot_version": 4
    }
  }
}
```

Uploader
---------

The uploader script receives information about the esp chip, the name of the sketch to be uploaded, and other information provided by the user. It writes a platformio.ini file within the sketch folder on the Pi, and some other definitional files, before executing a PlatformIO command to upload the sketch. Output from PlatformIO is logged to a file within the sketch folder and monitored. The uploader process will repeat 3 times, or until the sketch is successfully uploaded.

* Uploader takes the above object from initial config and builds the following object with provided user input.

  * Note: Although this user input is currently done with Crouton, like everything else with Clod, it will work the same way with anything that can produce and send an object over MQTT.

* Adds user input: upload_sketch, name, path, endpoint names and values if card_display_choice is custom.

* If card_display_choice is "default", it will grab the "default_endpoints.json" file from the sketch folder and insert it.

* Modifies the init_device_name def file in the sketch folder so that the esp knows its name on startup.

(link to endpoints section of MQTT standard docs)

(link to default_endpoints section of create your own sketch)


Example upload object sent to ` /deviceInfo/control/[name] `
```json
{
  "deviceInfo": {
	"current_ip": "192.168.1.141",
	"type": "esp",
    "espInfo": {
   		"chipID": "16019999",
		"board_type": "esp01_1m",
		"platform": "espressif",
		"flash_size": 1048576,
		"real_size": 1048576,
		"boot_version": 4,
    	"upload_sketch": "basic_esp8266",
    },
    "device_name": "Floodlight Monitor",
    "device_name_key": "floodlightMonitor",
    "device_status": "good",
    "path": "/backyard/floodlight/",
    "card_display_choice": "custom",
    "endPoints": {
      "lastTimeTriggered": {
        "title": "Last Time Triggered",
        "card-type": "crouton-simple-input",
        "static_endpoint_id": "time_input",
        "values": {
          "value": "n/a"
        }
      },
      "alertEmail": {
        "title": "Alert Email",
        "card-type": "crouton-simple-input",
        "static_endpoint_id": "alert_input",
        "values": {
          "value": "your_email_address@gmail.com"
        }
      }
    }
  }	
}
```

If the upload is successful, the script will:

* send the same object back to ` /deviceInfo/confirm/[name] ` so that it is stored by the persistence script to all_devices.

* send a slightly different object to ` /uploader/confirm/[name] ` that can be used by a GUI to notify the user. That object will look like this:

```
{
	"device_name": "Floodlight Monitor",
	"upload_sketch": "basic_esp8266",
	"result": "success"
}
```

If the upload is not successful, the script will:

* send a retry message: ` /uploader/confirm/[name] "retry attempt # x" `

... or, when x is more than 2:

* send a failed message: ` /uploader/confirm/[name] "failed" ` 



Persistence
-----------

The persistence script is the workhorse of the Clod system. It listens to MQTT traffic in order to update device information files, handles requests from devices, and helps Crouton function properly. 


* Manage requests from newly-added esp chips

* Manage new additions of devices to the Crouton dashboard

* Listen for Last Will and Testament ("LWT") messages to update the connection status of device.

* List to confirm messages to update endpoint states.

* Maintain all_devices.json, active_device_list.json, active_init_list.json, and active_device_list_esp.json


#### ESP chip after upload

1. Persistence adds a newly uploaded device to all_devices as seen in the `esp-uploaded-example` device object below and to the active device lists. 

2. In the Crouton connections tab, the active devices will show up in the "Select Available Device" dropdown menu. 

3. Crouton will add the device by sending ` /deviceInfo/control/[name] "get" `. 

4. Persistence will respond by sending the device object, an example of which can be seen below in `esp-uploaded-example`, to ` /deviceInfo/confirm/[name] `. 

5. Crouton uses the device object to fill out the dashboard display cards.

Here's what all_devices.json looks like with two device objects:

```json
{
	"esp-uploaded-example": {
		"deviceInfo":{
			"current_ip": "192.168.1.141",
			"type": "esp",
			"espInfo": {
				"chipID": "16013813",
				"board_type": "esp01_1m",
				"platform": "espressif",
				"flash_size": 1048576,
				"real_size": 1048576,
				"boot_version": 4,
				"upload_sketch": "basic_esp8266",
    		},
			"device_status": "good",
			"device_name": "esp-uploaded-example",
			"description": "testing new uploader",
			"color":"#4D90FE",
			"path":"/house/upstairs/spare-room/test",
			"card_display_choice": "custom",
			"endPoints":{
				"lastTimeTriggered": {
					"title": "Last Time Triggered",
					"card-type": "crouton-simple-input",
					"static_endpoint_id": "time_input",
					"values":{
						"value": "n/a"
					}
				},
				"alertEmail": {
					"title": "Alert Email",
					"card-type": "crouton-simple-input",
					"static_endpoint_id": "email_input",
					"values": {
						"value": "blah.blah@gmail.com"
					}
				}
			}
		}
	},
	"crouton-demo-new":{
		"deviceInfo":{
			"current_ip": "192.168.1.135",
			"type":"python script on pi",
			"device_status":"good",
			"device_name":"crouton-demo-new",
			"device_name_key": "crouton-demo-new",
			"description":"Kroobar's IOT devices",
			"color":"#4D90FE",
			"path":"/house/downstairs/office/test",
			"card_display_choice": "default",
			"endPoints":{
				"lastCall":{
				   "card-type":"crouton-simple-button",
				   "title":"Last Call Bell",
				   "values":{
				      "value":true
				   },
				   "icons":{
				      "icon":"bell"
				   }
				},
				"danceLights":{
				   "function":"toggle",
				   "card-type":"crouton-simple-toggle",
				   "labels":{
				      "false":"OFF",
				      "true":"ON"
				   },
				   "values":{
				      "value":true
				   },
				   "title":"Dance Floor Lights"
				},
				"drinks":{
				   "units":"drinks",
				   "card-type":"crouton-simple-text",
				   "values":{
				      "value":83
				   },
				   "title":"Drinks Ordered"
				}
			}
		}
	}
}

```


#### List of Currently Connected Devices

Persistence keeps a running list of all_devices who have a device_status of "connected" ...

active_device_list:
``` ["crouton-demo-new", "esp-uploaded-example"] ```

... and a list of devices that are "connected" and have "type:esp" (this is only for use by the uploader).

active_device_list_esp:
``` ["esp-uploaded-example"] ```


#### Remembering States

Within Clod, when a user sends a device a command, a MQTT message is sent to the ` /[path]/control/[name]/[endpoint] ` topic. All devices receive the command, do whatever they are programmed to do, and then return the message to the `/[path]/confirm/[name]/[endpoint] ` topic. Persistence listens and stores all messages that come in on the confirm topics. Persistence is always available to reply to devices requesting their last known states, whether during the startup process or any other reason. It will reply by sending a message to every endpoint's control topic. 

* As an example, imagine a device programmed with a single endpoint that toggled a switch. By default, the switch is `off`. The user toggles the switch `on`, causing a message to be sent: ` /[path]/control/[name]/[endpoint] "on" `

* The device loses power, so it will startup with the default state of `off`. But that's not what the user intends.

* Instead, the device asks persistence for the last state: ``` /persistence/control/[name] "request states" ```. 

* Persistence replies with exactly the same message that a user would trigger: ` /[path]/control/[name]/[endpoint] "on" `

* The device doesn't know or care that the message came from persistence, it just responds as it normally would to any message on the control channel by doing whatever it was programmed to do.

* If the user wasn't around when the device was interrupted, it will have no idea that this persistence bit happened. The switch is just `on` as intended.


#### ESP Startup Process

1. On startup, esp devices ask persistence for its states by publishing to ``` /persistence/control/[name] "request states" ```. 

2. Persistence checks active_devices for the name key.
  
  * If found, it returns the endpoint states in a series of ` /[path]/control/[name]/[endpoint] ` messages. One for each endpoint within the device.

  * If not found, it sends `/deviceInfo/control/[name] "no states" `. If this happens, something went wrong during the upload process.

4. The device sets endpoints accordingly and starts to function according to the sketch.

5. If the esp disconnects from the MQTT broker or loses power, this process will repeat.


#### Non-ESP Startup Process

1. On startup, devices ask persistence for its states by publishing to ``` /persistence/control/[name] "request states" ```. 

2. Persistence checks active_devices for the name key.
  
  * If found, it sends the entire device object to `/deviceInfo/control/[name] `

  * If not found, it sends `/deviceInfo/control/[name] "no states" `. The device then sends its internally stored device object to ` /deviceInfo/confirm/[name] `. See example clients for more details.

4. The device sets endpoints accordingly and starts to function as programmed.

5. If the device disconnects from the MQTT broker or loses power, this process will repeat.


#### IP Verification

The uploader script needs to know the device's IP address to function properly. Therefore, every time a device restarts or re-connects to the MQTT broker, it should verify the IP address stored in all_devices is accurate.

* Device sends persistence its IP: ` /persistence/control/[name]/ip "192.168.1.99" `

* Persistence checks to see if it matches and updates all_devices if necessary.

  * Return ` /persistence/confirm/[name]/ip "no change" ` if IP matched.

  * Return ` /persistence/confirm/[name]/ip "192.168.1.99" ` if IP did not match.


#### LWT messages

Devices should be programmed to send an LWT message to ` /[location path]/errors/[name] `. Persistence will mark the device as disconnected within all_devices and remove it from the active lists. Depending on the circumstances, persistence may also delete the device from all_devices.


Scheduler
---------

The scheduler sends normal MQTT commands to endpoints at specified times. It can be used through the dashboard or by sending a message to the /schedule topic. It does not interract with all_devices or the active lists. All data is stored in schedule_data.json.

example schedule_data:

```json
{
	"esp-bedside-lamp":{
		"alarmLight":{
			"interval":{},
			"schedule":{
				"schedules":[{"D":[1]}],
				"exceptions":[],
				"error":6
			},
		"plain_language":"every day at 5pm",
		"action":"toggle",
		"path":"/house/upstairs/bedroom/",
		"value":"off"}
	},
	"idk":{
		"the endpoint":{
			"interval":{},
			"schedule":{
				"schedules":[],
				"exceptions":[],
				"error":6
			},
		"plain_language":"every morning at 8am",
		"action":"toggle",
		"path":"/whatever/man/",
		"value":"45"
		}
	}
}
```


#### Command Syntax
On recepit of a message to /schedule/#, the scheduler gets information about the endpoint from the rest of the message topic, and parses the message payload to create the schedule. The example message will ask the scheduler to lock the "backDoorLock" endpoint every night at 6pm. To change it to every night at 7pm, simply send another message to the same topic with a payload of "every night at 7pm."
```
/schedule/[path]/[action type]/[device name]/[endPoint name]/[value]
```
```
example schedule message:
topic: /schedule/house/downstairs/office/toggle/crouton-demo/backDoorLock/off 
payload: "every night at 6pm"
```

#### Action Types and Values
* *toggle*: Toggles a Simple Toggle card. Accepts true/false or on/off as values.
* *button*: Presses a Simple Button card. Does not require a value.
* *slide_to*: Moves a Simple Slider card to the value.
* *slide_above*: Moves a Simple Slider card to the value, unless the current value is higher.
* *slide_below*: Moves a Simple Slider card to the value, unless the current value is lower.

**Note**: slide_above and slide_below are not yet supported.

#### Removing Schedules
* *clear*: deletes a previously created schedule.
* *clearall*: deletes all active schedules. Should be sent to /schedule/clearall/ topic.

To delete a schedule, send a message to:
```
/schedule/[path]/ clear /[device name]/[endPoint name]/[value]
```
To delete all active schedules send a message to:
```
/schedule/clearall
```

**Note**: If you use the scheduler, you cannot use any of the action type keywords in a path, device_name, or endpoints.

#### Dashboard Interface
The schedule page is a more visual way to add, edit, or delete schedules. Currently active schedules will always be displayed above the form. You can only set one schedule per endpoint, to edit/overwrite an existing schedule, re-enter it by using the form.

If an endpoint was added earlier from the dashboard on the connections page, it's name is the card title converted to camelCase. It's good practice to name all of your endpoints in camelCase.

**Note**: The dashboard interface doesn't do anything more than format and send a message to the scheduler. 

#### Plain Language Parsing

Plain language schedules are parsed using later.js. A complete guide to text parsing can be found [here](http://bunkat.github.io/later/parsers.html#text).

#### Schedule Objects
You can also place your own JSON schedules in the message payload. Later.js has a complete guide to forming schedule objects [here](http://bunkat.github.io/later/schedules.html).

**Note**: Later.js is listed as a dependency in the bower file. You will not have to install it separately.

The scheduler connects to the MQTT broker from the information stored in the /public/common/mqtt_broker_config.json file.





	
























	

	









Clod Sketch Library
===================

The Clod Sketch Library allows users with little programming knowledge to easily customize and upload sketches to esp devices. From an interface such as the Crouton dashboard, a user can simply select the name of a sketch, enter some customizing information, and press upload. Since all sketches work with over-the-air updates, a user can re-select the esp chip and upload a new sketch at anytime. This section covers how to prepare a sketch so that it is compatible with the Clod Sketch Library, and how to add it to the library. **If you just want to use Clod, you do not need to read this section.**


Custom Sketch Protocol
----------------------

#### Why is this Necessary?

Suppose a user wants to upload a sketch that monitors and logs the temperature, we'll call it `Basic_Temp`. Per the [walkthrough](https://github.com/jakeloggins/Clod-scripts/blob/master/README.md), the sketch needs to know how to respond to `/deviceInfo/` requests to its endpoint topics. So we'll program the sketch to give the device a name of `Basic Temperature`, a single endpoint of `temperature`, and a path of `/house/`. Easy enough. 

The uploader script will happily upload this sketch to a device. It will lookup information about the esp device, navigate to the sketch's Platform IO project folder, modify the platformio.ini file, and run the Platform IO command. Everything is great.

But what happens when a user wants to monitor the temperature in 5 different locations on the same network? All five devices will be transmitting to `/house/control/basicTemperature/temperature`. The Crouton dashboard will display the temperature readings of 5 different devices as if it's a single device. 

Therefore, the protocol allows the user to specify unique device and endpoint names but keep the same basic sketch.


#### Namepins.h

The uploader script handles user customization by writing to the file `namepins.h` that is included in the main sketch file of `main.cpp`. In this way, customized information received by the uploader script may be incorporated to the sketch without repeatedly hardcoding the main file. Each time a user uploads a sketch, the `namepins.h` file is overwritten.


#### Identifcation Variables

Within `namepins.h`, the following variables are created from the device object:

* `String thisDeviceName` - User customized device name.

* `String thisDevicePath` - User customized device path.

* `String subscribe_path` - The path followed by a `#` symbol to subscribe to all required endpoints.


#### Pin Numbers

Different espressif models have different I/O pin configurations. To allow a sketch to work with all models, the uploader script reads the `board_type` from within the `espInfo` object and defines letters to pin numbers. For example, a `board_type` of `esp01_1m` would write this into the `namepins.h` file:

`#define PIN_A 2`

... while a `board_type` of `esp07` would write the following:

```
#define PIN_A 4
#define PIN_B 5
#define PIN_C 12
#define PIN_D 13
```

Therefore, a sketch can refer to PIN_A and the uploader script will decide the appropriate pin number.


#### Static Endpoint Id

To begin the upload process, a device object is sent to the uploader script. Within the device object is the endpoints object, which is the main way Clod updates values between users and devices (a detailed look at the device object is in the [walkthrough](https://github.com/jakeloggins/Clod-scripts/blob/master/README.md). Here's an example of a device's endpoints object:

```json
"endPoints": {
      "lastTimeTriggered": {
        "title": "Last Time Triggered",
        "card-type": "crouton-simple-input",
        "static_endpoint_id": "time_input",
        "values": {
          "value": "n/a"
        }
      },
      "alarmLight": {
        "title": "Alarm Light",
		"card-type": "crouton-rgb-slider",
        "static_endpoint_id": "RGB",
        "values": {
		    "red": 0,
    		"green": 0,
    		"blue": 0
  		},
		"min": 0,
		"max": 255,
      }
    }
```

Each object is an endpoint, its key is simply a camelized version of the title. `static_endpoint_id` tells the sketch *what the endpoint actually does* so that the user can ensure all device and endpoint names throughout Clod are unique without hardcoding the main sketch file.

The uploader script writes a `lookup` function into `namepins.h` that allows the sketch to associate the `endpoint_key` with the `static_endpoint_id`. Here's an example:

```arduino
String lookup(String endpoint_key) {
	if (endpoint_key == "alarmLight") {
		return "RGB";
	}
}
```

When the sketch receives a message, it should send the endpoint portion of the MQTT topic to the `lookup` function. The function will return a String, which the sketch can use to know what the endpoint is supposed to do. In the example above, the user has named an endpoint `Alarm Light`. The lookup function associates that name with `RGB`. The sketch is programmed to adjust the values of a neopixel strip whenever the lookup function sends it the string `RGB`. 


#### Platform IO File Structure

For now, all custom sketches can be found in their own folder within Crouton's sketches directory. These follow Platform IO's project folder format, which you can read about [here](http://docs.platformio.org/en/latest/ide/atom.html#setting-up-the-project). The Crouton dashboard's uploader interface looks through the sketches directory and generates a list of the project folders. Your sketch should be named main.cpp and placed in the project's `src` folder. Libraries for only your project go in the project's `lib` folder. The `.pioenvs` folder is Platform IO's cache system. Do not edit the `.pioenvs` folder, it will be periodically deleted by Platform IO and ignored within this git.  


#### Default Endpoints

A sketch should have a `default_endpoints.json` file within its project folder. This is used by the uploader script to create the endpoints object if one was not sent to it. Default endpoint keys and titles should be intuitive and short. The default endpoint file *must* have a static_endpoint_id, because uploader creates a fresh namepins.h file before each upload. Here's an example default_endpoints.json file:

```json
{
  "alarmLight": {
    "min": 0,
    "max": 255,
    "card-type": "crouton-rgb-slider",
    "static_endpoint_id": "RGB",
    "title": "Alarm Lights",
    "values": {
      "red": 0,
      "green": 0,
      "blue": 0
    }
  }
}
```

#### Custom Endpoints

Custom endpoints can be changed on the user's end by simply changing the `default_endpoints.json` file. Future releases of Clod will bring this process to the Crouton dashboard, and introduce the use of a separate `custom_endpoints.json` file.


#### Hookup Guide

The hookup guide provides an additional set of instructions to the user after they have completed the upload process. This step is particularly important for non-technical users. A `guide.md` file should be included in the sketch's project folder. If there are any specific precautions, notes, or wiring instructions that go along with the sketch, they should be placed here. When the upload process is executed within the Crouton dashboard, the user will be prompted to view this file. 


### But I don't care about sharing with other people, can I just quickly hack something together for myself?

Yes. Look at the example clients, MQTT Standard, and the walkthrough. Write a sketch that does the minimum to follow those guidelines and upload it to your device manually. Every aspect of Clod, excluding the uploader, will be fully available to your device.


### What about non-espressif devices?

Currently, the uploader is only compatible with espressif based devices. But future releases of the uploader script can, in theory, easily handle any device or platform that works with Platform IO.


Add Your Sketch to the Library
------------------------------

For now, just add your sketch folder to this git and submit a pull request. 


Automatic Sketch Library Updates
--------------------------------

Coming soon: updating the library after installation.
Coming soon: browsing and downloading individual sketches from within an easy interface.







































Scheduling
==========

The scheduler is a node.js app that can send normal MQTT commands to endpoints at specified times. It can be used through the dashboard or by sending a message to the /schedule topic.

### Command Syntax
On recepit of a message to /schedule/#, the scheduler gets information about the endpoint from the rest of the message topic, and parses the message payload to create the schedule. The example message will ask the scheduler to lock the "backDoorLock" endpoint every night at 6pm. To change it to every night at 7pm, simply send another message to the same topic with a payload of "every night at 7pm."
```
/schedule/[path]/[action type]/[device name]/[endPoint name]/[value]
```
```
example schedule message:
topic: /schedule/house/downstairs/office/toggle/crouton-demo/backDoorLock/off 
payload: "every night at 6pm"
```

### Action Types and Values
* *toggle*: Toggles a Simple Toggle card. Accepts true/false or on/off as values.
* *button*: Presses a Simple Button card. Does not require a value.
* *slide_to*: Moves a Simple Slider card to the value.
* *slide_above*: Moves a Simple Slider card to the value, unless the current value is higher.
* *slide_below*: Moves a Simple Slider card to the value, unless the current value is lower.

**Note**: slide_above and slide_below are not yet supported.

### Removing Schedules
* *clear*: deletes a previously created schedule.
* *clearall*: deletes all active schedules. Should be sent to /schedule/clearall/ topic.

To delete a schedule, send a message to:
```
/schedule/[path]/ clear /[device name]/[endPoint name]/[value]
```
To delete all active schedules send a message to:
```
/schedule/clearall
```

**Note**: If you use the scheduler, you cannot use any of the action type keywords in a path, device_name, or endpoints.

### Dashboard Interface
The schedule page is a more visual way to add, edit, or delete schedules. Currently active schedules will always be displayed above the form. You can only set one schedule per endpoint, to edit/overwrite an existing schedule, re-enter it by using the form.

If an endpoint was added earlier from the dashboard on the connections page, it's name is the card title converted to camelCase. It's good practice to name all of your endpoints in camelCase.

**Note**: The dashboard interface doesn't do anything more than format and send a message to the scheduler. 

### Plain Language Parsing

Plain language schedules are parsed using later.js. A complete guide to text parsing can be found [here](http://bunkat.github.io/later/parsers.html#text).

### Schedule Objects
You can also place your own JSON schedules in the message payload. Later.js has a complete guide to forming schedule objects [here](http://bunkat.github.io/later/schedules.html).

**Note**: Later.js is listed as a dependency in the bower file. You will not have to install it separately.

### Stored JSON files
Active schedules are stored in the /public/common/schedule_data.json file to resume normally after system restart and to allow for direct editing. You can also directly edit schedule objects in this file, but must restart the scheduler for them to take effect. 

The scheduler connects to the MQTT broker from the information stored in the /public/common/mqtt_broker_config.json file.


### Unsupported Card Types
These card types only report data and thus are not supported:
  * Line Chart
  * Donut Chart

These card types are not yet supported:
  * Simple Input
  * RGB Slider



