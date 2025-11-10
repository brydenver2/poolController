# Controller System Documentation

## Overview

The controller system is the core of nodejs-poolController, managing equipment discovery, protocol communication, state management, and hardware-specific board implementations.

**Location:** `/controller` folder

---

## Core Files

### `Equipment.ts`
**Purpose:** Central equipment management and configuration persistence

**Class:** `PoolSystem` (exported as `sys` singleton)

#### Key Responsibilities
- Equipment configuration storage (`/data/poolConfig.json`)
- Collection management (bodies, circuits, pumps, heaters, etc.)
- Board instantiation via `BoardFactory`
- Configuration persistence with change tracking
- Equipment discovery and initialization

#### Equipment Collections

| Collection | Purpose | Examples |
|-----------|---------|----------|
| `bodies` | Pool/spa bodies | Main pool, spa |
| `circuits` | On/off circuits | Lights, fountains, cleaners |
| `features` | Feature circuits | Spillover, jets |
| `pumps` | Variable speed pumps | IntelliFlo, relay pumps |
| `chlorinators` | Chlorine generators | IntelliChlor, Aqua-Rite |
| `heaters` | Heating equipment | Gas, solar, heat pump |
| `chemControllers` | Chemistry automation | IntelliChem, REM |
| `schedules` | Timed events | Pump schedules, circuit timers |
| `valves` | Valve actuators | IntelliValve |
| `filters` | Filter cycles | Filter pump runtime |
| `circuitGroups` | Multi-circuit groups | Light groups, sync circuits |
| `covers` | Pool covers | Automatic covers |
| `remotes` | Remote controls | Wireless controllers |

#### Key Methods

**`init()`**
- Loads configuration from `/data/poolConfig.json`
- Creates default config if file missing
- Initializes board via `BoardFactory`
- Sets up equipment collections

**`start()`**
- Starts communication with equipment
- Begins polling cycles
- Initiates equipment discovery

**`persist()`**
- Saves configuration to disk
- Debounced to prevent excessive writes
- Atomic write with temp file + rename

**`stopAsync()`**
- Graceful shutdown
- Stops all polling
- Closes connections
- Saves final state

---

### `State.ts`
**Purpose:** Real-time equipment state management and event broadcasting

**Class:** `State` (exported as `state` singleton)

#### Key Responsibilities
- Current equipment state (`/data/poolState.json`)
- State change detection with Proxy pattern
- Event emission to web interfaces
- Dirty state tracking for persistence
- Temperature and time calculations

#### State Collections

All equipment collections have corresponding state objects:
- `bodies` → temperature, heat status
- `circuits` → on/off state
- `pumps` → RPM, watts, flow
- `chlorinators` → output level, salt level
- `chemControllers` → pH, ORP, dosing status
- `equipment` → connection status, errors

#### Key Methods

**`init()`**
- Loads state from `/data/poolState.json`
- Creates proxied state objects for change detection
- Initializes heliotrope (sunrise/sunset calculator)

**`persist()`**
- Saves current state to disk
- Debounced (3 second delay)
- Only writes if dirty flag set

**`emitEquipmentChange()`**
- Emits state changes to web clients
- Includes only changed data
- Triggers MQTT, InfluxDB, etc.

**`getExtended()`**
- Returns complete state snapshot
- Includes computed values
- Used by API responses

#### Change Detection

Uses JavaScript Proxy pattern to automatically track changes:
```typescript
// Setting a value automatically marks state as dirty
state.pumps.getItemById(1).rpm = 2400;  // Triggers dirty flag
// State auto-persists after 3 seconds
```

---

### `Constants.ts`
**Purpose:** Shared enumerations, utilities, and value maps

#### Key Exports

**`ControllerType` enum:**
- `IntelliCenter`
- `IntelliTouch`
- `EasyTouch`
- `IntelliCom`
- `SunTouch`
- `AquaLink`
- `Nixie` (standalone)
- `Unknown`

**`utils` object:**
- `uuid()` - Generate UUIDs for config
- `formatDuration()` - Human-readable time
- `makeBool()` - Parse boolean values
- `parseCommaList()` - Parse CSV strings

**`byteValueMap` class:**
- Maps numeric values to descriptive objects
- Used for protocol enumerations
- Example: `0 = { val: 0, name: 'off', desc: 'Off' }`

**Value Maps:**
- `sys.board.valueMaps.circuitFunctions` - Circuit types
- `sys.board.valueMaps.heatModes` - Heating modes
- `sys.board.valueMaps.heatSources` - Heat source types
- `sys.board.valueMaps.pumpTypes` - Pump models
- Many more...

---

### `Errors.ts`
**Purpose:** Custom error types for specific failure scenarios

#### Error Classes

**`EquipmentNotFoundError`**
- Equipment ID not found in collections
- Used when API requests reference non-existent equipment

**`InvalidEquipmentDataError`**
- Invalid data provided for equipment
- Used during configuration validation

**`InvalidEquipmentIdError`**
- Equipment ID out of valid range
- Used during equipment creation

**`BoardProcessError`**
- Board-specific processing failure
- Used in protocol handlers

**`InvalidOperationError`**
- Operation not supported by current equipment
- Used when attempting incompatible actions

**`OutboundMessageError`**
- Failed to send message to equipment
- Used in communication layer

---

### `Lockouts.ts`
**Purpose:** Timing constraints and equipment interlocks

**Class:** `DelayManager` (exported as `delayMgr` singleton)

#### Key Features

1. **Equipment Change Delays**
   - Prevents rapid changes to same equipment
   - Configurable delay periods
   - Prevents equipment damage

2. **Interlock Management**
   - Prevents conflicting operations
   - Example: Can't heat spa while pool heating
   - Enforces equipment dependencies

3. **Startup Delays**
   - Staggers equipment startup
   - Prevents power surges
   - Smooth initialization

#### Key Methods

**`setDelay(key: string, delayMs: number)`**
- Sets a delay for specific equipment/operation
- Prevents actions until delay expires

**`hasDelay(key: string): boolean`**
- Checks if delay is active
- Returns true if operation should be blocked

**`clearDelay(key: string)`**
- Manually clears a delay
- Used when equipment reset needed

---

## Communication Layer

### `comms/Comms.ts`
**Purpose:** RS-485 serial communication management

**Class:** `Connection` (exported as `conn` singleton)

#### Key Features

1. **Multi-Port Support**
   - Primary RS-485 port (portId: 0)
   - Additional auxiliary ports (portId: 1, 2, ...)
   - Each port has independent configuration

2. **Transport Types**
   - Serial (USB/RS-485 adapter)
   - Network (Socat, remote serial servers)
   - Mock (simulated for testing)

3. **Connection Management**
   - Auto-reconnect on failure
   - Inactivity detection
   - Port health monitoring

4. **Message Queue**
   - Outbound message prioritization
   - Retry logic with exponential backoff
   - Collision detection and avoidance

5. **Transmit Pacing (txDelays)**
   - `idleBeforeTxMs` - Quiet time before transmit
   - `interFrameDelayMs` - Delay between frames
   - `interByteDelayMs` - Delay between bytes (for slow hardware)

#### Key Classes

**`RS485Port`**
- Represents a single physical port
- Manages SerialPort instance
- Handles open/close/write operations
- Tracks connection statistics

**`Connection`**
- Manages collection of RS485Ports
- Routes messages to appropriate port
- Coordinates multi-port scenarios

#### Key Methods

**`initAsync()`**
- Opens configured RS-485 ports
- Establishes connections
- Starts message pumps

**`setPortAsync(data: any)`**
- Configures RS-485 port
- Updates settings dynamically
- Persists to configuration

**`deleteAuxPort(data: any)`**
- Removes auxiliary port
- Cannot delete primary port (id: 0)

**`writeAsync(msg: Outbound, portId: number)`**
- Sends message to equipment
- Queues if port busy
- Returns Promise for response

**`queueSendMessage(msg: Outbound)`**
- Adds message to outbound queue
- Priority handling
- Auto-retry on failure

#### Message Flow

```
API/State Change
    ↓
Outbound Message Created
    ↓
Added to Message Queue
    ↓
Wait for Idle Window (txDelays)
    ↓
Transmit to RS-485
    ↓
Await Response (or timeout)
    ↓
Process Inbound Message
    ↓
Update Equipment/State
    ↓
Emit to Web Clients
```

---

### `comms/ScreenLogic.ts`
**Purpose:** ScreenLogic protocol support (alternative to RS-485)

**Class:** `ScreenLogicComms` (exported as `sl` singleton)

#### Key Features

1. **Network Communication**
   - Connects via TCP to ScreenLogic bridge
   - Local or remote connection types
   - Auto-discovery via MDNS

2. **Protocol Translation**
   - Translates ScreenLogic messages to internal format
   - Bidirectional communication
   - Status polling

3. **Reduced Feature Set**
   - Less granular than RS-485
   - Primary/fallback connection
   - Used when RS-485 not available

#### Key Methods

**`openAsync()`**
- Establishes ScreenLogic connection
- Discovers devices if local type
- Connects to specified address if remote

**`closeAsync()`**
- Closes ScreenLogic connection
- Cleanup resources

**`sendAsync(message: any)`**
- Sends command via ScreenLogic
- Returns Promise for response

---

### `comms/messages/Messages.ts`
**Purpose:** Protocol message encoding/decoding

#### Key Classes

**`Message`**
- Base class for all messages
- Common header/footer handling
- Checksum calculation

**`Inbound`**
- Received from equipment
- Decodes bytes to structured data
- Updates equipment/state

**`Outbound`**
- Sent to equipment
- Encodes structured data to bytes
- Queued for transmission

**`Response`**
- Expected response to outbound
- Timeout handling
- Success/failure indication

#### Message Structure

**Standard Pentair Protocol:**
```
[Header][Dest][Source][Action][Length][...Data...][Checksum]
 0xA5 0x00   1byte    1byte    1byte   1byte    N bytes    2bytes
```

**Example:**
```
Message: Set circuit 1 ON
Bytes: [A5, 00, 10, 00, 86, 02, 01, 01, 01, 3A]
       [Header][Dest][Src][Action][Len][Circuit][State][Checksum]
```

#### Message Types

Located in `/controller/comms/messages/config/` and `/controller/comms/messages/status/`:

**Config Messages:**
- `CircuitMessage` - Circuit configuration
- `PumpMessage` - Pump settings
- `HeaterMessage` - Heater settings
- `ChlorinatorMessage` - Chlorinator config
- `ScheduleMessage` - Schedule configuration
- `GeneralMessage` - General settings

**Status Messages:**
- `EquipmentStateMessage` - Overall equipment status
- `PumpStateMessage` - Pump current state
- `HeaterStateMessage` - Heater current state
- `ChlorinatorStateMessage` - Chlorinator status
- `IntelliChemStateMessage` - Chemistry controller status

---

## Board Implementations

### `boards/SystemBoard.ts`
**Purpose:** Abstract base class for all controller boards

**Class:** `SystemBoard`

#### Key Responsibilities
- Protocol-specific message handling
- Equipment validation
- Value map definitions
- Feature availability by controller type

#### Key Methods (Must Override)

**`checkConfiguration()`**
- Validates equipment configuration
- Returns list of errors/warnings

**`requestConfiguration(equip?: any, ver?: ConfigVersion)`**
- Requests equipment config from controller
- Queues necessary messages

**`processStatusMessage(msg: Inbound)`**
- Processes incoming status messages
- Updates equipment state

**`setCircuitStateAsync(id: number, state: boolean)`**
- Turns circuit on/off
- Board-specific implementation

**`setHeatModeAsync(body: Body, mode: number)`**
- Sets heating mode
- Validates mode for board type

#### Available Boards

Located in `/controller/boards/`:

| Board Class | Controller Type | Features |
|------------|----------------|----------|
| `IntelliCenterBoard` | IntelliCenter | Full feature set, dual equipment |
| `IntelliTouchBoard` | IntelliTouch | Full feature set |
| `EasyTouchBoard` | EasyTouch | Standard features |
| `SunTouchBoard` | SunTouch | Limited features |
| `IntelliComBoard` | IntelliCom | Two-body system |
| `AquaLinkBoard` | AquaLink | Basic features |
| `NixieBoard` | Nixie | Standalone mode, no OCP |

### `boards/BoardFactory.ts`
**Purpose:** Factory for instantiating correct board type

**Function:** `BoardFactory.fromControllerType(type: ControllerType)`

Selects board implementation based on detected/configured controller type:
```typescript
switch(type) {
  case ControllerType.IntelliCenter:
    return new IntelliCenterBoard(sys);
  case ControllerType.IntelliTouch:
    return new IntelliTouchBoard(sys);
  // ... etc
}
```

---

## Nixie System (Standalone Equipment)

### `nixie/Nixie.ts`
**Purpose:** Standalone equipment control without OCP (Outdoor Control Panel)

**Class:** `NixieControlPanel` (exported as `ncp` singleton)

#### Key Features
- Direct equipment control without Pentair controller
- GPIO/relay integration via REM
- Software-based schedules and logic
- Custom chemistry control

#### Supported Equipment (Standalone)
- Relay-controlled pumps
- Standalone chlorinators
- Temperature sensors
- GPIO circuits (lights, valves, etc.)

#### Nixie Equipment Classes

Located in `/controller/nixie/`:
- `nixie/pumps/Pump.ts` - Pump control
- `nixie/chemistry/Chlorinator.ts` - Chlorinator control
- `nixie/chemistry/ChemController.ts` - Chemistry automation
- `nixie/heaters/Heater.ts` - Heater control
- `nixie/circuits/Circuit.ts` - Circuit control
- `nixie/valves/Valve.ts` - Valve actuators
- `nixie/bodies/Body.ts` - Body management
- `nixie/schedules/Schedule.ts` - Schedule execution

---

## File Location Summary

| File/Folder | Purpose |
|-------------|---------|
| `Equipment.ts` | Equipment configuration and collections |
| `State.ts` | Real-time equipment state |
| `Constants.ts` | Enumerations and utilities |
| `Errors.ts` | Custom error types |
| `Lockouts.ts` | Timing delays and interlocks |
| `comms/Comms.ts` | RS-485 communication |
| `comms/ScreenLogic.ts` | ScreenLogic protocol |
| `comms/messages/` | Protocol message codecs |
| `boards/` | Controller-specific implementations |
| `nixie/` | Standalone equipment support |

---

## Usage Examples

### Get Equipment Configuration
```typescript
import { sys } from './controller/Equipment';

// Get all circuits
const circuits = sys.circuits.get();

// Get specific pump
const pump1 = sys.pumps.getItemById(1);
console.log(pump1.name, pump1.type.name);
```

### Change Equipment State
```typescript
import { sys, state } from './controller';

// Turn circuit on
await sys.board.setCircuitStateAsync(1, true);

// Set pump speed
await sys.board.setPumpFlowAsync(pump, 30);  // 30 GPM

// Set heat mode
const body = sys.bodies.getItemById(1);
await sys.board.setHeatModeAsync(body, 1);  // Heater mode
```

### Monitor State Changes
```typescript
import { state } from './controller/State';

// Listen for temperature changes
state.emitter.on('body', (body) => {
  console.log(`Body ${body.id} temp: ${body.temp}`);
});

// Listen for circuit changes
state.emitter.on('circuit', (circuit) => {
  console.log(`Circuit ${circuit.id} is ${circuit.isOn ? 'on' : 'off'}`);
});
```

---

## Troubleshooting

**Issue:** Equipment not discovered
- Check RS-485 connections
- Verify port configuration
- Enable packet logging to see traffic

**Issue:** Commands not working
- Check board type matches physical controller
- Verify equipment IDs are correct
- Check for lockouts/delays

**Issue:** State not updating
- Verify polling is active
- Check connection status
- Look for communication errors in logs
