# Architecture Overview

## System Architecture

nodejs-poolController follows a layered architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                     External Clients                         │
│  (dashPanel, Home Automation, MQTT, InfluxDB, Mobile Apps)  │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ HTTP/WS/MQTT
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      Web Layer                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────────┐ │
│  │  HTTP    │  │ Socket   │  │   External Interfaces    │ │
│  │  Server  │  │  .IO     │  │  (MQTT, InfluxDB, REM)   │ │
│  └──────────┘  └──────────┘  └──────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         REST API Routes & Services                    │  │
│  │    (Config, State, Utilities)                         │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ Events
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    State Management Layer                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  State.ts - Real-time Equipment State                │  │
│  │  • Change detection with Proxy pattern               │  │
│  │  • Event emission to web clients                     │  │
│  │  • Persistence to poolState.json                     │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ Updates
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Equipment Management Layer                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Equipment.ts (sys) - Equipment Configuration        │  │
│  │  • Bodies, Circuits, Pumps, Heaters, etc.           │  │
│  │  • Collections and persistence                       │  │
│  │  • Board-specific logic delegation                   │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  SystemBoard - Controller Type Abstraction           │  │
│  │  • IntelliCenter, IntelliTouch, EasyTouch, etc.     │  │
│  │  • Protocol-specific implementations                 │  │
│  │  • Value maps and feature availability              │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Nixie - Standalone Equipment (No OCP)              │  │
│  │  • Direct equipment control via GPIO/relays          │  │
│  │  • Software-based schedules and automation           │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ Messages
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Communication Layer                         │
│  ┌────────────────────────┐  ┌─────────────────────────┐   │
│  │  Comms.ts (conn)       │  │ ScreenLogic.ts (sl)     │   │
│  │  • RS-485 transport    │  │ • Network protocol      │   │
│  │  • Multi-port support  │  │ • Alternative to RS-485 │   │
│  │  • Message queuing     │  │ • Auto-discovery        │   │
│  │  • Collision avoidance │  │                         │   │
│  └────────────────────────┘  └─────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Messages - Protocol Encoding/Decoding               │  │
│  │  • Inbound: Equipment → Application                  │  │
│  │  • Outbound: Application → Equipment                 │  │
│  │  • Config messages, Status messages                  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ Serial/Network
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Physical Hardware                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ RS-485   │  │ Screen   │  │  Mock    │  │  GPIO    │   │
│  │ Adapter  │  │ Logic    │  │  Port    │  │  (REM)   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ Protocol
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Pool Equipment                            │
│  (IntelliCenter, Pumps, Heaters, Chlorinators, etc.)       │
└─────────────────────────────────────────────────────────────┘

        Cross-Cutting Concerns (Used by all layers)
┌─────────────────────────────────────────────────────────────┐
│  Configuration (config)  │  Logger (logger)  │  Utils       │
└─────────────────────────────────────────────────────────────┘
```

---

## Data Flow

### Startup Sequence

1. **Configuration Initialization**
   - Load `defaultConfig.json`
   - Merge with `config.json`
   - Apply environment variable overrides
   - Initialize file watcher

2. **Logger Initialization**
   - Create log directories
   - Configure Winston transports
   - Start packet capture if enabled

3. **Equipment System Initialization**
   - Load `/data/poolConfig.json`
   - Instantiate board via `BoardFactory`
   - Initialize equipment collections
   - Set up Nixie equipment (if applicable)

4. **State System Initialization**
   - Load `/data/poolState.json`
   - Create proxied state objects
   - Initialize heliotrope (sun position calculator)

5. **Web Server Initialization**
   - Start HTTP/HTTPS servers
   - Configure Socket.IO
   - Set up REST API routes
   - Initialize external interfaces (MQTT, InfluxDB, etc.)
   - Start MDNS/SSDP discovery

6. **Communications Initialization**
   - Open RS-485 ports
   - Connect ScreenLogic (if configured)
   - Start message pumps
   - Begin polling equipment

7. **Equipment Startup**
   - Request configuration from equipment
   - Poll for current status
   - Synchronize clocks
   - Activate schedules

8. **Post-Startup**
   - Initialize automatic backups
   - Start version checking
   - System ready for client connections

### Command Flow (User Action → Equipment)

```
1. User clicks "Turn on Pool Light" in dashPanel

2. dashPanel sends Socket.IO event:
   emit('setCircuitState', { id: 1, state: true })

3. StateSocket.ts handler receives event:
   → Validates request
   → Calls sys.board.setCircuitStateAsync(1, true)

4. Board implementation (e.g., IntelliCenterBoard):
   → Creates Outbound message
   → Encodes circuit command
   → Queues message

5. Comms.ts message pump:
   → Waits for idle window (txDelays)
   → Transmits bytes via RS-485
   → Awaits response

6. Equipment (IntelliCenter) receives command:
   → Processes request
   → Changes circuit relay state
   → Sends acknowledgment

7. Comms.ts receives response:
   → Creates Inbound message
   → Decodes response
   → Routes to board

8. Board processes response:
   → Updates equipment configuration (if changed)
   → Updates state (state.circuits[1].isOn = true)

9. State change detection:
   → Proxy triggers dirty flag
   → Emits 'circuit' event

10. WebServer broadcasts:
    → emitToClients('circuit', { id: 1, isOn: true })
    → All connected clients receive update
    → MQTT publishes to broker
    → InfluxDB logs data point
    → Rules engine evaluates triggers

11. dashPanel receives update:
    → UI reflects new state
    → Shows circuit as ON
```

### Status Polling Flow

```
1. Equipment periodically broadcasts status (e.g., every 10 seconds)

2. Comms.ts receives bytes via RS-485:
   → Validates checksum
   → Creates Inbound message
   → Decodes message type

3. Message routed to appropriate handler:
   → EquipmentStateMessage
   → PumpStateMessage
   → HeaterStateMessage
   → etc.

4. Handler updates state:
   → state.pumps[1].rpm = 2400
   → state.temps.pool.temp = 78.5
   → state.chlorinator.currentOutput = 50

5. State change detection:
   → Proxy detects changes
   → Marks dirty
   → Schedules persistence

6. Event emission:
   → WebServer broadcasts changes
   → External interfaces notified

7. Persistence:
   → After 3-second debounce
   → Writes /data/poolState.json
   → Atomic write (temp + rename)

8. Clients receive updates:
   → Real-time UI updates
   → MQTT state sync
   → InfluxDB data logging
```

---

## Key Design Patterns

### 1. Singleton Pattern
Used for core system components:
- `config` - Configuration management
- `logger` - Logging system
- `sys` - Equipment system
- `state` - State management
- `conn` - Communications
- `webApp` - Web server
- `sl` - ScreenLogic

**Rationale:** Single instance needed, globally accessible, lazy initialization

### 2. Factory Pattern
`BoardFactory` creates appropriate board implementation:
```typescript
BoardFactory.fromControllerType(ControllerType.IntelliCenter)
  → returns IntelliCenterBoard instance
```

**Rationale:** Encapsulates board instantiation, easy to add new controller types

### 3. Strategy Pattern
`SystemBoard` defines interface, subclasses implement:
- Different message handling per controller type
- Controller-specific features
- Protocol variations

**Rationale:** Varies behavior by controller type, maintains consistent interface

### 4. Observer Pattern
EventEmitter-based change notification:
- State changes emit events
- Web layer subscribes
- External interfaces react

**Rationale:** Loose coupling, multiple observers, reactive architecture

### 5. Proxy Pattern
JavaScript Proxy for change detection:
```typescript
const proxiedState = new Proxy(state, handler);
// Automatically detects property changes
```

**Rationale:** Transparent change tracking, no manual dirty flagging

### 6. Template Method Pattern
`SystemBoard` provides template, subclasses override:
```typescript
class SystemBoard {
  async setCircuitStateAsync(id, state) {
    // Common validation
    this.doSetCircuitState(id, state);  // Override point
    // Common cleanup
  }
}
```

**Rationale:** Reuse common logic, customize specific steps

### 7. Repository Pattern
Equipment collections (bodies, circuits, pumps):
```typescript
sys.circuits.getItemById(1);
sys.pumps.get();
sys.bodies.find(b => b.name === 'Pool');
```

**Rationale:** Centralized data access, consistent interface, encapsulated queries

---

## Concurrency & Timing

### Message Queue
- Single outbound queue per port
- FIFO with priority support
- Retry logic with exponential backoff
- Collision detection and avoidance

### State Persistence Debouncing
- Changes marked as "dirty"
- 3-second delay before write
- Multiple changes batched
- Prevents excessive disk I/O

### Configuration Updates
- Semaphore prevents concurrent writes
- File watcher debounced
- Atomic writes (temp + rename)

### Transmit Pacing (txDelays)
- `idleBeforeTxMs` - Wait for bus quiet
- `interFrameDelayMs` - Gap between messages
- `interByteDelayMs` - Byte-by-byte spacing (rare)

### Polling Cycles
- Equipment status polled every 10-30 seconds
- Configurable per controller type
- Reduced during inactivity

---

## Error Handling

### Custom Error Types
- `EquipmentNotFoundError` - Invalid equipment ID
- `InvalidEquipmentDataError` - Bad configuration data
- `InvalidOperationError` - Unsupported operation
- `OutboundMessageError` - Communication failure
- `BoardProcessError` - Protocol processing error

### Error Recovery Strategies

**Communication Errors:**
- Auto-reconnect on disconnect
- Message retry with backoff
- Timeout and move to next message

**Configuration Errors:**
- Validate before applying
- Restore from backup if corrupt
- Fall back to defaults

**State Errors:**
- Refresh from equipment
- Mark state as stale
- Warn user of inconsistency

---

## Security Considerations

### Authentication
- Optional HTTP basic auth
- `.htpasswd` file support
- Username/password validation

### SSL/TLS
- HTTPS support
- Certificate management
- Auto-redirect from HTTP

### Input Validation
- All API inputs validated
- Type checking on messages
- Range validation on values

### Configuration Protection
- Sensitive data in environment variables
- File permissions on config files
- No credentials in logs

### Network Security
- Configurable listen IPs
- Firewall-friendly ports
- No unnecessary services

---

## Performance Characteristics

### Throughput
- ~50-100 messages/second (RS-485)
- Limited by physical bus speed (9600 baud)
- Transmit pacing reduces collision overhead

### Latency
- Command to equipment: 100-500ms
- State update to clients: <50ms
- API response time: <10ms

### Memory Usage
- Base: ~100-150 MB
- Packet capture: +10-50 MB
- Scales with equipment count

### Disk I/O
- Config writes: Infrequent (manual changes)
- State writes: Every 3 seconds (when dirty)
- Log writes: Buffered, periodic flush

### CPU Usage
- Idle: <5%
- Active communication: 10-20%
- Web traffic: Scales with clients

---

## Scalability

### Equipment Limits
- Circuits: 40+ (controller dependent)
- Pumps: 16
- Bodies: 4
- Schedules: 99
- Chlorinators: 4 (with REM)

### Client Connections
- Socket.IO: 100+ concurrent clients
- REST API: Limited by Node.js/Express
- MQTT: Offloaded to broker

### Multi-Port Support
- Primary + multiple auxiliary RS-485 ports
- Separate chlorinators per port
- Dual chemistry controllers

---

## Extensibility Points

### Adding New Controller Types
1. Create `NewControllerBoard.ts`
2. Extend `SystemBoard`
3. Implement required methods
4. Add to `BoardFactory`
5. Update `ControllerType` enum

### Adding External Interfaces
1. Create `newInterface.ts`
2. Extend `InterfaceServer`
3. Implement `connect()` and `emitData()`
4. Register in `Server.ts initInterfaces()`
5. Add binding template JSON

### Adding Equipment Types
1. Define equipment class in `Equipment.ts`
2. Add collection to `PoolSystem`
3. Create state class in `State.ts`
4. Add message handlers
5. Update board implementations

### Adding API Endpoints
1. Create route in appropriate service
2. Add validation logic
3. Call board methods
4. Return JSON response
5. Document in API reference

---

## Technology Stack

### Core
- **Node.js 20+** - Runtime environment
- **TypeScript 4.9** - Type-safe JavaScript
- **Express.js 4** - Web framework

### Communication
- **SerialPort 11** - RS-485 communication
- **Socket.IO 4** - WebSocket library
- **MQTT.js 4** - MQTT client

### Storage
- **File System** - Configuration and state
- **InfluxDB Client** - Time-series database

### Networking
- **multicast-dns** - mDNS (Bonjour)
- **node-ssdp** - SSDP (UPnP)
- **node-screenlogic** - Pentair protocol

### Logging
- **Winston 3** - Logging framework
- **Custom** - Packet capture system

### Utilities
- **extend** - Deep object merging
- **jszip** - Archive creation
- **multer** - File upload handling

---

## Deployment Architectures

### Standard Deployment
```
Raspberry Pi (or similar)
├─ nodejs-poolController (Port 4200)
├─ dashPanel UI (Port 5150)
├─ RS-485 Adapter (/dev/ttyUSB0)
└─ Pool Equipment (RS-485 bus)
```

### Docker Deployment
```
Docker Host
├─ njspc container
│  ├─ Application
│  ├─ Named volumes (data, logs, backups)
│  └─ Device mapping (/dev/ttyUSB0)
└─ njspc-dash container (optional)
```

### Network-Based (ScreenLogic)
```
Pool Equipment
└─ ScreenLogic Bridge (Network)
    └─ nodejs-poolController (Any server)
        ├─ Local network access
        └─ No RS-485 adapter needed
```

### Home Automation Integration
```
nodejs-poolController
├─ MQTT Interface → MQTT Broker
│  └─ Home Assistant / Hubitat / SmartThings
├─ HTTP Interface → Webhook endpoint
└─ InfluxDB Interface → Grafana dashboards
```

---

## Future Architecture Considerations

### Microservices
Potential split into separate services:
- Core controller service
- Web API service
- Interface gateway service

### Message Queue
Replace direct calls with message queue:
- Redis/RabbitMQ for scaling
- Async command processing
- Better failure handling

### Database Backend
Move from file-based to database:
- PostgreSQL for configuration
- TimescaleDB for time-series
- Better query capabilities

### Containerization
- Smaller, specialized containers
- Kubernetes orchestration
- Better resource management

### API Gateway
- Rate limiting
- Authentication/Authorization
- API versioning
- Load balancing
