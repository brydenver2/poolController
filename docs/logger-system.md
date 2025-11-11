# Logger System Documentation

## Overview

The logger system provides comprehensive logging, packet capture, and time-series data recording capabilities using Winston as the underlying logging framework.

**Location:** `/logger` folder

---

## Files

### `Logger.ts`
**Purpose:** Application-wide logging with packet capture and replay functionality

**Class:** `Logger`

#### Key Features

1. **Multi-Level Logging**
   - Standard log levels: error, warn, info, verbose, debug, silly
   - Configurable per-transport (console, file)
   - Winston-based formatting with timestamps

2. **Packet Capture**
   - Records all RS-485 protocol messages
   - Optional ScreenLogic message capture
   - Timestamped packet logs
   - Replay capability for debugging

3. **Console-to-File Logging**
   - Mirrors console output to file
   - Useful for Docker/systemd deployments
   - Separate from packet logs

4. **Capture-for-Replay Mode**
   - Records complete communication sessions
   - Includes configuration snapshots
   - Packages into ZIP archives
   - Used for bug reproduction and protocol analysis

#### Key Methods

**`init()`**
- Initializes Winston logger
- Creates `/logs` directory if needed
- Sets up transports (console, file)
- Starts packet capture if configured
- Called during application startup

**`log(level: string, message: string)`**
- Main logging method
- Levels: 'error', 'warn', 'info', 'verbose', 'debug', 'silly'
- Formats with timestamp and level

**`packet(msg: Message)`**
- Logs RS-485 protocol packets
- Buffers packets for batch writing
- Only logs if packet logging enabled or capture mode active
- Flushes to file periodically

**`startCaptureForReplay(reset: boolean)`**
- Enables capture mode
- Creates timestamped capture directory
- Records packets, config, and state
- Optionally clears existing packet buffers

**`stopCaptureForReplayAsync(): Promise<string>`**
- Stops capture mode
- Packages captured data into ZIP
- Returns path to ZIP file
- Automatically called during shutdown

**`slCapture(msg: any)`**
- Captures ScreenLogic protocol messages
- Separate from RS-485 packets
- Used when ScreenLogic is primary connection

#### Configuration

**`log.app` section:**
```json
{
  "enabled": true,
  "level": "info",           // error|warn|info|verbose|debug|silly
  "logToFile": false,        // Mirror console to file
  "captureForReplay": false  // Enable full packet capture
}
```

**`log.packet` section:**
```json
{
  "enabled": false,          // Log packets to console/file
  "logToConsole": false,     // Show packets in console
  "logToFile": false         // Write packets to file
}
```

#### Log File Locations

- **Packet Logs:** `/logs/packetLog(YYYY-MM-DD_HH-MM-SS).log`
- **Console Logs:** `/logs/consoleLog(YYYY-MM-DD_HH-MM-SS).log`
- **Capture Archives:** `/logs/YYYY-MM-DD_HH-MM-SS/packetCapture.zip`

#### Packet Buffer

- Buffers packets in memory before flushing to disk
- Default flush interval: 500ms (configurable)
- Prevents excessive I/O during high packet traffic
- Automatically flushed during shutdown

#### Capture-for-Replay Format

**Directory Structure:**
```
/logs/YYYY-MM-DD_HH-MM-SS/
  ├── packetCapture.json       # Array of captured packets
  ├── screenlogicCapture.json  # ScreenLogic messages (if used)
  ├── config.json              # Application config snapshot
  ├── poolConfig.json          # Equipment config snapshot
  ├── poolState.json           # Equipment state snapshot
  └── packetCapture.zip        # Archive of above
```

**packetCapture.json format:**
```json
[
  {
    "timestamp": "2025-01-10T12:34:56.789Z",
    "direction": "in",  // or "out"
    "packet": [165, 0, 16, ...],  // Byte array
    "decoded": {
      "header": [165, 0],
      "dest": 16,
      "source": 0,
      // ... message fields
    }
  }
]
```

#### Usage Example

```typescript
import { logger } from './logger/Logger';

// Standard logging
logger.info('Application started');
logger.debug('Debug information');
logger.error('Error occurred', error);

// Packet logging (internal use)
logger.packet(message);

// Start capture for bug report
await startPacketCapture(true);  // From app.ts
// ... reproduce issue ...
const zipPath = await logger.stopCaptureForReplayAsync();
console.log(`Capture saved to: ${zipPath}`);
```

---

### `DataLogger.ts`
**Purpose:** Time-series data logging for equipment metrics

**Class:** `DataLogger`

#### Key Features

1. **Equipment State Logging**
   - Periodic snapshots of equipment state
   - Configurable logging interval
   - JSON format for easy parsing

2. **Historical Tracking**
   - Temperature readings
   - Chemical levels
   - Pump speeds
   - Circuit states
   - Heater status

3. **InfluxDB Integration**
   - Optionally sends data to InfluxDB
   - Structured as time-series points
   - Supports tags and fields

#### Key Methods

**`log(entry: DataLoggerEntry)`**
- Records a timestamped data entry
- Appends to current log file
- Rotates files at midnight

**`getEntries(start: Date, end: Date): DataLoggerEntry[]`**
- Retrieves historical entries
- Date range filtering
- Returns array of entries

#### DataLoggerEntry Structure

```typescript
interface DataLoggerEntry {
  timestamp: Date;
  equipment: {
    bodies: {
      temp: number;
      setPoint: number;
      heaterOn: boolean;
    }[];
    pumps: {
      rpm: number;
      watts: number;
      flow: number;
    }[];
    chlorinator: {
      currentOutput: number;
      targetOutput: number;
      saltLevel: number;
    };
    chemistry: {
      pH: number;
      orp: number;
      saturationIndex: number;
    };
  };
}
```

#### Log File Format

**Location:** `/data/dataLog-YYYY-MM-DD.json`

**Format:**
```json
[
  {
    "timestamp": "2025-01-10T12:00:00.000Z",
    "bodies": [
      {
        "id": 1,
        "temp": 78.5,
        "setPoint": 82,
        "heaterOn": true
      }
    ],
    "pumps": [
      {
        "id": 1,
        "rpm": 1800,
        "watts": 450
      }
    ]
  }
]
```

---

## Exported Singleton

```typescript
import { logger } from './logger/Logger';

// Application logging
logger.info('Starting pump');
logger.warn('Temperature sensor not responding');
logger.error('Communication error', error);

// Debug logging
logger.debug('Received message', { msg: decodedMessage });
logger.silly('Detailed trace information');
```

---

## Logging Best Practices

### 1. Use Appropriate Log Levels

**Error:** System failures, exceptions, critical issues
```typescript
logger.error('Failed to start communications', error);
```

**Warn:** Recoverable issues, configuration problems
```typescript
logger.warn('Temperature sensor reading invalid, using default');
```

**Info:** Normal operations, state changes
```typescript
logger.info('Pool temperature reached setpoint');
```

**Verbose:** Detailed operational information
```typescript
logger.verbose('Sending pump configuration', { rpm: 2400 });
```

**Debug:** Development/troubleshooting information
```typescript
logger.debug('Decoded message', { source: 16, dest: 0, action: 134 });
```

**Silly:** Extremely detailed trace information
```typescript
logger.silly('Byte-by-byte packet analysis', { bytes: [...] });
```

### 2. Include Context in Log Messages

❌ **Poor:**
```typescript
logger.error('Failed');
```

✅ **Good:**
```typescript
logger.error('Failed to set pump speed', { pumpId: 1, requestedRpm: 2400, error: err.message });
```

### 3. Packet Logging for Protocol Issues

When reporting protocol issues:
1. Enable packet logging: `log.packet.enabled = true`
2. Reproduce the issue
3. Share packet log file

For complex issues:
1. Enable capture mode: `log.app.captureForReplay = true`
2. Reproduce issue
3. Stop capture (creates ZIP)
4. Share ZIP file

### 4. Log Rotation

- Packet logs: New file per application start
- Console logs: New file per application start
- Data logs: New file per day (midnight rotation)
- Manual cleanup recommended for old logs

### 5. Performance Considerations

- Packet logging has minimal impact (buffered writes)
- 'silly' level can generate large logs
- Capture mode records everything (disk space!)
- Disable packet logging in production unless debugging

---

## Debugging Workflow

### Diagnosing Communication Issues

1. **Enable Packet Logging**
```json
{
  "log": {
    "packet": {
      "enabled": true,
      "logToFile": true,
      "logToConsole": false
    }
  }
}
```

2. **Restart Application**
```bash
npm start
```

3. **Check Packet Log**
```bash
cat logs/packetLog(*.log) | grep -A5 "ERROR"
```

### Creating Bug Report with Capture

1. **Enable Capture Mode**
```bash
# Edit config.json
{
  "log": {
    "app": {
      "captureForReplay": true
    }
  }
}
```

2. **Reproduce Issue**

3. **Stop Capture**
- Application automatically creates ZIP on shutdown
- Or use API: `GET /app/stop/capture`

4. **Locate ZIP**
```bash
ls -lh logs/*/packetCapture.zip
```

5. **Share ZIP File**
- Upload to GitHub issue
- Includes all necessary diagnostic data

---

## File Location Summary

| File | Location | Purpose |
|------|----------|---------|
| `Logger.ts` | `/logger/Logger.ts` | Main logging singleton |
| `DataLogger.ts` | `/logger/DataLogger.ts` | Time-series data logger |
| Packet Logs | `/logs/packetLog(timestamp).log` | Protocol packet logs |
| Console Logs | `/logs/consoleLog(timestamp).log` | Application output |
| Data Logs | `/data/dataLog-YYYY-MM-DD.json` | Historical metrics |
| Captures | `/logs/timestamp/packetCapture.zip` | Complete diagnostic package |

---

## Troubleshooting

**Issue:** Logs not being written
- **Cause:** `/logs` directory missing or permissions
- **Solution:** Check directory exists and is writable

**Issue:** Log files growing too large
- **Cause:** 'silly' log level in production
- **Solution:** Use 'info' or 'warn' level, implement log rotation

**Issue:** Packet capture missing data
- **Cause:** Capture not started before issue occurred
- **Solution:** Enable `captureForReplay` before reproducing

**Issue:** ScreenLogic packets not captured
- **Cause:** ScreenLogic capture separate from RS-485
- **Solution:** Check `screenlogicCapture.json` in capture archive
