---
title: Device Description Files
date: 2021-07-11

---

# Device Description Files

This is a draft document to explore Device Description Files (DDF).

<mark>Work in Progress</mark>

### Goals

* Describe devices and their capabilities in JSON files.
* Simplify contribution of new device handlers without C++ knowledge.
* Cleanup many of the C++ device specific quirks to simplify the code base.
* Smooth transition: DDF files won't replace existing implementations in one big release, but rather will be used for a device when available.
* Automated generation of device documentation and supported devices lists.

### Handling in other projects

**SmartThings** Device handlers are implemented in *.groovy* files which may contain one or more device descriptions, e.g. [xiaomi-aqara-button.groovy](https://github.com/bspranger/Xiaomi/blob/master/devicetypes/bspranger/xiaomi-aqara-button.src/xiaomi-aqara-button.groovy).

**Zigbee2MQTT** Used one large devices.js file in the past. New code uses modular device specific converters, one per manufacturer, written in Javascript, e.g. [xiaomi.js](https://github.com/Koenkk/zigbee-herdsman-converters/blob/master/devices/xiaomi.js). Similar devices inherit common functions from base files.

<https://www.zigbee2mqtt.io/how_tos/how_to_support_new_devices.html>



**Home Assistant ZHA** Device handlers are maintained in the zigpy project written in Python.

<https://github.com/zigpy/zha-device-handlers>

### Directory structure

Since the amount of devices is expected to grow quickly, over time there will be hundreds of device descriptions. In order to be maintainable they need to be organized with the aim to prevent "monster" files. The current implementation uses one directory per vendor and one file per product:

```
devices/
    generic/
        items/
    ikea/
        motion-sensor.json
        button5-remote1.json
        button5-remote2.json
        e27_cws_opal_600lm_light.json
        gu10_ws_400lm_light.json
        kadrilj_blind.json
    philips/
        sml001_motion_sensor.json
        dimmer_switch.json
    sunricher/
        dimmer.json
```

This is similar to SmartThings device descriptions. The REST-API plugin loads DDF files by traversing through the directory tree, and for easier development can even hot reload them when a file changes.
// Explain how to handle re-branded devices? (I missed this when making this DDF https://github.com/dresden-elektronik/deconz-rest-plugin/pull/8076)
// Example: Namron devices which are re-branded Sunricher devices.
// Should we make dedicated folders or piggyback on the first one to the party?

## DDF file content

A DDF file provides a high level view and contains references to where data is located as well as the commands which can be called. In contrast to other projects most of the implementation details are not exposed.

**Example** *IKEA TRÅDFRI bulb GU10 WS 400lm*

```json
{
    "schema": "devcap1.schema.json",
    "doc:path": "ikea/gu10_ws_400lm_light.md",
    "doc:hdr": "TRÅDFRI bulb GU10 WS 400lm",
    "manufacturername": "$MF_IKEA",
    "modelid": "TRADFRI bulb GU10 WS 400lm",
    "product": "TRÅDFRI bulb GU10 WS 400lm",
    "status": "Bronze",
    "md:known_issues": [ "ikea_known_issues_radio_silence.md" ],
    "supportsMgmtBind": true,
    "subdevices": [
        {
            "type": "$TYPE_COLOR_TEMPERATURE_LIGHT",
            "restapi": "/lights",
            "uuid": ["$address.ext", "0x01"],
            "items": [
                { "name": "attr/lastannounced" },
                { "name": "attr/lastseen" },
                { "name": "attr/manufacturername" },
                { "name": "attr/modelid" },
                { "name": "attr/name" },
                { "name": "attr/swversion" },
                { "name": "attr/type" },
                { "name": "attr/uniqueid" },
                { "name": "config/colorcapabilities", "default": 16 },
                { "name": "config/ctmin", "default": 250 },
                { "name": "config/ctmax", "default": 454 },
                { "name": "state/on" },
                { "name": "state/bri"},
                { "name": "state/ct" },
                { "name": "state/reachable" },
                { "name": "state/colormode" },
                { "name": "state/alert" }
            ]
        }
    ],
    "bindings":  [
        {
            "bind": "unicast",
            "src.ep": 1,
            "cl": "0x0006",
            "report": [
                {"at": "0x0000", "dt": "0x10", "min": 5, "max": 1800 }
            ]
        },
        {
            "bind": "unicast",
            "src.ep": 1,
            "cl": "0x0008",
            "report": [
                {"at": "0x0000", "dt": "0x20", "min": 5, "max": 1800, "change": "0x01" }
            ]
        },
        {
            "bind": "unicast",
            "src.ep": 1,
            "cl": "0x0300",
            "report": [
                {"at": "0x0008", "dt": "0x30", "min": 1, "max": 1800 },
                {"at": "0x0003", "dt": "0x21", "min": 5, "max": 1795, "change": "0x0a" },
                {"at": "0x0004", "dt": "0x21", "min": 5, "max": 1795, "change": "0x0a" },
                {"at": "0x0007", "dt": "0x21", "min": 5, "max": 1800, "change": "0x01" }
            ]
        }
    ]
}
```

This DDF file describes the light with one sub-resource.

1. The **schema** key references the [JSON schema](https://json-schema.org/learn/getting-started-step-by-step.html) against which the device can be verified.
2. String values which are prefixed with `$`, like `"$MF_IKEA"` are constants and will be replaced with the actual values in the plugin. These keys might be defined in another JSON file or come from the ZCLDB.
3. The `uuid` field specifies the parts of which the `uniqueId` will be generated.
4. The `subdevices` array may hold one or more sub devices like lights and sensors.
5. The `items` array contains items where the `name` refers to a `ResourceItem` suffix. These items are merged with the content of `generic/items/<name>.json`.
6. The `bindings` array contains all bindings and respective reporting configuration.

### Generic items

Each item may have read, write and parse functions to build the bridge between Zigbee and the `ResourceItem`. The generic items provide defaults for these functions, for example [generic/items/config_checkin_item.json](https://github.com/manup/deconz-rest-plugin/blob/device_descriptions/devices/generic/items/config_checkin_item.json) is written as:

```json
{
    "schema": "resourceitem1.schema.json",
    "id": "config/checkin",
    "datatype": "UInt32",
    "access": "RW",
    "public": false,
    "range": [0, 4294967295],
    "description": "Configures the check-in interval for the Poll Control cluster.",
    "parse": {"fn": "zcl", "ep": 0, "cl": "0x0020", "at": "0x0000", "eval": "Item.val = Attr.val"},
    "read": {"fn": "zcl", "ep": 0, "cl": "0x0020", "at": "0x0000"},
    "write": {"fn": "zcl", "ep": 0, "cl": "0x0020", "at": "0x0000", "dt": "0x23", "eval": "Item.val"},
    "refresh.interval": 3600
}
```

If a DDF file contains the item `{"name": "config/checkin"}` the above content will be loaded. Every key can be overwritten in the DDF file if device specific customizations are needed.

### Parsing Zigbee messages

The `parse` key references a named C++ function which can be specified to parse and transform an incoming APSDE-DATA.indication and derive the `ResourceItem` value from it. Often the generic parse function is sufficient for example the item `state/bri` is parameterized as:

```json
{
    "name": "state/bri",
    "parse": {
        "fn": "zcl",
        "ep": 1, "cl": "0x0008", "at": "0x0000",
        "eval": "Item.val = Attr.val * 254 / 100"
    }
}
```

which means:

* Use the named C++ parse function `"zcl"`;  
  (Note the `"fn": "zcl"` key is optional since it's the default if ommited)
* When a ZCL Read Attributes Response or ZCL Attribute Report command is received on
  endpoint 1, cluster 0x0008, containing the attribute 0x0000;
* Parse the value and evaluate the expression `Item.val = Attr.val * 254 / 100` where `Item` and `Attr` are the `ResourceItem` and `deCONZ::ZclAttribute` exposed to the Javascript engine. The result will be the new value of the `ResourceItem` and related events are fired automatically.

**Note:** Javascript expressions are evaluated via `QJSEngine`.

#### The function pointer "fn"

The `"fn"` key is like a function pointer in C++ which points to a specific read, write or parse function. The default `{"parse": { "fn": "zcl", ... }}` function can be applied in many cases but specialized functions for other cases like `{"parse": { "fn": "xiaomi:special", ... }}` can be implemented to simplify certain tasks.

The remaining key-value pairs in the `"parse"` object are interpretet within the specific parse function. Note that alternative functions can use different parameters and some parameters like `"mfcode"` are optional and have default values.

#### Evaluating Javascript

The `"eval"` key in the `"zcl"` functions is the most basic yet powerful approach to support reading writing and parsing of arbitrary Zigbee messages without having to implement any C++ code for yet another custom Zigbee device.


#### Without Javascript

Since read, write and parse functions are just abstract function pointers it's always possible to implement a tailored named C++ function which can be referenced via the `"fn"` key, for example common functions like `{"fn": "parse.zcl.battery"}` can be added which don't use Javascript at all.

The C++ code for currently implemended functions can be found in [device_access_fn.cpp](https://github.com/manup/deconz-rest-plugin/blob/device_descriptions/device_access_fn.cpp).


### Commands

<mark>Not Implemented yet</mark>

Commands reference C++ functions which will be called via the REST-API. Similar to parsing ZCL values they can transform the values coming from the REST-API into values which fit the specific device, for example in order to reverse the lift value.

For some devices the standard functions don't fit and alternate commands can be specified to carry out the same task. For example the Xiaomi curtain motor uses non standard interface to change the `state/lift` value, here a

 `{"cmd": "lift", "fn": "xiaomiWindowCoveringLift", "ep": 1, "cl": "0x0102", "at": "0x0008", "eval": "100 - Item.val"}` could be specified.

For convenience simpler functions can be provided, for example the standard window covering lift function can be specified as:

`{"cmd": "lift", "fn": "windowCoveringLift", "eval": "Item.val"}` since often the {Endpoint, ClusterId, AttributeId} parameters can figured out automatically. In the example above the `windowCoveringStop` is such a case. The goal should be to figure out functions which need as little parameters as possible to keep the device description files as simple as possible.

### Button maps

For switches `state/buttonevent` refers to device specific clusters and attributes. The JSON button maps <https://github.com/dresden-elektronik/deconz-rest-plugin/pull/3127> describe the transformations from Zigbee &rarr; `state/buttonevent`. These button maps could be part of the device capabilities file itself to keep everything together.

### Validation

Each device description can be validated against a JSON schema to prevent errors. Like it's already done for Button Maps the verification can be automated via GitHub actions for pull request. The JSON schema further provides a way to specify versions of supported schemas.


## Example DDF files

The following examples demonstrate notable features in already working DDF files. Note these DDF files deliberately use Javascript *eval* capabilities to show what is possible without writing C++ code.

The auto generated documentation can be found at:  
<https://dresden-elektronik.github.io/deconz-rest-doc/devices>

#### IKEA KADRILJ roller blind

Source: [devices/ikea/kadrilj_blind.json](https://github.com/manup/deconz-rest-plugin/blob/device_descriptions/devices/ikea/kadrilj_blind.json)

* Two sub-resources under /lights and /sensors
* Customized parse function parameters
* Some items marked as deprecated
* Binding and reporting configuration

#### Philips Hue motion sensor

Source: [devices/philips/sml001_motion_sensor.json](https://github.com/manup/deconz-rest-plugin/blob/device_descriptions/devices/philips/sml001_motion_sensor.json)

* Three /sensors sub-resources
* Manufacturer specific ZCL attribute parsing
* External Javascript to parse `state/lightlevel`
* Binding and reporting configuration

#### Philips Hue dimmer switch

Source: [devices/philips/rwl02_dimmer_switch.json](https://github.com/manup/deconz-rest-plugin/blob/device_descriptions/devices/philips/rwl02_dimmer_switch.json)

* One /sensors sub-resource
* Manufacturer specific ZCL attribute reading, writing and parsing
* Binding and reporting configuration

<!--
#### LifeControl MCLH-08 environment sensor

This sensor transports values for carbon monoxide and humidity in non standard ways within the temperature cluster.

```json

{
    "schema": "devcap1.schema.json",
    "manufacturer": "$MF_NEXTURN",
    "modelid": "VOC_Sensor",
    "product": "MCLH-08 environment sensor",
    "reporting": [
        ["report/5", 1, "0x0402", "0x0000", 1, 1800],
        ["report/5", 1, "0x0402", "0x0001", 1, 1800],
        ["report/5", 1, "0x0402", "0x0002", 1, 1800],
        ["report/5", 1, "0x0001", "0x0021", 5, 3600]
    ],
    "subdevices": [
        {
            "type": "$TYPE_TEMPERATURE_SENSOR",
            "uuid": [ "$address.ext", "01", "0402"],
            "items": [
                {
                    "name": "state/temperature",
                    "parse": ["parseGenericAttr/4", 1, "0x0402", "0x0000", "$raw + $config/offset"],
                },
                {
                    "name": "config/offset",
                    "default": 0,
                },
                {
                    "name": "state/battery",
                    "parse": ["parseGenericAttr/4", 1, "0x0001", "0x0021", "$raw"]
                }
            ]
        },
        {
            "type": "$TYPE_HUMIDITY_SENSOR",
            "uuid": [ "$address.ext", "01", "0405"],
            "items": [
                {
                    "name": "state/humidity",
                    "parse": ["parseGenericAttr/4", 1, "0x0402", "0x0001", "Math.min(Math.max(0, $raw + $config/offset), 10000)"],
                },
                {
                    "name": "config/offset",
                    "default": 0,
                }
            ],                
        },
        {
            "type": "$TYPE_CARBON_MONOXIDE_SENSOR",
            "uuid": [ "$address.ext", "01", "0500"],
            "items": [
                {
                    "name": "state/carbonmonoxide",
                    "parse": ["parseGenericAttr/4", 1, "0x0402", "0x0002", "$raw"],
                }
            ]
        }
    ]
}

```
-->

#### Xiaomi vibration sensor DJT11LM

Source: [devices/xiaomi/aq1_vibration_sensor.json](https://github.com/manup/deconz-rest-plugin/blob/device_descriptions/devices/xiaomi/aq1_vibration_sensor.json)

* One /sensors sub-resource
* Manufacturer specific ZCL attribute reading, writing and parsing
* Customized `attr/swversion` parsing
* Using custom `{"fn":"xiaomi:special"}` parse function
* External Javascript to parse `config/battery` which can be shared with other Xiaomi DDF files
* External Javascript to parse `state/orientation`
