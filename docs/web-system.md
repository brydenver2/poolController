# Web Server & Interfaces Documentation

## Overview

The web system provides HTTP/HTTPS REST APIs, WebSocket real-time communication, and external integrations (MQTT, InfluxDB, Rules, REM).

**Location:** `/web` folder

---

## Core Files

### `Server.ts`
**Purpose:** Main web server orchestrator and multiple interface manager

**Class:** `WebServer` (exported as `webApp` singleton)

#### Key Responsibilities
- HTTP/HTTPS/HTTP2 server management
- Socket.IO real-time communication
- MDNS/SSDP service discovery
- External interface orchestration (MQTT, InfluxDB, etc.)
- Automatic configuration backup

#### Server Types

**`HttpServer`**
- Standard HTTP server (port 4200 default)
- Express.js based REST API
- Socket.IO websocket support
- Optional authentication

**`HttpsServer`**
- SSL/TLS encrypted HTTP
- Requires certificate and key files
- Can redirect HTTP traffic to HTTPS
- Same API surface as HTTP

**`Http2Server`**
- HTTP/2 protocol support
- Better performance for modern clients
- Multiplexed connections
- Currently not actively used

**`MdnsServer`**
- Multicast DNS (Bonjour/Avahi)
- Service discovery on local network
- Advertises as "_poolcontroller._tcp"
- Auto-discovery by dashPanel clients

**`SsdpServer`**
- Simple Service Discovery Protocol
- UPnP-style device discovery
- Broader network discovery
- Alternative to MDNS

#### Key Methods

**`init()`**
- Initializes all configured servers
- Sets up Express routes
- Configures Socket.IO
- Starts listening on configured ports

**`initInterfaces()`**
- Initializes external interfaces (MQTT, InfluxDB, etc.)
- Loads binding configurations
- Establishes connections

**`initAutoBackup()`**
- Configures automatic backups
- Scheduled configuration snapshots
- Stored in `/backups` folder

**`emitToClients(event: string, data: any)`**
- Broadcasts event to all connected Socket.IO clients
- Used for real-time state updates

**`emitToChannel(channel: string, event: string, data: any)`**
- Broadcasts to specific Socket.IO channel
- More targeted than `emitToClients`
- Examples: 'controller', 'equipment', 'state'

**`stopAsync()`**
- Gracefully shuts down all servers
- Closes all connections
- Stops interfaces

#### Express Routes

Routes are mounted from service modules:

```
/config/*    - Configuration API (ConfigRoute)
/state/*     - Equipment state API (StateRoute)
/utilities/* - Utility functions (UtilitiesRoute)
```

#### Socket.IO Namespaces

**Default namespace (`/`):**
- Equipment state updates
- Configuration changes
- Connection status

**Events Emitted:**
- `controller` - Controller status
- `config` - Configuration changes
- `equipment` - Equipment changes
- `circuit` - Circuit state
- `body` - Body temperature
- `pump` - Pump state
- `chlorinator` - Chlorinator state
- `chemController` - Chemistry controller state

#### Automatic Backups

**Configuration:**
```json
{
  "web": {
    "autoBackup": {
      "enabled": true,
      "interval": 86400000,  // 24 hours
      "retention": 7         // Keep 7 days
    }
  }
}
```

**Backup Contents:**
- `config.json`
- `/data/poolConfig.json`
- `/data/poolState.json`

**Backup Location:** `/backups/backup-YYYY-MM-DD_HH-MM-SS.zip`

---

## Services

### `services/config/Config.ts`
**Purpose:** REST API routes for configuration management

**Class:** `ConfigRoute`

#### Endpoints

**GET `/config/all`**
- Returns complete configuration
- Includes equipment, schedules, options

**GET `/config/:section`**
- Returns specific configuration section
- Examples: `/config/circuits`, `/config/pumps`

**PUT `/config/:section/:id`**
- Updates specific equipment
- Validates changes before applying

**POST `/config/:section`**
- Creates new equipment
- Auto-assigns IDs

**DELETE `/config/:section/:id`**
- Removes equipment
- Validates removal is safe

**GET `/config/options/circuits`**
- Returns available circuit options
- Depends on controller type

---

### `services/config/ConfigSocket.ts`
**Purpose:** Socket.IO handlers for configuration

**Class:** `ConfigSocket`

#### Events Handled

**`setCircuitState`**
- Client request to change circuit
- Validates and applies change
- Emits result to all clients

**`setHeatMode`**
- Change body heat mode
- Validates mode for controller

**`setHeatSetpoint`**
- Change temperature setpoint
- Range validation

**`setChemController`**
- Update chemistry controller settings
- IntelliChem or REM configuration

---

### `services/state/State.ts`
**Purpose:** REST API routes for real-time state

**Class:** `StateRoute`

#### Endpoints

**GET `/state/all`**
- Complete equipment state snapshot
- All circuits, pumps, temperatures, etc.

**GET `/state/:section`**
- Specific state section
- Examples: `/state/temps`, `/state/pumps`

**GET `/state/circuit/:id`**
- Single circuit state

**PUT `/state/circuit/:id/setState`**
- Turn circuit on/off
- Immediate action

**GET `/state/body/:id`**
- Body temperature and heat state

**PUT `/state/body/:id/heatMode`**
- Set heating mode

**PUT `/state/body/:id/setPoint`**
- Set target temperature

---

### `services/state/StateSocket.ts`
**Purpose:** Socket.IO handlers for state control

**Class:** `StateSocket`

#### Events Handled

**`getState`**
- Client requests current state
- Returns complete snapshot

**`setCircuitState`**
- Toggle circuit
- Emits updated state

**`cancelDelay`**
- Cancel equipment delay/lockout
- Administrative override

---

### `services/utilities/Utilities.ts`
**Purpose:** Utility endpoints and file operations

**Class:** `UtilitiesRoute`

#### Endpoints

**GET `/app/version`**
- Returns application version
- Git information if available

**GET `/app/reload`**
- Reloads configuration from disk
- Hot-reload without restart

**POST `/app/backup`**
- Triggers manual backup
- Returns backup file path

**POST `/app/restore`**
- Restores from backup file
- Validates before applying

**GET `/app/logs/packet`**
- Downloads packet log file
- For debugging

**POST `/app/capture/start`**
- Starts packet capture mode
- For bug reports

**POST `/app/capture/stop`**
- Stops capture and creates ZIP
- Returns download URL

**POST `/upload/bindings`**
- Upload custom binding file
- Validates JSON format

**GET `/bindings/list`**
- Lists available binding templates
- From `/web/bindings/` folder

---

## Interfaces

External integration interfaces located in `/web/interfaces/`

### `baseInterface.ts`
**Purpose:** Abstract base class for all interfaces

**Class:** `InterfaceServer`

#### Key Features
- Configuration management
- Connection lifecycle
- Event subscription
- UUID-based identification

#### Key Methods (Override in Subclasses)

**`connect()`**
- Establish connection to external system
- Called during initialization

**`disconnect()`**
- Close connection gracefully
- Called during shutdown

**`emitData(event: string, data: any)`**
- Send data to external system
- Event-based routing

---

### `mqttInterface.ts`
**Purpose:** MQTT broker integration

**Class:** `MqttInterfaceBindings`

#### Key Features

1. **Bidirectional Communication**
   - Publishes equipment state changes
   - Subscribes to control commands
   - QoS support

2. **Topic Structure**
```
{baseTopic}/state/circuits/1      # Circuit 1 state
{baseTopic}/state/pumps/1/rpm     # Pump 1 RPM
{baseTopic}/state/temps/pool      # Pool temperature
{baseTopic}/config/circuit/1      # Circuit 1 config
```

3. **Discovery Support**
   - Home Assistant autodiscovery
   - Publishes device capabilities
   - Dynamic entity creation

4. **Retained Messages**
   - Last-value cache on broker
   - Survives client disconnect
   - Immediate state on connect

#### Configuration

```json
{
  "type": "mqtt",
  "enabled": true,
  "name": "MQTT",
  "config": {
    "broker": {
      "host": "mqtt.local",
      "port": 1883,
      "username": "user",
      "password": "pass"
    },
    "baseTopic": "pool",
    "retain": true,
    "qos": 1,
    "discovery": {
      "enabled": true,
      "topicPrefix": "homeassistant"
    }
  }
}
```

#### Published Events
- Circuit changes
- Temperature updates
- Pump speed/status
- Chlorinator output
- Chemistry readings
- Equipment alarms

#### Subscribed Topics
```
{baseTopic}/cmd/circuit/+/setState    # Turn circuits on/off
{baseTopic}/cmd/body/+/setPoint       # Set temperature
{baseTopic}/cmd/pump/+/rpm            # Set pump speed
```

---

### `influxInterface.ts`
**Purpose:** InfluxDB time-series data logging

**Class:** `InfluxInterfaceBindings`

#### Key Features

1. **Time-Series Storage**
   - Automatic data points
   - Configurable interval
   - Efficient batching

2. **Measurements**
   - `temperature` - Body temps, heater status
   - `pump` - RPM, watts, flow
   - `chlorinator` - Output, salt level
   - `chemistry` - pH, ORP, dosing

3. **Tags**
   - `equipment_id` - Identifies specific equipment
   - `equipment_name` - Human-readable name
   - `circuit_type` - Circuit function

4. **Fields**
   - Numeric values (temps, RPM, watts)
   - Boolean states (on/off, running)
   - String states (mode names)

#### Configuration

```json
{
  "type": "influx",
  "enabled": true,
  "name": "InfluxDB",
  "config": {
    "url": "http://influxdb.local:8086",
    "token": "my-token",
    "org": "my-org",
    "bucket": "pool",
    "interval": 30000,  // 30 seconds
    "includeCircuits": true,
    "includePumps": true,
    "includeTemps": true,
    "includeChlorinator": true,
    "includeChemistry": true
  }
}
```

#### Data Point Example

```javascript
{
  measurement: 'temperature',
  tags: {
    body_id: '1',
    body_name: 'Pool',
    location: 'main'
  },
  fields: {
    temp: 78.5,
    setPoint: 82,
    heaterOn: true,
    solarOn: false
  },
  timestamp: Date.now()
}
```

---

### `ruleInterface.ts`
**Purpose:** Custom automation rules engine

**Class:** `RuleInterfaceBindings`

#### Key Features

1. **Event-Driven Rules**
   - Triggers on equipment events
   - Conditional logic
   - Actions on multiple equipment

2. **Rule Structure**
```json
{
  "name": "Auto Spa Heat",
  "enabled": true,
  "trigger": {
    "type": "time",
    "time": "18:00"
  },
  "conditions": [
    {
      "equipment": "body",
      "id": 2,
      "property": "temp",
      "operator": "<",
      "value": 100
    }
  ],
  "actions": [
    {
      "equipment": "circuit",
      "id": 6,
      "action": "setState",
      "value": true
    },
    {
      "equipment": "body",
      "id": 2,
      "action": "setHeatMode",
      "value": 1
    }
  ]
}
```

3. **Trigger Types**
   - Time-based (schedule)
   - Temperature thresholds
   - Equipment state changes
   - Circuit activation
   - Manual trigger

4. **Condition Operators**
   - `=`, `!=`, `>`, `<`, `>=`, `<=`
   - `between`, `outside`
   - `is`, `not`

5. **Action Types**
   - Set circuit state
   - Set heat mode/setpoint
   - Set pump speed
   - Set chlorinator output
   - Send notification

---

### `httpInterface.ts`
**Purpose:** HTTP webhook integration

**Class:** `HttpInterfaceBindings`

#### Key Features

1. **Webhook Notifications**
   - POST equipment changes to URL
   - Configurable endpoints
   - Custom headers
   - JSON payload

2. **Event Filtering**
   - Subscribe to specific events
   - Reduce noise
   - Only relevant changes

#### Configuration

```json
{
  "type": "http",
  "enabled": true,
  "name": "Webhook",
  "config": {
    "url": "https://example.com/webhook",
    "method": "POST",
    "headers": {
      "Authorization": "Bearer token123"
    },
    "events": ["circuit", "temp", "pump"]
  }
}
```

---

## Bindings

Pre-configured integration templates in `/web/bindings/`

| File | Purpose |
|------|---------|
| `mqtt.json` | MQTT basic configuration |
| `mqttAlt.json` | Alternative MQTT structure |
| `homeassistant.json` | Home Assistant discovery |
| `smartThings-Hubitat.json` | SmartThings/Hubitat |
| `vera.json` | Vera home automation |
| `influxDB.json` | InfluxDB configuration |
| `rulesManager.json` | Rules engine config |
| `valveRelays.json` | Valve relay control |
| `aqualinkD.json` | AqualinkD integration |

### Custom Bindings

Location: `/web/bindings/custom/`

Users can create custom binding files for unique integrations. Format matches standard bindings.

---

## REMInterfaceServer

**Purpose:** Relay Equipment Manager integration

**Class:** `REMInterfaceServer`

#### Key Features
- Connects to REM server
- GPIO control
- I2C device communication
- Sensor integration
- Chemical dosing hardware

#### REM Equipment Types
- Atlas Scientific probes (pH, ORP, EC)
- Temperature sensors
- Flow sensors
- Pressure transducers
- Dosing pumps
- Relay boards

#### Communication
- Bidirectional Socket.IO
- Equipment status from REM
- Commands to REM hardware

---

## File Location Summary

| File/Folder | Purpose |
|-------------|---------|
| `Server.ts` | Main web server orchestrator |
| `services/config/` | Configuration REST API |
| `services/state/` | State control REST API |
| `services/utilities/` | Utility endpoints |
| `interfaces/` | External integration interfaces |
| `bindings/` | Pre-configured integration templates |

---

## API Examples

### REST API

**Get All Equipment State:**
```bash
curl http://localhost:4200/state/all
```

**Turn Circuit On:**
```bash
curl -X PUT http://localhost:4200/state/circuit/1/setState \
  -H "Content-Type: application/json" \
  -d '{"state": true}'
```

**Set Temperature:**
```bash
curl -X PUT http://localhost:4200/state/body/1/setPoint \
  -H "Content-Type: application/json" \
  -d '{"setPoint": 82}'
```

### Socket.IO

**JavaScript Client:**
```javascript
const io = require('socket.io-client');
const socket = io('http://localhost:4200');

// Listen for temperature updates
socket.on('body', (body) => {
  console.log(`Pool temp: ${body.temp}°F`);
});

// Turn circuit on
socket.emit('setCircuitState', { id: 1, state: true });
```

---

## Troubleshooting

**Issue:** Web server not starting
- Check port not already in use
- Verify IP address configuration
- Check SSL certificate paths (HTTPS)

**Issue:** MQTT not connecting
- Verify broker address and credentials
- Check network connectivity
- Review MQTT logs

**Issue:** InfluxDB data not appearing
- Verify bucket and token
- Check interval setting
- Review InfluxDB server logs

**Issue:** Socket.IO clients disconnecting
- Check firewall settings
- Verify CORS configuration
- Review client-side error logs
