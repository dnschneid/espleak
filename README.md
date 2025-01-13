# espleak

Whole-home leak detection and mitigation with ESPHome and magnetometers!

Inspired by https://github.com/tronikos/esphome-magnetometer-water-gas-meter/

Features:
 * Uses two QMC5883L sensors attached to your water meter to precisely monitor and report flow
 * Configurable to detect multiple thresholds of abnormal flow
 * Configurable to react to abnormal sensor behavior indicative of severely abnormal flow
 * Automatically signals an external valve to shut off water
 * Protection functionality operates entirely standalone
 * Automatic calibration of sensors
 * Sensor resiliency features for reliable long-term operation


## How do I use this?

See the [example](example) directory for inspiration on installation. Note that for performance, espleak *only* uses the
Z axis of the magnetometers.  That means it's critical to have the QMC5883L installed flat against the meter (which is
the easiest way to install it anyway).

Once configured, the ESP32 operates autonomously. The on-board LED glows brighter for higher flows. If abnormal flow is
detected, the LED will blink and the GPIO output will signal to a connected water controller (or valve actuator) to shut
off water. Pressing the on-board button (or using the API) will reset any error conditions and allow the water to be
turned on again.

Several functional entities are exposed via the API:
 * The current flow rate
 * The accumulated total water
 * Whether a threshold is being approached (leak imminent)
 * Whether a threshold has been exceeded (leak detected)
 * A button to clear detected leaks and allow water to be turned on again

In addition, there are several diagnostic entities:
 * The monitor that triggered the leak mitigation (e.g., which threshold was exceeded)
 * A running count of sensor errors detected and mitigated
 * The current sample rate of the sensors
 * A status indicator for sensor failure, which may require recalibration or physical attention to resolve
 * A button to trigger recalibration

Finally, there are a few hidden entities that can be enabled as needed:
 * The on-board button status
 * A button to restart the sensors (normally this is done automatically upon sensor error)
 * A number input to simulate additional flow, for testing thresholds

espleak will self-calibrate when first installed, and whenever the recalibrate button is pressed. Ensure no water is
flowing when calibration is triggered. After 30 seconds you can run a small stream of water on a sink or flush a toilet
to complete calibration (or don't; calibration will advance whenever water next flows).


## How does it work?

The two sensors are installed at a 90-degree offset. Together they form a quadrature encoder for the spinning magnet
connected to the water sensor's nutator.  This nutator spins rapidly, signalling a tiny fraction of a gallon each time
it spins.

The ESP32 polls these sensors as quickly as possible. At calibration time, the detected sensor noise levels are used to
set the hysteresis level, and the accumulated min/max data from the sensors sets the threshold for positive and negative
state. The states of the two sensors are then quadrature decoded to detect forward and backwards spinning of the magnet.
The spins are counted over time to interpret the flow rate. This, along with the accumulated rotations, are exposed via
API as metrics.

On the ESP32, the flows are also compared against a set of thresholds. Exceeding the thresholds will advance a counter,
and once the allowed excursion time has elapsed, the ESP32 will assert a GPIO that signals an external water controller
(probably a valve actuator) to shut water off. The event is logged to the API and must be cleared either via a physical
button or via the API before the water controller is allowed to re-enable water.

In addition, if flow rate is so high that the rotations exceed the sample rate of the sensors, heuristics detect the
abnormal sensor data and will trigger the same mitigation mechanism.

See the yaml comments for more details on configuring the monitors.

To help with resiliency, the ESP32 can power cycle and reconfigure the sensors to recover from bus or sensor errors. At
the moment, this requires modified versions of the `i2c_bus` and `qmc5883l` components, which are hosted at
[dnschneid/esphome@i2creset](https://github.com/dnschneid/esphome/tree/i2creset) and pulled in by `espleak.yaml`.


## How do I test it?

Test the flow reporting by filling up a bucket in a fixture and timing how long it takes to fill up. The reported flow
during the fill should equal the volume of the bucket divided by the time it took to fill.

Test the flow-based leak response by enabling the flow simulator entity and increasing the flow until it is just below
the threshold of one of your entries. Then, turn on a fixture to push the flow over the limit and confirm the system
reacts after time has elapsed. You can shut off the fixture before time has elapsed to confirm that the timing resets.
Hitting the "clear leak" button will automatically zero out the flow simulator.

Testing the burst heuristics is more involved. The easiest way is to target a burst flow rate that can be quickly
achieved by something you control -- be it a garden hose nozzle, a washing machine, or even an irrigation system. Then,
increase the `sensor_update_interval_ms` setting until the flow rate is outside the measurable range of the sensors (see
the `flow_max` calculation). Once you've installed the updated image, you should be able to trigger the burst heuristics
by suddenly activating your flow source. You can try different update intervals to push the sensing into different
regions of operation. Remember to revert the update interval when you're done testing, otherwise your next shower will
be extremely short.


## Caveats?

Due to the 200Hz maximum sample rate, espleak may not suffice for larger homes. The exact thresholds are dependent on
what meter you have. For the Badger 25, for example, if you anticipate having >7.5 gallons/minute of normal water usage,
some of the detection features are diminished. Extreme flow mitigation won't be nearly as reliable if you expect >15
gallons/minute, and the system can't even correctly report flows above 30 gallons/minute, although it can still react to
abnormal flows beyond that. For a Badger 25, 2~3 baths is a reasonable limit. Larger homes may have larger meters,
though, so you should do the math.

You should understand how the system works and convince yourself that you have installed and configured all of the
requisite hardware and software correctly before relying on it to protect your property. You should test its behavior
regularly to confirm that the system is active and working, adjusting sensor positions or recalibrating as necessary.
You should regularly check that the water controller is functional and able to shut off water.

This software is provided as-is, without warranty of any kind. The author is not liable for any damages or inconveniences caused by the function or non-function of this software. See
LICENSE for more disclaimers.
