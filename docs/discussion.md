# Discussion regarding project purpose #
*note this is a WIP dicussion*

* [V] general aim is a bit broader than only monitoring Inverters
    * I like to come up with a small open source framework where several power related devices (Inverters, Ampere Counters, Battery management systems, Relays) can be monitored (Grafana) and orchestrated (Home assistant).
    * There are two goals:
        * to optimize the configuration of the power grid dynamically. Your e-vehicle comes to the wall box for loading. If you can afford to have a multi phase wall box you are happy. If not the wall box typically only draws from a single phase and you are using only 1/3 of your current solar power for loading your car. So we have figured out a dynamic configuration working with an additional fourth inverter to bundle 2/3 of the solar capacity into one phase to the wall box.
        * to prevent battery overloading. We are dealing with lithium ion batteries and it is quite complicated to just load them for 70% of their capacity (extending their lifetime tremendously) since the loading voltage is not linear with the load in this regime. So the idea is to measure all the currents in the power system and feed them into a physical simulation of the battery to derive the current battery loading.

* [C] Would be great to control the inverter mode via SOC from a battery management system
* [C] Has some python code reading a BMS but is just there for info at the moment. Mode is controlled internally via voltage in the Axpert MKS II which is not ideal
* [V] begun writing a registry based plugin system for components.

* [V] It's the general picture. In the end we all want quite similar (measure values and reacting on them ) stuff, but with probably very different algorithms, hardware, node number, node characteristic etc. Therefore I suggest to utilize existing frameworks in which we integrate our individual hardware ('plugins').
* [V] The current framework stack I am tinkering with is roughly:
    * Node red : High level management (not tried)
    * Homeassistant (HASS) : Management framework. Representation of the Components and Sensors as HASS objects. (first ecperiance)
    * MQTT : transport layer (running)
    * Ethernet/WLAN/(CAN-Bus) : physical transport layer (common knowledge)
    * Node : Node hardware z.B: Raspberry Pi
    * Registry/Plugins: for sensors/components (first prototype but nothing stable)
    * Sensors: GPIOs direct attached sensors, (I2C, serial, USB)- connected components.
    * Component(s) : Inverters, BMS, Current counters, High power Relays, ... (Inverter mpp-solar, BMS (own code) , Counter (own code))

* CAN-Bus can be a great alternative to ethernet/WLAN. I work with CAN-Bus in heavy duty vehicles and there it is quite robust against EMP and other stress factors. Also CAN-Bus is only two wires has a wire limit of 1km.

* [V] For the sensors/components we need a common interface specification/transport protocol. This may be
quite simple but structured well. A good start could be a request/response scheme with python dictionaries serialized as JSON.
```request:
{
  request_id: 1213455,
  time: timestamp,
  node: node_id (e.g. raspberry2 ),
  comp: comp_id (e.g. inverter3),
  command : command (e.g. QID),
  parameters : {
      e.g. for setting something to a defined value
      }
}

response:
{
  request_id: 1213455,
  time: timestamp,
  node: node_id (e.g. raspberry2 ),
  comp: comp_id (e.g. inverter3),
  command : command (e.g. QID)
  parameters : {
    e.g. for setting something to a defined value
    },
  values: {
    incoming_current: {
      value : 1200,
      unit: 'mA'
    }
  }
}
```


* [J] I suggest a flexible approach, where the components could be interchanged reasonably easily (this sounds easy, but changes some design choices). This probably means we need some sort of message definition
    * mpp-solar inverters have the 'fun' aspect that a query command provides multiple pieces of information...
    * lets not forget the various cloud services that may be of use to some
* [J] It feels like we have a bunch of layers (again random thoughts)
    * aa. 'measurement type' device (inverter, bms, sensor, etc), this layer can provide a measurement
    * ab. 'action type' device (inverter, bms relay etc), this layer can do something (may be same device as previous)
    * b. communications layer (physical connectivity)
    * c. message queue / bus (like MQTT)
    * d. display (UI component of Home Assistant or Grafana etc)
    * e. orchestration / rules engine (Home Assistant rules, node red or others)
    * f. something that talks to a and puts messages into c via b??
    * g. persistence / storage
    * h. (maybe) management interface

* Many of the above exist - point f. is probably where I see the lack (and for my scenario there will be multiple 'brains' (i.e the inverters are in a shed up the hill are connected to a Pi, but other devices wil be elsewhere and connected to different devices)
* The correct routing of information will one of the many crucial tasks. My picture was only the 10000 feet view.
We will have to cover installations where we have to deal with many nodes (e.g. Raspbarries) dealing with many devices (Inverters, etc.).
* I guess in my 'ideal world' we'd be aiming at something that fixed the gap without re-inventing stuff that already exists. But also not force people to use a specific piece (you might use Node-Red, I might want to use python)

* [J] Of course, it is possible that you achieve most of the above simply by creating an integration to Home Assistant....
    * Here I like to disagree.
    * HASS is not really designed for dealing with Devices that have several or complex internal states. So it would be not easy to model an Inverter that consists out of 3-4 sub-Inverters with HASS. I have not tried but the dev documentation on this topic is sparse. So we have to test if HASS supports this or we can make it to support it.
      * Well, the integrations are python based, so 'can make it do whatever we want' to a certain extent. It is certainly easy to publish a value to MQTT then use HASS to see that as a sensor. I think the question would be 'is the rules engine in HASS sufficient'? followed by where is the effort 'modelling' the complex state best spent

## Questions ##
So there has to be an authority to steer the flow of current.
In the current discussion we all agreed in giving HAAS the total control.
Do we really trust HASS to control the main power supply of our houses?

Or do we like to have some checks and balances beside HASS?
* Ah, well security is probably a good thing to consider early on...
    * Not only security. If we delegate our control to algorithms how can we prevent our homes to get stuck? So we need fallback mechanisms. E.g. to deal with the HASS going down.
    * Yes - I mean security in the wider sense, how to stop unauthorised access / actions is one aspect, fail-safes and stopping stupid requests are a few others



## Use Cases ##
### UC001 ###
* 50 qm cells. Large battery lithium ion based. Four inverters. Electric car.
* Car is at work on 5 of 7 days. Car returns 16:00.
* When at home a Button is pressed beside the wallbox indicating that the car has to be loaded.
* The solar stripes then are reconfigured to put 2/3 of their power into one inverter phase to power the wallbox with 8 kW.
* This is a good strategy if there is sun. But if there is no sun, the battery is the next best choice. And if the battery is low it may be a good choice to power the car from the grid.

### UC002 ###
Goal would be to integrate different energy storage device instead of the car.
Electric hot water heater inline with gas water heather already present in the home. During winter when sun is not as abundant as it is in summer use time of use tariff with power company to supplement power needs (charge batteries to certain level) which would help lower down the energy cost.

What I have:
* 53 KWh of LiFePo4 cells equally divided into 3 power walls.
* 3 LV5048 inverter/chargers
* 9 KW of solar panels
* 3 Dumb BMS's. They limit charge/discharge at 120A and monitor cell voltage. If cell voltage is above or below limits they stop everything.
* 3 Balancers/monitors from JKBMS for purpose of monitoring cell health.

Since Lithium chemistry has almost flat discharge curve at 0.2 C rate it is impossible to tell SOC by just looking at cell/power wall voltage. Reasonably accurate SOC device must be used in order to implement what is outlined in WIP discussion.

Reasonably accurate SOC device must take in to account Voltage, Amperage and Temp from power wall according to documentation I have read.

So far I have:
* Volts provided by JKBMS
* Amps provided by ADS1115
* Temp provided by DHT22

All going to a Raspberry Pi
