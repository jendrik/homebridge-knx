# General structure
homebridge-knx expects the following structure in the `knx_config.json`:

```
{
    "knxconnection": "knxjs",
    "knxd_ip": "192.168.1.1",
    "knxd_port": 6720,
    "AllowWebserver": true,
    "AllowKillHomebridge": false,
    "Devices": [
    ]
}
```
`knxconnection` if set to "knxjs" then [javascript based knx multicast](https://www.npmjs.com/package/knx) routing is used, set to any other value or non-existent will default to knxd  
`192.168.1.1` has to be changed to your [knxd](https://github.com/knxd/knxd) installation, and `6720` to the port you are using (6720 is default).  
`platform` and `name` have become obsolete.  
`AllowWebserver`: if set to true, homebridge-knx will start a tiny webserver on port 18081 to allow the removal of devices from homebridge cache (and the running instance). Enable for debugging, disable for secure productive operation! 
`AllowKillHomebridge`: if set to true, you'll get a link at the tiny web browser allowing to force a shutdown of homebridge by throwing an exception. You should reset that to false after debugging, because it allows anyone with access to the webserver to shut down your homebridge instance. 

`Devices` is an array of objects, each representing an *homekit accessory*. 

## Devices
`Devices` must have a unique `DeviceName`. The name cannot be changed in the config file after pairing with homekit, because homekit *copies* it into its database. You can change the name of the device in homekit afterwards; unless you erase your home or un-pair homebridge it will keep the changed name.  
In Apple's `Home` app (or other homekit enabled apps) devices can get assigned to rooms.  
As HomeKit can only handle 100 accessories (devices) per bridge, and homebridge bundles them all into one, it is recommended to bundle all functionality into one device per room. It is also less work to do when assigning the services to the rooms, as all services of one device are placed in one room automatically.   

Each device needs a section `Services`. Here all functionality of the device is defined.

## Services
`Services` again is an array of objects, each representing a *homekit service*.  
Each service **must** have:
- a unique `ServiceName` (regarding changing names in a paired instance [see Devices](#devices) )
- a known *homekit* `ServiceType`
- a `Characteristics` section, containing all the variables that homekit can handle.    
A service **can** have:  
- an `handler` entry, pointing to an add-in to handle all requests for the service.  
- a `KNXObjects` array containing KNX group addresses used by the handler and not directly connected to a characteristic.  
- a `KNXReadRequests` array containing group addresses that should be read upon startup of homebridge, to sync the status of characteristics with the physical devices.   

Service types are regularly amended by Apple, to get new service types into homebridge-knx we need to update homebridge using npm.
There is a Javascript file containing all valid services and characteristics located here: https://github.com/KhaosT/HAP-NodeJS/blob/master/lib/gen/HomeKitTypes.js

 
## Characteristics
Each characteristic is an object containing **at least** a `Type` field. Types can be "On" for power, "Brightness" for dimmable lights etc.  
Characteristics **can** have assigned group addresses in the `Set` and `Listen` arrays.
For the group addresses it is possible to define a *data point type* `DPT` (currently valid are DPT1, DPT5, DPT5.001, DPT9). Characteristics without group addresses can only be used by a `handler`.
  
For **boolean and percentage types** it is possible to *reverse* the read/write value between HomeKit and KNX (Restrictions: Only if it is a boolean or a percentage for **HomeKit**, some of the types that support only 0 and 1 are integer anyway; Service must not have a handler, see chapter below).

For **enumerative type**, such as TargetHeatingCoolingState with a given list of possible values, it is possible to restrict **Apple's Home** app even more with the "ValidValues" array keyword. Only working with **new** services. If the service is read into homekit database, this is stored there and not read again. If you need to change tha valid values, you have to remove the service from homekit and re-add it again.

For **continuous value types**, such as temperatures, the minimum and maximum values that define the range can be modfied with the "MaxValue" and "MinValue" keywords. Required for Temperatures in Celsius in middle/norther Europe at least, as Apple (designed in California, made in China) expected no sub-zero temperatures.     

Some exmples:
```json
"Characteristics": [
    {
        "Type": "On",
        "Set": [
            "1/3/5"
        ],
        "Listen": [
            "1/3/5"
        ]
    }
]
```

with **"Reverse"**
```json
"Characteristics": [
    {
        "Type": "On",
        "Set": [
            "1/3/5"
        ],
        "Listen": [
            "1/3/5"
        ],
        "Reverse": true
    }
]
```

With **"ValidValues"**, to remove "Cool" from the list of Target States:
```json
{
    "Type": "TargetHeatingCoolingState",
    "Listen": [
        "14/1/97"
    ],
	"ValidValues":[0,1,3]
}
```                      

Modifying **Min and Max**: 
```
{
    "Type": "CurrentTemperature",
    "Listen": [
        "7/1/11" 
    ],
    "MaxValue": 60,
    "MinValue": -20
}
```

## Handler
New in version 0.3 of homebridge-knx is a little add-in concept, allowing additional functionality to be added without changing the big mass of the code.  
`handler`s are defined as javascript files in `/lib/addins` and need to [follow some restrictions.](https://github.com/snowdd1/homebridge-knx/blob/plugin-2.0/handler-add-in.md)  
To assign a handler to a service the **Handler** keyword is used, see example below.  
Handlers cannot use the `Reverse` keyword for DPT1 and DPT5.001 types, this has to be taken care in the handler's programming.  

## KNXObjects
Handlers can make use of KNX group addresses that are not connected directly to characteristics. To allow references to those addresses, they are defined as if they were characteristics. 
- The `Type` field is a freely definable name field (uniqueness within the service required). 
- Definition of `DPT` **is mandatory** as the data point type cannot be inferred from a homekit characteristic!  

## KNXReadRequests
All addresses listed in that array are sent to the bus with a READ request telegram at the start of homebridge-knx. They are not ergularly polled. Use active status objects instead.## LocalConstants
Handlers can use service-local constants in their code. This allows using the same handler for alike-but-not-equal use cases. The values from the `LocalConstants` can be used in the handler code. This allows re-using the same handler for multiple objects that differ by more than group addresses.

## UUID and subtype
homebridge-knx creates a unique UUID for each device newly discovered in the *knx_config.json* and writes that back to the file. Similar, a unique `subtype` field is created for each service.  
**Do not alter these fields** unless you want to force homebridge-knx to accept this as a **new** device or service, rendering the old one stale and unreachable.

# Example
```json
{
    "knxd_ip": "192.168.178.100",
    "knxd_port": 6720,
    "AllowKillHomebridge": true,
    "Devices": [
        {
            "DeviceName": "Office",
            "Services": [
                {
                    "ServiceType": "Lightbulb",
                    "ServiceName": "Office desk lamp",
                    "Characteristics": [
                        {
                            "Type": "On",
                            "Set": [
                                "1/3/5"
                            ],
                            "Listen": [
                                "1/3/5"
                            ]
                        }
                    ]
                },
                {
                    "ServiceType": "WindowCovering",
                    "Handler": "GiraJalousieActuator",
                    "ServiceName": "Office blinds",
                    "Characteristics": [
                        {
                            "Type": "TargetPosition",
                            "Set": [
                                "2/3/46"
                            ],
                            "DPT": "DPT5"
                        },
                        {
                            "Type": "CurrentPosition",
                            "Listen": [
                                "2/3/26"
                            ]
                        },
                        {
                            "Type": "PositionState"
                        }
                    ],
                    "KNXObjects": [
                        {
                            "Type": "ShutterMove",
                            "Listen": "2/3/6",
                            "DPT": "DPT1"
                        }
                    ],
                    "KNXReadRequests": [
                        "2/3/26",
                        "2/3/46"
                    ],
                    "LocalConstants": {
                        "TimeOutSecs": 23
                    }
                }
            ]
        }
    ]
}
```
