# File-by-File Documentation

This document provides a quick reference for what each file in the repository does.

## Root Level Files

| File | Purpose |
|------|---------|
| `app.ts` | Application entry point, initializes all systems in sequence, handles shutdown |
| `sendSocket.js` | CLI utility to send Socket.IO commands to running instance |
| `package.json` | NPM package definition, dependencies, build scripts |
| `tsconfig.json` | TypeScript compiler configuration |
| `defaultConfig.json` | Default configuration template (don't edit) |
| `config.json` | User configuration (auto-created, gitignored) |
| `Dockerfile` | Container build instructions |
| `docker-compose.yml` | Multi-container orchestration |
| `Gruntfile.js` | Legacy build automation (mostly replaced by npm scripts) |
| `.eslintrc.json` | ESLint code quality rules |
| `.gitignore` | Files/folders to exclude from Git |
| `.npmrc` | NPM configuration |
| `README.md` | Main project documentation |
| `CONTRIBUTING.md` | Contribution guidelines |
| `LICENSE` | AGPL-3.0 license text |
| `Changelog` | Version history and release notes |

---

## /config

Configuration management system.

| File | Class/Export | Purpose |
|------|-------------|---------|
| `Config.ts` | `Config` class → `config` singleton | Manages application configuration, file watching, hot-reload, environment overrides |
| `VersionCheck.ts` | `VersionChecker` class → `versionCheck` singleton | Checks for application updates from GitHub, git status detection |

---

## /logger

Logging and data capture system.

| File | Class/Export | Purpose |
|------|-------------|---------|
| `Logger.ts` | `Logger` class → `logger` singleton | Winston-based application logging, packet capture, replay mode |
| `DataLogger.ts` | `DataLogger` class | Time-series data logging for equipment metrics |

---

## /controller

Core equipment management and protocol communication.

| File | Class/Export | Purpose |
|------|-------------|---------|
| `Equipment.ts` | `PoolSystem` class → `sys` singleton | Equipment configuration, collections (bodies, circuits, pumps, etc.), persistence to poolConfig.json |
| `State.ts` | `State` class → `state` singleton | Real-time equipment state, change detection with Proxy, event emission, persistence to poolState.json |
| `Constants.ts` | Enums, `utils` object, `byteValueMap` | Shared constants, enumerations (ControllerType), utility functions, value mapping |
| `Errors.ts` | Custom error classes | `EquipmentNotFoundError`, `InvalidEquipmentDataError`, `InvalidOperationError`, etc. |
| `Lockouts.ts` | `DelayManager` class → `delayMgr` singleton | Equipment change delays, interlocks, timing constraints |

---

## /controller/boards

Controller-specific implementations.

| File | Class | Purpose |
|------|-------|---------|
| `SystemBoard.ts` | `SystemBoard` (abstract) | Base class for all controller boards, defines interface, common logic |
| `BoardFactory.ts` | `BoardFactory` (static) | Factory to instantiate correct board type based on ControllerType |
| `IntelliCenterBoard.ts` | `IntelliCenterBoard` | IntelliCenter-specific protocol implementation |
| `IntelliTouchBoard.ts` | `IntelliTouchBoard` | IntelliTouch-specific protocol implementation |
| `EasyTouchBoard.ts` | `EasyTouchBoard` | EasyTouch-specific protocol implementation |
| `SunTouchBoard.ts` | `SunTouchBoard` | SunTouch-specific protocol implementation |
| `IntelliComBoard.ts` | `IntelliComBoard` | IntelliCom-specific protocol implementation |
| `AquaLinkBoard.ts` | `AquaLinkBoard` | AquaLink-specific protocol implementation |
| `NixieBoard.ts` | `NixieBoard` | Standalone/Nixie mode (no OCP) implementation |

---

## /controller/comms

Communication layer for RS-485 and ScreenLogic.

| File | Class/Export | Purpose |
|------|-------------|---------|
| `Comms.ts` | `Connection` → `conn` singleton, `RS485Port` | RS-485 serial communication, multi-port support, message queuing, transmit pacing |
| `ScreenLogic.ts` | `ScreenLogicComms` → `sl` singleton | ScreenLogic network protocol, alternative to RS-485, auto-discovery |

---

## /controller/comms/messages

Protocol message encoding/decoding.

| File | Class | Purpose |
|------|-------|---------|
| `Messages.ts` | `Message`, `Inbound`, `Outbound`, `Response` | Base classes for protocol messages, checksum calculation, byte manipulation |

### /controller/comms/messages/config

Configuration messages (equipment setup).

| File | Message Type | Purpose |
|------|-------------|---------|
| `CircuitMessage.ts` | Circuit config | Circuit name, function, delay settings |
| `CircuitGroupMessage.ts` | Circuit group config | Multi-circuit groups, light groups |
| `PumpMessage.ts` | Pump config | Pump model, circuits, RPM/GPM curves |
| `HeaterMessage.ts` | Heater config | Heater type, body association, priorities |
| `ChlorinatorMessage.ts` | Chlorinator config | Model, body assignment, output limits |
| `ScheduleMessage.ts` | Schedule config | Start/end times, days, circuit associations |
| `CustomNameMessage.ts` | Custom names | User-defined equipment names |
| `FeatureMessage.ts` | Feature config | Feature circuits (spillover, jets, etc.) |
| `ValveMessage.ts` | Valve config | Valve assignments, delays |
| `OptionsMessage.ts` | General options | System-wide settings |
| `SecurityMessage.ts` | Security settings | Equipment lock codes |
| `RemoteMessage.ts` | Remote config | Wireless remote configuration |
| `EquipmentMessage.ts` | Equipment inventory | Installed equipment detection |
| `GeneralMessage.ts` | General settings | Time, date, location |
| `IntellichemMessage.ts` | IntelliChem config | Chemistry controller settings |
| `CoverMessage.ts` | Cover config | Automatic pool cover settings |
| `ExternalMessage.ts` | External equipment | Additional equipment configuration |
| `ConfigMessage.ts` | General config | Catch-all for other config messages |

### /controller/comms/messages/status

Status messages (current equipment state).

| File | Message Type | Purpose |
|------|-------------|---------|
| `EquipmentStateMessage.ts` | Equipment status | Overall system status, circuits, bodies |
| `PumpStateMessage.ts` | Pump status | Current RPM, watts, flow, errors |
| `HeaterStateMessage.ts` | Heater status | On/off, temperature, operation mode |
| `ChlorinatorStateMessage.ts` | Chlorinator status | Current output, salt level, warnings |
| `IntelliChemStateMessage.ts` | IntelliChem status | pH, ORP, dosing status, alarms |
| `IntelliValveStateMessage.ts` | Valve status | Current valve position |
| `VersionMessage.ts` | Version info | Firmware version, model numbers |
| `RegalModbusStateMessage.ts` | Regal pump status | Modbus pump communication |

---

## /controller/nixie

Standalone equipment control (no Pentair OCP).

| File | Class/Export | Purpose |
|------|-------------|---------|
| `Nixie.ts` | `NixieControlPanel` → `ncp` singleton | Main Nixie controller, manages standalone equipment |
| `NixieEquipment.ts` | Various Nixie equipment classes | Base classes for Nixie-controlled equipment |

### /controller/nixie subdirectories

| Folder | Purpose |
|--------|---------|
| `pumps/` | Pump.ts - Standalone pump control (relay, VS pumps) |
| `chemistry/` | Chlorinator.ts, ChemController.ts, ChemDoser.ts - Chemistry equipment |
| `heaters/` | Heater.ts - Heater control without OCP |
| `circuits/` | Circuit.ts - GPIO/relay circuit control |
| `valves/` | Valve.ts - Valve actuators |
| `bodies/` | Body.ts, Filter.ts - Body and filter management |
| `schedules/` | Schedule.ts - Software-based schedule execution |

---

## /web

Web server, REST API, and external interfaces.

| File | Class/Export | Purpose |
|------|-------------|---------|
| `Server.ts` | `WebServer` → `webApp` singleton, `REMInterfaceServer` | Main web server orchestrator, HTTP/HTTPS/HTTP2 servers, Socket.IO, MDNS/SSDP, interface management |

---

## /web/services

REST API route handlers.

### /web/services/config

| File | Class | Purpose |
|------|-------|---------|
| `Config.ts` | `ConfigRoute` | REST API endpoints for equipment configuration (GET/PUT/POST/DELETE) |
| `ConfigSocket.ts` | `ConfigSocket` | Socket.IO event handlers for configuration changes |

### /web/services/state

| File | Class | Purpose |
|------|-------|---------|
| `State.ts` | `StateRoute` | REST API endpoints for equipment state and control |
| `StateSocket.ts` | `StateSocket` | Socket.IO event handlers for state control |

### /web/services/utilities

| File | Class | Purpose |
|------|-------|---------|
| `Utilities.ts` | `UtilitiesRoute` | Utility endpoints (version, backup/restore, logs, bindings) |

---

## /web/interfaces

External integration interfaces.

| File | Class | Purpose |
|------|-------|---------|
| `baseInterface.ts` | `InterfaceServer` (abstract) | Base class for all external interfaces |
| `mqttInterface.ts` | `MqttInterfaceBindings` | MQTT broker integration, bidirectional communication |
| `influxInterface.ts` | `InfluxInterfaceBindings` | InfluxDB time-series data logging |
| `ruleInterface.ts` | `RuleInterfaceBindings` | Custom automation rules engine |
| `httpInterface.ts` | `HttpInterfaceBindings` | HTTP webhook notifications |

---

## /web/bindings

Pre-configured integration templates (JSON files).

| File | Purpose |
|------|---------|
| `mqtt.json` | Basic MQTT configuration template |
| `mqttAlt.json` | Alternative MQTT structure |
| `homeassistant.json` | Home Assistant autodiscovery |
| `smartThings-Hubitat.json` | SmartThings/Hubitat integration |
| `vera.json` | Vera home automation |
| `influxDB.json` | InfluxDB configuration |
| `rulesManager.json` | Rules engine configuration |
| `valveRelays.json` | Valve relay control |
| `aqualinkD.json` | AqualinkD integration |

---

## /anslq25

Mock/simulated equipment for testing.

| File | Class | Purpose |
|------|-------|---------|
| `MessagesMock.ts` | `MockMessage` | Mock message generation and handling |

### /anslq25/boards

| File | Class | Purpose |
|------|-------|---------|
| `MockSystemBoard.ts` | `MockSystemBoard` | Base class for mock boards |
| `MockBoardFactory.ts` | `MockBoardFactory` | Factory for mock board instantiation |
| `MockEasyTouchBoard.ts` | `MockEasyTouchBoard` | Mock EasyTouch controller |

### /anslq25 equipment

| Folder | Purpose |
|--------|---------|
| `chemistry/MockChlorinator.ts` | Simulated chlorinator with salt levels, output |
| `pumps/MockPump.ts` | Simulated variable speed pump with power curve |

---

## /types

TypeScript type definitions.

| File | Purpose |
|------|---------|
| `express-multer.d.ts` | Type definitions for Express with Multer file uploads |

---

## /.github

GitHub-specific files.

| File | Purpose |
|------|---------|
| `copilot-instructions.md` | Instructions for AI-assisted development |
| `workflows/ghcr-publish.yml` | GitHub Actions workflow to build and publish Docker images |
| `ISSUE_TEMPLATE/` | Issue templates (bug report, docs, proposal) |

---

## Generated/Runtime Directories

| Directory | Contents | Gitignored |
|-----------|----------|------------|
| `/dist/` | Compiled JavaScript from TypeScript | Yes |
| `/node_modules/` | NPM dependencies | Yes |
| `/data/` | poolConfig.json, poolState.json, data logs | Yes |
| `/logs/` | Application logs, packet logs, capture archives | Yes |
| `/backups/` | Automatic configuration backups | Yes |
| `/web/bindings/custom/` | User-created custom bindings | Yes |

---

## Quick File Finder

Looking for specific functionality? Use this guide:

**Want to...**
- **Add new equipment type** → Edit `controller/Equipment.ts` and `controller/State.ts`
- **Change protocol handling** → Look in `controller/comms/messages/`
- **Add new controller board** → Create new file in `controller/boards/`
- **Add REST API endpoint** → Edit files in `web/services/`
- **Add external integration** → Create new file in `web/interfaces/`
- **Modify logging** → Edit `logger/Logger.ts`
- **Change configuration system** → Edit `config/Config.ts`
- **Modify web server** → Edit `web/Server.ts`
- **Add mock equipment** → Create files in `anslq25/`
- **Change startup sequence** → Edit `app.ts`

---

## File Count Summary

- **Root:** 15 files
- **config/:** 2 files
- **logger/:** 2 files  
- **controller/:** 5 core files + boards + comms + nixie
- **controller/boards/:** 9 files
- **controller/comms/:** 2 files + messages
- **controller/comms/messages/:** 20+ message files
- **controller/nixie/:** 10+ equipment files
- **web/:** 1 main file
- **web/services/:** 6 files
- **web/interfaces/:** 5 files
- **web/bindings/:** 9 JSON templates
- **anslq25/:** 5+ mock files
- **types/:** 1 file
- **.github/:** 2+ files
- **docs/:** 9 documentation files (this project!)

**Total:** ~80 TypeScript/JavaScript source files + config/docs

---

## See Also

- [Architecture Overview](./architecture.md) - System architecture and design patterns
- [Development Guide](./development.md) - How to develop and extend
- [Root Files](./root-files.md) - Detailed root-level file descriptions
- [Config System](./config-system.md) - Configuration management deep dive
- [Logger System](./logger-system.md) - Logging and capture details
- [Controller System](./controller-system.md) - Equipment and communication
- [Web System](./web-system.md) - Web server and interfaces
- [Mock System](./anslq25-system.md) - Testing without hardware
