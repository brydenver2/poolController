# MQTT Enhancement Examples

This file demonstrates the new MQTT features added to nodejs-poolController.

## New Features

### 1. Heater State Publishing

Heaters now publish their state to MQTT automatically:

**Published Topics:**
- `pool/state/heaters/1/Gas Heater` - Main heater state
- `pool/state/heaters/1/Gas Heater/isOn` - On/off state (boolean)
- `pool/state/heaters/1/Gas Heater/type` - Heater type (gas, solar, heatpump)
- `pool/state/heaters/1/Gas Heater/startTime` - When heater turned on
- `pool/state/heaters/1/Gas Heater/endTime` - When heater turned off

**Example Payload:**
```json
{
  "id": 1,
  "isOn": true,
  "name": "Gas Heater"
}
```

### 2. Schedule State Publishing

Schedules now publish their state including circuit information:

**Published Topics:**
- `pool/state/schedules/1/Pool Circulation` - Complete schedule state
- `pool/state/schedules/1/Pool Circulation/isOn` - Schedule running state
- `pool/state/schedules/1/Pool Circulation/startTime` - Next/current start time
- `pool/state/schedules/1/Pool Circulation/endTime` - Next/current end time

**Example Payload:**
```json
{
  "id": 1,
  "name": "Pool Circulation",
  "isOn": true,
  "circuit": {
    "id": 6,
    "name": "Pool"
  },
  "scheduleTime": {
    "startTime": "2024-01-15T08:00:00",
    "endTime": "2024-01-15T20:00:00"
  }
}
```

### 3. Pump Speed Control

You can now control pump speed via MQTT subscriptions:

**Subscription Topic:** `pool/state/pump/setSpeed`

**Set RPM:**
```json
{
  "id": 1,
  "rpm": 2400
}
```

**Set Flow (GPM):**
```json
{
  "id": 1,
  "flow": 55
}
```

**Set Speed (for single/dual speed pumps):**
```json
{
  "id": 1,
  "speed": 2
}
```

## Complete Integration Example

### Python Script for Monitoring and Control

```python
import paho.mqtt.client as mqtt
import json
from datetime import datetime

# Configuration
BROKER = "localhost"
PORT = 1883
USERNAME = "pool_user"
PASSWORD = "pool_pass"
ROOT_TOPIC = "pool"

# Track filter pressure for maintenance alerts
filter_pressure_history = []

def on_connect(client, userdata, flags, rc):
    print(f"Connected to MQTT broker with result code {rc}")
    
    # Subscribe to all state updates
    client.subscribe(f"{ROOT_TOPIC}/state/#")
    
def on_message(client, userdata, msg):
    topic = msg.topic
    
    try:
        payload = json.loads(msg.payload.decode())
    except:
        payload = msg.payload.decode()
    
    print(f"[{datetime.now().strftime('%H:%M:%S')}] {topic}: {payload}")
    
    # Monitor filter pressure
    if "filters" in topic and "pressure" in topic:
        handle_filter_pressure(payload)
    
    # Monitor heater runtime
    elif "heaters" in topic and "/isOn" in topic:
        handle_heater_state(topic, payload)
    
    # Monitor schedule changes
    elif "schedules" in topic:
        handle_schedule_update(topic, payload)

def handle_filter_pressure(data):
    """Monitor filter pressure and alert when cleaning needed"""
    pressure = data.get("pressure", 0)
    ref_pressure = data.get("refPressure", 0)
    
    if pressure > 0 and ref_pressure > 0:
        diff = pressure - ref_pressure
        filter_pressure_history.append({
            "timestamp": datetime.now(),
            "pressure": pressure,
            "diff": diff
        })
        
        # Alert if pressure difference exceeds threshold
        if diff > 8:
            print(f"⚠️  ALERT: Filter pressure high! {pressure} PSI (ref: {ref_pressure} PSI)")
            print(f"    Difference: {diff} PSI - Time to clean filter!")
            # Send notification here (email, push, etc.)

def handle_heater_state(topic, is_on):
    """Log heater on/off events for runtime tracking"""
    heater_name = topic.split('/')[-2]
    state = "ON" if is_on else "OFF"
    print(f"🔥 Heater '{heater_name}' turned {state}")
    # Log to database for runtime analysis

def handle_schedule_update(topic, data):
    """Monitor schedule execution"""
    if isinstance(data, dict) and data.get("isOn"):
        schedule_name = topic.split('/')[-1]
        print(f"📅 Schedule '{schedule_name}' is now running")
        # Log schedule execution

# Control functions
def set_pool_temperature(client, body_id, temp):
    """Set pool/spa temperature"""
    payload = json.dumps({"id": body_id, "setPoint": temp})
    client.publish(f"{ROOT_TOPIC}/state/body/setPoint", payload, qos=2)
    print(f"Setting body {body_id} temperature to {temp}°F")

def set_pump_speed(client, pump_id, rpm):
    """Set pump speed in RPM"""
    payload = json.dumps({"id": pump_id, "rpm": rpm})
    client.publish(f"{ROOT_TOPIC}/state/pump/setSpeed", payload, qos=2)
    print(f"Setting pump {pump_id} to {rpm} RPM")

def turn_on_circuit(client, circuit_id):
    """Turn on a circuit"""
    payload = json.dumps({"id": circuit_id, "isOn": True})
    client.publish(f"{ROOT_TOPIC}/state/circuit/setState", payload, qos=2)
    print(f"Turning on circuit {circuit_id}")

def turn_off_circuit(client, circuit_id):
    """Turn off a circuit"""
    payload = json.dumps({"id": circuit_id, "isOn": False})
    client.publish(f"{ROOT_TOPIC}/state/circuit/setState", payload, qos=2)
    print(f"Turning off circuit {circuit_id}")

# Main
if __name__ == "__main__":
    client = mqtt.Client()
    client.username_pw_set(USERNAME, PASSWORD)
    client.on_connect = on_connect
    client.on_message = on_message
    
    client.connect(BROKER, PORT, 60)
    
    # Example: Set pool temp to 82°F
    set_pool_temperature(client, 1, 82)
    
    # Example: Set pump to 2400 RPM
    set_pump_speed(client, 1, 2400)
    
    # Example: Turn on pool circuit
    turn_on_circuit(client, 6)
    
    # Start listening for messages
    print("Monitoring pool equipment...")
    client.loop_forever()
```

### Node-RED Flow for Filter Pressure Monitoring

```json
[
  {
    "id": "mqtt_filter_pressure",
    "type": "mqtt in",
    "topic": "pool/state/filters/+/+/pressure",
    "qos": "2",
    "broker": "mqtt_broker",
    "name": "Filter Pressure"
  },
  {
    "id": "check_pressure",
    "type": "function",
    "name": "Check Pressure",
    "func": "const pressure = msg.payload.pressure;\nconst refPressure = msg.payload.refPressure;\nconst diff = pressure - refPressure;\n\nmsg.pressure = pressure;\nmsg.refPressure = refPressure;\nmsg.diff = diff;\n\nif (diff > 8) {\n    msg.payload = `Filter needs cleaning! Pressure: ${pressure} PSI (${diff} PSI above reference)`;\n    return msg;\n}\n\nreturn null;"
  },
  {
    "id": "send_notification",
    "type": "pushbullet",
    "title": "Pool Maintenance Alert"
  }
]
```

### Home Assistant Configuration

```yaml
# configuration.yaml

mqtt:
  sensor:
    # Monitor filter pressure
    - name: "Pool Filter Pressure"
      state_topic: "pool/state/filters/1/main filter/pressure"
      value_template: "{{ value_json.pressure }}"
      unit_of_measurement: "PSI"
      device_class: pressure
    
    - name: "Pool Filter Reference Pressure"
      state_topic: "pool/state/filters/1/main filter/refPressure"
      value_template: "{{ value_json.refPressure }}"
      unit_of_measurement: "PSI"
    
    - name: "Pool Filter Clean Percentage"
      state_topic: "pool/state/filters/1/main filter/cleanPercentage"
      value_template: "{{ value_json.cleanPercentage }}"
      unit_of_measurement: "%"
    
    # Monitor heater state
    - name: "Pool Heater State"
      state_topic: "pool/state/heaters/1/Gas Heater/isOn"
      value_template: "{{ 'On' if value else 'Off' }}"
    
    # Monitor schedule state
    - name: "Pool Circulation Schedule"
      state_topic: "pool/state/schedules/1/Pool Circulation/isOn"
      value_template: "{{ 'Running' if value else 'Off' }}"
    
    # Monitor pump
    - name: "Pool Pump RPM"
      state_topic: "pool/state/pumps/1/Pool Pump/rpm"
      value_template: "{{ value_json.rpm }}"
      unit_of_measurement: "RPM"
    
    - name: "Pool Pump Power"
      state_topic: "pool/state/pumps/1/Pool Pump/watts"
      value_template: "{{ value_json.watts }}"
      unit_of_measurement: "W"
      device_class: power
  
  # Control pump speed
  number:
    - name: "Pool Pump Speed"
      command_topic: "pool/state/pump/setSpeed"
      command_template: '{"id": 1, "rpm": {{ value | int }}}'
      min: 600
      max: 3450
      step: 100
      mode: slider

# Automation: Alert when filter pressure is high
automation:
  - alias: "Filter Pressure High Alert"
    trigger:
      - platform: numeric_state
        entity_id: sensor.pool_filter_pressure
        above: 20
        for:
          minutes: 5
    action:
      - service: notify.mobile_app
        data:
          title: "Pool Maintenance Needed"
          message: "Filter pressure is {{ states('sensor.pool_filter_pressure') }} PSI. Time to clean the filter!"
  
  # Log heater runtime
  - alias: "Log Heater Runtime"
    trigger:
      - platform: state
        entity_id: sensor.pool_heater_state
    action:
      - service: logbook.log
        data:
          name: "Pool Heater"
          message: "Heater {{ trigger.to_state.state }}"
```

## Testing Your Setup

### 1. Subscribe to all pool topics
```bash
mosquitto_sub -h localhost -p 1883 -t "pool/#" -u username -P password -v
```

### 2. Test pump control
```bash
# Set pump to 2400 RPM
mosquitto_pub -h localhost -p 1883 -t "pool/state/pump/setSpeed" -u username -P password -m '{"id": 1, "rpm": 2400}'

# Set pump to 50 GPM
mosquitto_pub -h localhost -p 1883 -t "pool/state/pump/setSpeed" -u username -P password -m '{"id": 1, "flow": 50}'
```

### 3. Monitor specific equipment
```bash
# Monitor heaters
mosquitto_sub -h localhost -p 1883 -t "pool/state/heaters/#" -u username -P password -v

# Monitor schedules
mosquitto_sub -h localhost -p 1883 -t "pool/state/schedules/#" -u username -P password -v

# Monitor filter pressure
mosquitto_sub -h localhost -p 1883 -t "pool/state/filters/+/+/pressure" -u username -P password -v
```

## Data Logging with EMQX

### Create Rule for Filter Pressure Logging

1. Go to EMQX Dashboard → Rules
2. Create new rule with SQL:

```sql
SELECT
  payload.pressure as filter_pressure,
  payload.refPressure as ref_pressure,
  payload.cleanPercentage as clean_percentage,
  timestamp as ts
FROM
  "pool/state/filters/#"
WHERE
  payload.pressure > 0
```

3. Add action to save to your database (InfluxDB, PostgreSQL, MySQL, etc.)

### Example InfluxDB Action

```json
{
  "measurement": "pool_filter",
  "tags": {
    "filter_id": "${filter_id}"
  },
  "fields": {
    "pressure": "${filter_pressure}",
    "ref_pressure": "${ref_pressure}",
    "clean_percentage": "${clean_percentage}"
  },
  "timestamp": "${ts}"
}
```

## Benefits

1. **Complete Equipment Monitoring**: Monitor all equipment including heaters and schedules
2. **Pump Control**: Remotely adjust pump speeds for energy optimization
3. **Maintenance Alerts**: Track filter pressure to know when cleaning is needed
4. **Runtime Tracking**: Log heater and pump runtime for cost analysis
5. **Schedule Monitoring**: Track when automated schedules run
6. **Home Automation**: Full integration with Home Assistant, Node-RED, etc.
7. **Data Analytics**: Historical data logging for trend analysis

For complete documentation, see [docs/mqtt-interface.md](../docs/mqtt-interface.md)
