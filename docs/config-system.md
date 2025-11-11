# Configuration System Documentation

## Overview

The configuration system manages all application settings with support for hot-reloading, environment variable overrides, and automatic backup/recovery.

**Location:** `/config` folder

---

## Files

### `Config.ts`
**Purpose:** Core configuration management singleton

**Class:** `Config`

#### Key Features

1. **Multi-Layer Configuration Merging**
   - Default settings from `defaultConfig.json`
   - User overrides from `config.json`
   - Environment variable overrides (POOL_* prefix)
   - Runtime version injection from `package.json`

2. **File Watching & Hot Reload**
   - Monitors `config.json` for changes
   - Automatically reloads configuration
   - Debounced to prevent rapid reloads
   - Triggers logger reinitialization on log config changes

3. **Corruption Handling**
   - Detects invalid JSON in `config.json`
   - Backs up corrupt files to `config.corrupt-<timestamp>.json`
   - Falls back to defaults automatically
   - Logs warnings for operator awareness

4. **Empty File Handling**
   - Auto-populates empty `config.json` with defaults
   - Prevents initialization failures

#### Key Methods

**`init()`**
- Initializes configuration system
- Loads and merges configuration layers
- Sets up file watcher
- Called once during application startup

**`getSection(section: string, defaultValue?: any): any`**
- Retrieves a configuration section
- Supports nested path notation (e.g., 'web.servers.http')
- Returns deep clone (prevents accidental mutation)
- Returns defaultValue if section doesn't exist

**`setSection(section: string, value: any)`**
- Updates a configuration section
- Uses nested path notation
- Triggers file watcher debounce
- Does NOT automatically persist to disk

**`updateAsync(callback?: Function): Promise<void>`**
- Asynchronously writes configuration to `config.json`
- Uses semaphore (`_isLoading`) to prevent concurrent writes
- Pretty-prints JSON with 2-space indentation
- Invokes callback on completion/error

**`removeSection(section: string)`**
- Removes a configuration section
- Uses nested path notation
- Requires manual `updateAsync()` to persist

**`getEnvVariables(): object`**
- Extracts environment variable overrides
- Maps POOL_* variables to config paths

#### Environment Variable Mapping

| Environment Variable | Config Path | Description |
|---------------------|-------------|-------------|
| `POOL_NET_CONNECT` | `controller.comms.netConnect` | Enable network connection |
| `POOL_NET_HOST` | `controller.comms.netHost` | Network host address |
| `POOL_NET_PORT` | `controller.comms.netPort` | Network port |
| `POOL_RS485_PORT` | `controller.comms.rs485Port` | Serial port path |
| `POOL_LATITUDE` | `location.latitude` | Geographic latitude |
| `POOL_LONGITUDE` | `location.longitude` | Geographic longitude |
| `POOL_LOG_LEVEL` | `log.app.level` | Logging verbosity |

**Example:**
```bash
export POOL_NET_CONNECT=true
export POOL_NET_HOST=192.168.1.100
export POOL_NET_PORT=9801
npm start
```

#### Configuration Structure

**controller:**
```json
{
  "comms": {
    "rs485Port": "/dev/ttyUSB0",
    "netConnect": false,
    "netHost": "raspberrypi.local",
    "netPort": 9801,
    "mockPort": false,
    "portSettings": {
      "baudRate": 9600,
      "dataBits": 8,
      "parity": "none",
      "stopBits": 1,
      "flowControl": false
    },
    "inactivityRetry": 10
  },
  "txDelays": {
    "idleBeforeTxMs": 40,
    "interFrameDelayMs": 50,
    "interByteDelayMs": 0
  },
  "screenlogic": {
    "type": "local",
    "address": "Pentair: xx-xx-xx"
  }
}
```

**web:**
```json
{
  "servers": {
    "http": {
      "enabled": true,
      "ip": "0.0.0.0",
      "port": 4200,
      "httpsRedirect": false
    },
    "https": {
      "enabled": false,
      "ip": "0.0.0.0",
      "port": 4201,
      "sslKeyFile": "path/to/key.pem",
      "sslCertFile": "path/to/cert.pem"
    },
    "mdns": {
      "enabled": true,
      "name": "nodejs-poolController"
    },
    "ssdp": {
      "enabled": true
    }
  },
  "interfaces": [
    {
      "type": "mqtt",
      "enabled": false,
      "uuid": "generated-uuid",
      "name": "MQTT",
      "config": { /* MQTT settings */ }
    }
  ]
}
```

**log:**
```json
{
  "app": {
    "enabled": true,
    "level": "info",
    "logToFile": false,
    "captureForReplay": false
  },
  "packet": {
    "enabled": false,
    "logToConsole": false,
    "logToFile": false
  }
}
```

#### Internal State

- `cfgPath` - Path to config.json
- `_cfg` - Merged configuration object
- `_isInitialized` - Initialization flag
- `_fileTime` - Last modification time (for change detection)
- `_isLoading` - Semaphore for preventing concurrent writes
- `emitter` - EventEmitter for configuration change events

#### Events

**`reloaded`**
- Emitted after configuration file is reloaded
- Payload: Updated configuration object

**Usage:**
```typescript
config.emitter.on('reloaded', (newConfig) => {
  console.log('Config reloaded', newConfig);
});
```

---

### `VersionCheck.ts`
**Purpose:** Automatic version update checking

**Class:** `VersionChecker`

#### Key Features

1. **GitHub API Integration**
   - Queries GitHub releases API
   - Compares local version with latest release
   - Detects beta/pre-release versions

2. **Git Detection**
   - Checks if running from git repository
   - Executes `git fetch` and `git status` to detect updates
   - Prefers git-based checks when available

3. **Throttling**
   - Checks at configurable intervals (default: daily)
   - Prevents excessive API calls
   - Respects GitHub rate limits

4. **Safe HTTP Handling**
   - Validates redirects
   - Handles SSL/TLS securely
   - Timeouts prevent hanging

5. **Warning Suppression**
   - Can be disabled via configuration
   - Useful for air-gapped/offline deployments

#### Key Methods

**`check()`**
- Initiates version check
- Runs on configurable timer
- Logs warnings if updates available

**`getLatestVersion(): Promise<string>`**
- Fetches latest version from GitHub
- Handles API responses
- Caches result

**`compareVersions(current: string, latest: string): number`**
- Semantic version comparison
- Returns -1 (behind), 0 (current), or 1 (ahead)

#### Configuration

```json
{
  "versionCheck": {
    "enabled": true,
    "interval": 86400000,  // 24 hours in ms
    "suppressWarnings": false
  }
}
```

---

## Exported Singleton

```typescript
import { config } from './config/Config';

// Get configuration section
const webConfig = config.getSection('web');

// Update configuration
config.setSection('web.servers.http.port', 4200);
await config.updateAsync();

// Remove section
config.removeSection('web.interfaces.mqtt');
await config.updateAsync();
```

---

## Configuration Best Practices

### 1. Always Use Getters/Setters
❌ **Wrong:**
```typescript
let cfg = config.getSection('web');
cfg.servers.http.port = 4200;  // Changes lost!
```

✅ **Right:**
```typescript
let cfg = config.getSection('web');
cfg.servers.http.port = 4200;
config.setSection('web', cfg);
await config.updateAsync();
```

### 2. Use Environment Variables for Docker
```bash
# docker-compose.yml
environment:
  - POOL_NET_CONNECT=true
  - POOL_NET_HOST=192.168.1.100
  - POOL_LATITUDE=40.7128
  - POOL_LONGITUDE=-74.0060
```

### 3. Batch Configuration Changes
```typescript
// Inefficient: Multiple writes
config.setSection('web.servers.http.port', 4200);
await config.updateAsync();
config.setSection('web.servers.http.ip', '0.0.0.0');
await config.updateAsync();

// Efficient: Single write
let web = config.getSection('web');
web.servers.http.port = 4200;
web.servers.http.ip = '0.0.0.0';
config.setSection('web', web);
await config.updateAsync();
```

### 4. Handle Missing Sections Gracefully
```typescript
const section = config.getSection('optional.feature', { enabled: false });
// Always returns an object, never undefined
```

---

## File Location Summary

| File | Location | Purpose |
|------|----------|---------|
| `Config.ts` | `/config/Config.ts` | Configuration manager singleton |
| `VersionCheck.ts` | `/config/VersionCheck.ts` | Version update checker |
| `defaultConfig.json` | `/defaultConfig.json` | Default settings template |
| `config.json` | `/config.json` | User settings (auto-created) |

---

## Troubleshooting

**Issue:** Configuration changes not persisting
- **Cause:** Forgot to call `updateAsync()`
- **Solution:** Always call `await config.updateAsync()` after `setSection()`

**Issue:** Config file corrupt after crash
- **Cause:** Application terminated during write
- **Solution:** Check for `config.corrupt-*.json` backups

**Issue:** Hot reload not working
- **Cause:** File watcher not established
- **Solution:** Ensure file system supports file watching (some Docker volumes don't)

**Issue:** Environment variables ignored
- **Cause:** Incorrect variable names or timing
- **Solution:** Use exact POOL_* names, set before `npm start`
