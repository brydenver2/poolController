# MQTT Interface Guide

## Overview

The nodejs-poolController MQTT interface provides bidirectional communication with pool equipment through an MQTT broker. This guide covers integration with EMQX and other MQTT brokers, including all available topics for publishing and subscribing.

## Table of Contents

- [Configuration](#configuration)
- [Connection](#connection)
- [Published Topics (State Updates)](#published-topics-state-updates)
- [Subscription Topics (Commands)](#subscription-topics-commands)
- [EMQX Integration](#emqx-integration)
- [Data Logging](#data-logging)
- [Examples](#examples)

## Configuration

### Basic Configuration

Add an MQTT interface to your `config.json`:

```json
{
  "web": {
    "interfaces": [
      {
        "type": "mqtt",
        "name": "MQTT",
        "enabled": true,
        "fileName": "mqtt.json",
        "context": {
          "options": {
            "protocol": "mqtt://",
            "host": "localhost",
            "port": 1883,
            "username": "your_username",
            "password": "your_password",
            "rootTopic": "pool",
            "retain": true,
            "qos": 2
          }
        }
      }
    ]
  }
}
```

### Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `protocol` | MQTT protocol (`mqtt://`, `mqtts://`, `ws://`, `wss://`) | `mqtt://` |
| `host` | MQTT broker hostname or IP | `localhost` |
| `port` | MQTT broker port | `1883` |
| `username` | MQTT username (if authentication enabled) | - |
| `password` | MQTT password (if authentication enabled) | - |
| `rootTopic` | Root topic prefix for all messages | - |
| `retain` | Retain messages on broker | `true` |
| `qos` | Quality of Service level (0, 1, or 2) | `2` |
| `clientId` | MQTT client identifier | Auto-generated |
| `selfSignedCertificate` | Accept self-signed certificates for TLS | `false` |
| `changesOnly` | Only publish when values change | `true` |

### Root Topic

The root topic can be dynamically generated using bindings:

```json
"rootTopic": "@bind=(state.equipment.alias).replace(' ','-').replace('/','').toLowerCase();"
```

Or set as a static value:

```json
"rootTopic": "pool"
```

All published and subscribed topics will be prefixed with this root topic.

## Connection

### Last Will and Testament (LWT)

Configure LWT to notify clients when connection is lost:

```json
"willTopic": "state/status",
"willPayload": "{\"val\":3,\"name\":\"lost\",\"desc\":\"Connection Lost\",\"percent\":0}"
```

### Connection Status

The controller publishes its status on connect:

**Topic:** `{rootTopic}/state/status`

**Payload:**
```json
{
  "val": 1,
  "name": "ready",
  "desc": "Ready",
  "percent": 100
}
```

## Published Topics (State Updates)

All state changes are published to MQTT. Topics follow the pattern: `{rootTopic}/{category}/{id}/{name}/{property}`

### Controller State

**Topic:** `{rootTopic}/state/controller`

Published on controller state changes, includes time, status, mode, and equipment information.

**Topic:** `{rootTopic}/state/time`

Published every minute when controller is ready.

**Payload:** ISO 8601 timestamp

**Topic:** `{rootTopic}/state/mode`

Published when panel mode changes (pool, spa, etc.).

### Circuits

**Topic:** `{rootTopic}/state/circuits/{id}/{name}`

**Payload:**
```json
{
  "id": 1,
  "isOn": "on",
  "endTime": "2024-01-15T14:30:00"
}
```

**Properties Available:**
- `{rootTopic}/state/circuits/{id}/{name}/isOn/string` - "on" or "off"
- `{rootTopic}/state/circuits/{id}/{name}/isOn/boolean` - true or false
- `{rootTopic}/state/circuits/{id}/{name}/endTime` - End time for timed circuits
- `{rootTopic}/state/circuits/{id}/{name}/lightingTheme` - For light circuits
- `{rootTopic}/state/circuits/{id}/{name}/object` - Complete circuit object

### Features

**Topic:** `{rootTopic}/state/features/{id}/{name}`

**Payload:**
```json
{
  "id": 10,
  "isOn": "on",
  "endTime": "2024-01-15T14:30:00"
}
```

### Virtual Circuits

**Topic:** `{rootTopic}/state/virtualcircuits/{id}/{name}`

**Payload:**
```json
{
  "id": 200,
  "isOn": "on"
}
```

### Bodies (Pool/Spa)

**Topic:** `{rootTopic}/state/temps/bodies/{id}/{name}`

**Payload:**
```json
{
  "id": 1,
  "isOn": "on"
}
```

**Properties Available:**
- `{rootTopic}/state/temps/bodies/{id}/{name}/heatMode` - Heat mode setting
- `{rootTopic}/state/temps/bodies/{id}/{name}/heatStatus` - Current heating status
- `{rootTopic}/state/temps/bodies/{id}/{name}/setPoint` - Heat setpoint
- `{rootTopic}/state/temps/bodies/{id}/{name}/heatSetpoint` - Heat setpoint
- `{rootTopic}/state/temps/bodies/{id}/{name}/coolSetpoint` - Cool setpoint
- `{rootTopic}/state/temps/bodies/{id}/{name}/temp` - Current temperature

### Temperatures

**Topic:** `{rootTopic}/state/temps/air`

**Payload:**
```json
{
  "temp": 75
}
```

**Available Topics:**
- `{rootTopic}/state/temps/air` - Air temperature
- `{rootTopic}/state/temps/solar` - Solar temperature (if equipped)
- `{rootTopic}/state/temps/solarSensor2` - Additional solar sensor
- `{rootTopic}/state/temps/solarSensor3` - Additional solar sensor
- `{rootTopic}/state/temps/solarSensor4` - Additional solar sensor
- `{rootTopic}/state/temps/waterSensor1` - Water temperature sensor 1
- `{rootTopic}/state/temps/waterSensor2` - Water temperature sensor 2
- `{rootTopic}/state/temps/waterSensor3` - Water temperature sensor 3
- `{rootTopic}/state/temps/waterSensor4` - Water temperature sensor 4
- `{rootTopic}/state/temps/units` - Temperature units (F or C)

### Pumps

**Topic:** `{rootTopic}/state/pumps/{id}/{name}`

**Payload:**
```json
{
  "id": 1,
  "isOn": "on"
}
```

**Properties Available:**
- `{rootTopic}/state/pumps/{id}/{name}/rpm` - Current RPM
- `{rootTopic}/state/pumps/{id}/{name}/flow` - Current flow rate (GPM)
- `{rootTopic}/state/pumps/{id}/{name}/watts` - Current power consumption
- `{rootTopic}/state/pumps/{id}/{name}/status` - Pump status

### Heaters

**Topic:** `{rootTopic}/state/heaters/{id}/{name}`

Published when heater state changes.

**Payload:**
```json
{
  "id": 1,
  "name": "Gas Heater",
  "isOn": true,
  "startTime": "2024-01-15T12:00:00",
  "type": {
    "val": 1,
    "name": "gas",
    "desc": "Gas Heater"
  }
}
```

### Chlorinators

**Topic:** `{rootTopic}/state/chlorinators/{id}/{name}`

**Payload:**
```json
{
  "id": 1,
  "isOn": "on"
}
```

**Properties Available:**
- `{rootTopic}/state/chlorinators/{id}/{name}/currentOutput` - Current chlorine output %
- `{rootTopic}/state/chlorinators/{id}/{name}/poolSetpoint` - Pool output setpoint
- `{rootTopic}/state/chlorinators/{id}/{name}/spaSetpoint` - Spa output setpoint
- `{rootTopic}/state/chlorinators/{id}/{name}/status` - Chlorinator status
- `{rootTopic}/state/chlorinators/{id}/{name}/superChlor` - Super chlorination mode
- `{rootTopic}/state/chlorinators/{id}/{name}/superChlorHours` - Super chlor hours remaining
- `{rootTopic}/state/chlorinators/{id}/{name}/saltLevel` - Current salt level (ppm)
- `{rootTopic}/state/chlorinators/{id}/{name}/saltRequired` - Salt to add (lbs)
- `{rootTopic}/state/chlorinators/{id}/{name}/saltTarget` - Target salt level (ppm)
- `{rootTopic}/state/chlorinators/{id}/{name}/type` - Chlorinator type
- `{rootTopic}/state/chlorinators/{id}/{name}/model` - Chlorinator model
- `{rootTopic}/state/chlorinators/{id}/{name}/targetOutput` - Target output %

### Chemical Controllers

**Topic:** `{rootTopic}/state/chemControllers/{id}/{name}`

**pH Properties:**
- `{rootTopic}/state/chemControllers/{id}/{name}/ph/tankLevel` - pH tank level
- `{rootTopic}/state/chemControllers/{id}/{name}/ph/setpoint` - pH setpoint
- `{rootTopic}/state/chemControllers/{id}/{name}/ph/level` - Current pH level
- `{rootTopic}/state/chemControllers/{id}/{name}/ph/doseTime` - pH dose time (ms)
- `{rootTopic}/state/chemControllers/{id}/{name}/ph/doseVolume` - pH dose volume (mL)

**ORP Properties:**
- `{rootTopic}/state/chemControllers/{id}/{name}/orp/tankLevel` - ORP tank level
- `{rootTopic}/state/chemControllers/{id}/{name}/orp/setpoint` - ORP setpoint
- `{rootTopic}/state/chemControllers/{id}/{name}/orp/level` - Current ORP level
- `{rootTopic}/state/chemControllers/{id}/{name}/orp/demand` - ORP demand
- `{rootTopic}/state/chemControllers/{id}/{name}/orp/doseTime` - ORP dose time (ms)
- `{rootTopic}/state/chemControllers/{id}/{name}/orp/doseVolume` - ORP dose volume (mL)
- `{rootTopic}/state/chemControllers/{id}/{name}/orp/saltLevel` - Salt level (ppm)

**Water Chemistry:**
- `{rootTopic}/state/chemControllers/{id}/{name}/saturationIndex` - LSI value
- `{rootTopic}/state/chemControllers/{id}/{name}/alkalinity` - Total alkalinity (ppm)
- `{rootTopic}/state/chemControllers/{id}/{name}/calciumHardness` - Calcium hardness (ppm)
- `{rootTopic}/state/chemControllers/{id}/{name}/cyanuricAcid` - CYA level (ppm)
- `{rootTopic}/state/chemControllers/{id}/{name}/borates` - Borate level (ppm)

**Alarms:**
- `{rootTopic}/state/chemControllers/{id}/{name}/alarms/flow` - Flow alarm
- `{rootTopic}/state/chemControllers/{id}/{name}/alarms/ph` - pH alarm
- `{rootTopic}/state/chemControllers/{id}/{name}/alarms/orp` - ORP alarm
- `{rootTopic}/state/chemControllers/{id}/{name}/alarms/phTank` - pH tank alarm
- `{rootTopic}/state/chemControllers/{id}/{name}/alarms/orpTank` - ORP tank alarm
- `{rootTopic}/state/chemControllers/{id}/{name}/alarms/orpProbeFault` - ORP probe fault
- `{rootTopic}/state/chemControllers/{id}/{name}/alarms/pHProbeFault` - pH probe fault

**Warnings:**
- `{rootTopic}/state/chemControllers/{id}/{name}/warnings/waterChemistry` - Water chemistry warning
- `{rootTopic}/state/chemControllers/{id}/{name}/warnings/pHLockout` - pH lockout
- `{rootTopic}/state/chemControllers/{id}/{name}/warnings/pHDailyLimitReached` - pH daily limit
- `{rootTopic}/state/chemControllers/{id}/{name}/warnings/orpDailyLimitReached` - ORP daily limit
- `{rootTopic}/state/chemControllers/{id}/{name}/warnings/invalidSetup` - Invalid setup
- `{rootTopic}/state/chemControllers/{id}/{name}/warnings/chlorinatorCommError` - Chlorinator comm error

### Chemical Dosing Events

**Topic:** `{rootTopic}/state/chemControllers/{id}/{chem}`

Published when a chemical is being dosed (real-time updates).

**Payload:**
```json
{
  "id": 1,
  "chem": "ph",
  "volumeDosed": 25.5,
  "timeDosed": 5000,
  "timeRemaining": 1000
}
```

### Filters

**Topic:** `{rootTopic}/state/filters/{id}/{name}`

**Payload:**
```json
{
  "id": 1,
  "isOn": "on"
}
```

**Properties Available:**
- `{rootTopic}/state/filters/{id}/{name}/pressure` - Current filter pressure (PSI)
- `{rootTopic}/state/filters/{id}/{name}/refPressure` - Reference/clean pressure (PSI)
- `{rootTopic}/state/filters/{id}/{name}/body` - Associated body description
- `{rootTopic}/state/filters/{id}/{name}/filterType` - Filter type (sand, DE, cartridge)
- `{rootTopic}/state/filters/{id}/{name}/pressureUnits` - Pressure units
- `{rootTopic}/state/filters/{id}/{name}/cleanPercentage` - Filter cleanliness percentage

### Valves

**Topic:** `{rootTopic}/state/valve/{id}/{name}`

**Payload:**
```json
{
  "id": 1,
  "isOn": "on",
  "isVirtual": false,
  "pinId": 5
}
```

### Circuit Groups

**Topic:** `{rootTopic}/state/circuitGroups/{id}/{name}`

**Payload:**
```json
{
  "id": 1,
  "isOn": "on",
  "endTime": "2024-01-15T14:30:00"
}
```

**Properties Available:**
- `{rootTopic}/state/circuitGroups/{id}/{name}/type` - Group type
- `{rootTopic}/state/circuitGroups/{id}/{name}/showInFeatures` - Show in features flag

### Light Groups

**Topic:** `{rootTopic}/state/lightgroups/{id}/{name}`

**Payload:**
```json
{
  "id": 1,
  "isOn": "on"
}
```

**Properties Available:**
- `{rootTopic}/state/lightgroups/{id}/{name}/action` - Current action
- `{rootTopic}/state/lightgroups/{id}/{name}/lightingTheme` - Current theme
- `{rootTopic}/state/lightgroups/{id}/{name}/type` - Light group type
- `{rootTopic}/state/lightgroups/{id}/{name}/endTime` - End time

### Schedules

**Topic:** `{rootTopic}/state/schedules/{id}/{name}`

Published when schedule state changes.

**Payload:**
```json
{
  "id": 1,
  "name": "Pool Circulation",
  "isOn": true,
  "startTime": "2024-01-15T08:00:00",
  "endTime": "2024-01-15T20:00:00",
  "circuit": {
    "id": 6,
    "name": "Pool"
  }
}
```

### Covers

**Topic:** `{rootTopic}/state/covers/{id}/{name}`

**Payload:**
```json
{
  "id": 1,
  "isClosed": "true"
}
```

## Subscription Topics (Commands)

Subscribe to these topics to send commands to the controller.

### Circuit Control

**Set Circuit State:**

**Topic:** `{rootTopic}/state/circuit/setState` or `{rootTopic}/state/circuits/setState`

**Payload:**
```json
{
  "id": 1,
  "isOn": true
}
```
or
```json
{
  "id": 1,
  "state": true
}
```

**Toggle Circuit State:**

**Topic:** `{rootTopic}/state/circuit/toggleState` or `{rootTopic}/state/circuits/toggleState`

**Payload:**
```json
{
  "id": 1
}
```

### Feature Control

**Set Feature State:**

**Topic:** `{rootTopic}/state/feature/setState` or `{rootTopic}/state/features/setState`

**Payload:**
```json
{
  "id": 10,
  "isOn": true
}
```

**Toggle Feature State:**

**Topic:** `{rootTopic}/state/feature/toggleState` or `{rootTopic}/state/features/toggleState`

**Payload:**
```json
{
  "id": 10
}
```

### Light Group Control

**Set Light Group State:**

**Topic:** `{rootTopic}/state/lightgroup/setState` or `{rootTopic}/state/lightgroups/setState`

**Payload:**
```json
{
  "id": 1,
  "isOn": true
}
```

**Set Light Theme:**

**Topic:** `{rootTopic}/state/lightgroup/setTheme` or `{rootTopic}/state/lightgroups/setTheme`

**Payload:**
```json
{
  "id": 1,
  "theme": "party"
}
```

### Circuit Group Control

**Set Circuit Group State:**

**Topic:** `{rootTopic}/state/circuitgroup/setState` or `{rootTopic}/state/circuitgroups/setState`

**Payload:**
```json
{
  "id": 1,
  "isOn": true
}
```

### Body Temperature Control

**Set Heat Setpoint:**

**Topic:** `{rootTopic}/state/body/heatSetpoint` or `{rootTopic}/state/body/setPoint`

**Payload:**
```json
{
  "id": 1,
  "setPoint": 82
}
```
or
```json
{
  "id": 1,
  "heatSetpoint": 82
}
```

**Set Cool Setpoint:**

**Topic:** `{rootTopic}/state/body/coolSetpoint`

**Payload:**
```json
{
  "id": 1,
  "coolSetpoint": 80
}
```

**Set Heat Mode:**

**Topic:** `{rootTopic}/state/body/heatMode`

**Payload:**
```json
{
  "id": 1,
  "mode": "heater"
}
```

Valid modes: `off`, `heater`, `solar`, `solarpreferred`, `heatpump`

### Temperature Sensors

**Set Temperature Values:**

**Topic:** `{rootTopic}/state/temps` or `{rootTopic}/config/temps`

**Payload:**
```json
{
  "air": 75,
  "waterSensor1": 80
}
```

**Configure Temperature Sensors:**

**Topic:** `{rootTopic}/config/tempSensors`

**Payload:**
```json
{
  "sensors": [
    {
      "id": 1,
      "name": "Pool Temp",
      "type": "water"
    }
  ]
}
```

### Chlorinator Control

**Topic:** `{rootTopic}/state/chlorinator` or `{rootTopic}/config/chlorinator`

**Payload:**
```json
{
  "id": 1,
  "poolSetpoint": 50,
  "spaSetpoint": 10,
  "superChlor": true,
  "superChlorHours": 8
}
```

### Chemical Controller Control

**Topic:** `{rootTopic}/state/chemController` or `{rootTopic}/config/chemController`

**Payload:**
```json
{
  "id": 1,
  "ph": {
    "setpoint": 7.4
  },
  "orp": {
    "setpoint": 650
  }
}
```

## EMQX Integration

### EMQX Configuration

1. **Install EMQX:**
```bash
docker run -d --name emqx -p 1883:1883 -p 8083:8083 -p 8084:8084 -p 8883:8883 -p 18083:18083 emqx/emqx:latest
```

2. **Access EMQX Dashboard:**
   - URL: http://localhost:18083
   - Default credentials: admin / public

3. **Create Authentication:**
   - Go to Authentication → Create
   - Choose "Password-Based" authentication
   - Add users for nodejs-poolController

4. **Configure ACL (Access Control):**

Create topic-based access control:

```json
{
  "topic": "pool/#",
  "action": "all",
  "effect": "allow"
}
```

### EMQX Rule Engine for Data Logging

Configure EMQX rules to log specific data points to a database:

1. **Create a Rule:**

Go to Rules → Create Rule

```sql
SELECT
  payload.pressure as pressure,
  payload.refPressure as refPressure,
  payload.cleanPercentage as cleanPercentage,
  timestamp as ts
FROM
  "pool/state/filters/+/+/pressure"
```

2. **Add Action:**

Add an action to save to your database (InfluxDB, MySQL, PostgreSQL, etc.)

### Example Rules for Important Data Points

**Filter Pressure Monitoring:**
```sql
SELECT
  payload.pressure as filter_pressure,
  payload.refPressure as ref_pressure,
  topic as topic,
  timestamp as ts
FROM
  "pool/state/filters/#"
WHERE
  payload.pressure > 0
```

**Temperature Monitoring:**
```sql
SELECT
  payload.temp as temperature,
  topic as topic,
  timestamp as ts
FROM
  "pool/state/temps/+",
  "pool/state/temps/bodies/+/+"
WHERE
  payload.temp IS NOT NULL
```

**Chemical Levels:**
```sql
SELECT
  payload as level,
  topic as topic,
  timestamp as ts
FROM
  "pool/state/chemControllers/+/+/ph/level",
  "pool/state/chemControllers/+/+/orp/level",
  "pool/state/chlorinators/+/+/saltLevel"
```

**Pump Power Consumption:**
```sql
SELECT
  payload.watts as power_watts,
  payload.rpm as rpm,
  payload.flow as flow_gpm,
  topic as topic,
  timestamp as ts
FROM
  "pool/state/pumps/#"
WHERE
  payload.watts > 0
```

## Data Logging

### Important Data Points for Logging

#### Equipment Runtime
- `{rootTopic}/state/pumps/+/+/watts` - Track energy consumption
- `{rootTopic}/state/heaters/+/+` - Track heating cycles
- `{rootTopic}/state/circuits/+/+` - Track circuit on/off times

#### Water Quality
- `{rootTopic}/state/chemControllers/+/+/ph/level` - pH trends
- `{rootTopic}/state/chemControllers/+/+/orp/level` - ORP trends
- `{rootTopic}/state/chlorinators/+/+/saltLevel` - Salt level trends
- `{rootTopic}/state/chemControllers/+/+/saturationIndex` - LSI trends

#### Filter Maintenance
- `{rootTopic}/state/filters/+/+/pressure` - Filter pressure over time
- `{rootTopic}/state/filters/+/+/cleanPercentage` - Filter cleanliness

#### Temperature Monitoring
- `{rootTopic}/state/temps/air` - Air temperature
- `{rootTopic}/state/temps/bodies/+/+/temp` - Water temperature
- `{rootTopic}/state/temps/solar` - Solar collector temperature

#### Chemical Consumption
- `{rootTopic}/state/chemControllers/+/+/ph/doseVolume` - pH chemical usage
- `{rootTopic}/state/chemControllers/+/+/orp/doseVolume` - Chlorine/oxidizer usage
- `{rootTopic}/state/chemControllers/+/+/ph/tankLevel` - pH tank level trends
- `{rootTopic}/state/chemControllers/+/+/orp/tankLevel` - ORP tank level trends

### InfluxDB Line Protocol Example

If using direct InfluxDB integration instead of EMQX rules:

```
pool_filter,id=1,name=main_filter pressure=15.2,ref_pressure=10.0,clean_percentage=85.5
pool_temperature,sensor=air value=75.3
pool_temperature,sensor=pool value=82.1
pool_chemistry,controller=1,probe=ph value=7.42
pool_chemistry,controller=1,probe=orp value=685
pool_pump,id=1,name=main_pump rpm=2400,watts=1850,flow=55
```

## Examples

### Example 1: Home Assistant Integration via MQTT

```yaml
# configuration.yaml
mqtt:
  sensor:
    - name: "Pool Temperature"
      state_topic: "pool/state/temps/bodies/1/pool/temp"
      value_template: "{{ value_json.temp }}"
      unit_of_measurement: "°F"
      device_class: temperature
    
    - name: "Filter Pressure"
      state_topic: "pool/state/filters/1/main filter/pressure"
      value_template: "{{ value_json.pressure }}"
      unit_of_measurement: "PSI"
    
    - name: "Pool pH"
      state_topic: "pool/state/chemControllers/1/intellichem/pHLevel"
      value_template: "{{ value_json.pHLevel }}"
  
  switch:
    - name: "Pool Pump"
      command_topic: "pool/state/circuit/setState"
      state_topic: "pool/state/circuits/6/pool"
      value_template: "{{ 'ON' if value_json.isOn == 'on' else 'OFF' }}"
      payload_on: '{"id": 6, "isOn": true}'
      payload_off: '{"id": 6, "isOn": false}'
      retain: true
```

### Example 2: Node-RED Flow

```json
[
  {
    "id": "mqtt_in",
    "type": "mqtt in",
    "topic": "pool/state/filters/+/+/pressure",
    "broker": "mqtt_broker"
  },
  {
    "id": "function",
    "type": "function",
    "func": "const pressure = msg.payload.pressure;\nconst refPressure = msg.payload.refPressure;\nconst diff = pressure - refPressure;\n\nif (diff > 5) {\n  msg.payload = 'Filter pressure high: ' + pressure + ' PSI';\n  return msg;\n}\nreturn null;"
  },
  {
    "id": "notification",
    "type": "pushbullet",
    "title": "Pool Alert"
  }
]
```

### Example 3: Python MQTT Client

```python
import paho.mqtt.client as mqtt
import json

def on_connect(client, userdata, flags, rc):
    print(f"Connected with result code {rc}")
    # Subscribe to all state updates
    client.subscribe("pool/state/#")
    
def on_message(client, userdata, msg):
    topic = msg.topic
    payload = json.loads(msg.payload)
    print(f"Topic: {topic}")
    print(f"Payload: {payload}")
    
    # Example: Check filter pressure
    if "filters" in topic and "pressure" in topic:
        pressure = payload.get("pressure", 0)
        if pressure > 20:
            print(f"WARNING: High filter pressure: {pressure} PSI")

# Create client
client = mqtt.Client()
client.username_pw_set("username", "password")
client.on_connect = on_connect
client.on_message = on_message

# Connect to EMQX
client.connect("localhost", 1883, 60)

# Set pool temperature
def set_pool_temp(temp):
    payload = json.dumps({"id": 1, "setPoint": temp})
    client.publish("pool/state/body/setPoint", payload, qos=2, retain=True)

# Turn on pool pump
def turn_on_pump():
    payload = json.dumps({"id": 6, "isOn": True})
    client.publish("pool/state/circuit/setState", payload, qos=2, retain=True)

# Start the client
client.loop_forever()
```

### Example 4: Monitoring Filter Pressure

```javascript
// Node.js example
const mqtt = require('mqtt');
const client = mqtt.connect('mqtt://localhost:1883', {
  username: 'username',
  password: 'password'
});

client.on('connect', () => {
  console.log('Connected to MQTT broker');
  client.subscribe('pool/state/filters/+/+/pressure');
});

client.on('message', (topic, message) => {
  const data = JSON.parse(message.toString());
  const pressure = data.pressure;
  const refPressure = data.refPressure;
  const cleanPercentage = data.cleanPercentage;
  
  console.log(`Filter Pressure: ${pressure} PSI`);
  console.log(`Reference Pressure: ${refPressure} PSI`);
  console.log(`Clean Percentage: ${cleanPercentage}%`);
  
  // Alert if pressure is too high
  if (pressure - refPressure > 8) {
    console.warn('⚠️  Time to clean the filter!');
    // Send notification, email, etc.
  }
  
  // Log to database
  logToDatabase({
    timestamp: new Date(),
    pressure: pressure,
    refPressure: refPressure,
    cleanPercentage: cleanPercentage
  });
});
```

## Troubleshooting

### Connection Issues

1. **Check broker connectivity:**
```bash
mosquitto_sub -h localhost -p 1883 -t "pool/#" -u username -P password -v
```

2. **Verify credentials in config.json**

3. **Check firewall settings for MQTT port**

4. **Review EMQX logs:**
```bash
docker logs emqx
```

### Message Not Publishing

1. **Enable MQTT debug logging in nodejs-poolController**

2. **Check retain and QoS settings**

3. **Verify topic structure matches bindings**

4. **Check changesOnly setting** - messages only publish on state changes by default

### Subscription Not Working

1. **Verify topic subscription in MQTT client**

2. **Check message format matches expected JSON structure**

3. **Ensure controller is ready** (status.percent === 100)

4. **Review nodejs-poolController logs for MQTT errors**

## Advanced Topics

### Custom Subscriptions with Processors

You can create custom subscription processors in `mqtt.json`:

```json
{
  "subscriptions": [
    {
      "topic": "custom/command",
      "enabled": true,
      "processor": [
        "// Custom processing logic",
        "const command = value.command;",
        "if (command === 'turnOnAll') {",
        "  state.circuits.forEach(circuit => {",
        "    sys.board.circuits.setCircuitStateAsync(circuit.id, true);",
        "  });",
        "}"
      ]
    }
  ]
}
```

### Topic Formatting

Customize topic formatting with formatters in `mqtt.json`:

```json
{
  "formatter": [
    {
      "transform": ".toLowerCase()"
    },
    {
      "regexkey": "\\s",
      "replace": "_",
      "description": "Replace spaces with underscores"
    }
  ]
}
```

## Security Best Practices

1. **Use strong passwords** for MQTT authentication
2. **Enable TLS/SSL** for encrypted communication (mqtts://)
3. **Implement ACL rules** to restrict topic access
4. **Use separate credentials** for different clients
5. **Keep EMQX updated** to latest stable version
6. **Monitor authentication failures** in EMQX dashboard
7. **Limit QoS 2** usage to critical messages only

## Conclusion

The MQTT interface provides comprehensive control and monitoring of all pool equipment. With proper integration with EMQX or another MQTT broker, you can:

- Monitor all equipment states in real-time
- Control pool equipment remotely
- Log historical data for analysis
- Integrate with home automation systems
- Create custom automation rules
- Receive alerts and notifications

For more information, see:
- [Web System Documentation](web-system.md)
- [Configuration System](config-system.md)
- [nodejs-poolController GitHub](https://github.com/tagyoureit/nodejs-poolController)
