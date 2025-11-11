# Development Guide

## Getting Started

### Prerequisites

**Required:**
- Node.js v20.0.0 or higher
- npm 8.0.0 or higher
- Git
- Text editor (VS Code recommended)

**Optional:**
- RS-485 adapter for hardware testing
- Docker for container testing
- Pool equipment (or use mock mode)

### Initial Setup

```bash
# Clone repository
git clone https://github.com/tagyoureit/nodejs-poolController.git
cd nodejs-poolController

# Install dependencies
npm install

# Build TypeScript
npm run build

# Run in mock mode (no hardware required)
# Edit config.json: set controller.comms.mockPort = true
npm start
```

---

## Development Workflow

### Quick Reference Commands

```bash
# Development cycle
npm run build          # Compile TypeScript
npm run watch          # Auto-compile on file changes
npm run start:cached   # Run without rebuilding

# Full cycle
npm start              # Build + run

# Testing
npm test               # Run test suite (if available)

# Linting
eslint .              # Check code quality
```

### Typical Development Loop

1. **Edit TypeScript files** in your editor
2. **Compile** with `npm run build` or use `npm run watch`
3. **Run** with `npm run start:cached`
4. **Test** your changes
5. **Check logs** in `/logs` folder
6. **Iterate**

### Hot Reload

Some changes can be hot-reloaded without restart:
- **Configuration changes** (`config.json`) - Auto-reload via file watcher
- **Code changes** - Require restart
- **Web client updates** - Refresh browser

---

## Project Structure

### Source Code Organization

```
/nodejs-poolController
├── app.ts                    # Entry point
├── config/                   # Configuration system
│   ├── Config.ts            # Config manager
│   └── VersionCheck.ts      # Update checker
├── logger/                   # Logging system
│   ├── Logger.ts            # Main logger
│   └── DataLogger.ts        # Time-series logger
├── controller/               # Core equipment logic
│   ├── Equipment.ts         # Equipment configuration
│   ├── State.ts             # Equipment state
│   ├── Constants.ts         # Enums and utilities
│   ├── Errors.ts            # Custom errors
│   ├── Lockouts.ts          # Timing delays
│   ├── boards/              # Controller implementations
│   │   ├── SystemBoard.ts   # Base class
│   │   ├── BoardFactory.ts  # Factory
│   │   ├── IntelliCenterBoard.ts
│   │   ├── IntelliTouchBoard.ts
│   │   ├── EasyTouchBoard.ts
│   │   └── ...
│   ├── comms/               # Communication layer
│   │   ├── Comms.ts         # RS-485 transport
│   │   ├── ScreenLogic.ts   # Network protocol
│   │   └── messages/        # Protocol codecs
│   │       ├── Messages.ts
│   │       ├── config/      # Configuration messages
│   │       └── status/      # Status messages
│   └── nixie/               # Standalone equipment
│       ├── Nixie.ts
│       ├── NixieEquipment.ts
│       ├── pumps/
│       ├── chemistry/
│       └── ...
├── web/                      # Web server & interfaces
│   ├── Server.ts            # Main web server
│   ├── services/            # REST API services
│   │   ├── config/          # Config API
│   │   ├── state/           # State API
│   │   └── utilities/       # Utility API
│   ├── interfaces/          # External integrations
│   │   ├── mqttInterface.ts
│   │   ├── influxInterface.ts
│   │   ├── ruleInterface.ts
│   │   └── ...
│   └── bindings/            # Integration templates
├── anslq25/                  # Mock/testing system
│   ├── MessagesMock.ts
│   ├── boards/
│   ├── chemistry/
│   └── pumps/
├── types/                    # TypeScript type definitions
├── dist/                     # Compiled JavaScript (generated)
├── data/                     # Runtime data (generated)
├── logs/                     # Log files (generated)
└── backups/                  # Backups (generated)
```

---

## TypeScript Guidelines

### Compilation Configuration

**tsconfig.json highlights:**
- Target: ES2020
- Module: CommonJS
- Strict: false (allowing gradual migration)
- Source maps: Enabled

### Type Safety

**Use types where available:**
```typescript
// Good
const pump: Pump = sys.pumps.getItemById(1);
const body: Body = sys.bodies.getItemById(1);

// Avoid
const pump: any = sys.pumps.getItemById(1);
```

**Define interfaces:**
```typescript
interface PumpConfig {
  id: number;
  name: string;
  type: number;
  circuits: number[];
}
```

### Async/Await

**Always use async/await for async operations:**
```typescript
// Good
async function setCircuitState(id: number, state: boolean) {
  await sys.board.setCircuitStateAsync(id, state);
}

// Avoid
function setCircuitState(id: number, state: boolean) {
  sys.board.setCircuitStateAsync(id, state).then(...);
}
```

---

## Adding New Features

### Adding a New Equipment Type

**Example: Adding a Water Feature**

1. **Define Equipment Class** (`controller/Equipment.ts`)
```typescript
export class WaterFeature extends EquipmentItem {
  public nozzleType: number;
  public flowRate: number;
  
  constructor() {
    super();
  }
  
  public get(bCopy?: boolean) {
    const obj = {
      id: this.id,
      name: this.name,
      nozzleType: this.nozzleType,
      flowRate: this.flowRate
    };
    return bCopy ? extend(true, {}, obj) : obj;
  }
}

export class WaterFeatureCollection extends EquipmentItemCollection {
  constructor(data: any) {
    super();
    if (typeof data !== 'undefined') {
      for (let i = 0; i < data.length; i++) {
        const obj = new WaterFeature();
        obj.id = data[i].id;
        obj.name = data[i].name;
        obj.nozzleType = data[i].nozzleType;
        obj.flowRate = data[i].flowRate;
        this.push(obj);
      }
    }
  }
}
```

2. **Add to PoolSystem Interface**
```typescript
interface IPoolSystem {
  // ... existing properties
  waterFeatures: WaterFeatureCollection;
}
```

3. **Add State Class** (`controller/State.ts`)
```typescript
export class WaterFeatureState {
  public id: number;
  public isOn: boolean;
  public currentFlow: number;
  
  public get() {
    return {
      id: this.id,
      isOn: this.isOn,
      currentFlow: this.currentFlow
    };
  }
}
```

4. **Add Board Method** (`controller/boards/SystemBoard.ts`)
```typescript
public async setWaterFeatureAsync(feature: WaterFeature, state: boolean) {
  // Create and queue outbound message
  const msg = new OutboundWaterFeatureMessage();
  msg.setPayload(feature.id, state);
  await conn.queueSendMessage(msg);
}
```

5. **Add REST API Endpoint** (`web/services/config/Config.ts`)
```typescript
app.get('/config/waterFeatures', async (req, res) => {
  const features = sys.waterFeatures.get();
  return res.status(200).send(features);
});

app.put('/config/waterFeature/:id', async (req, res) => {
  const feature = sys.waterFeatures.getItemById(req.params.id);
  // Update feature properties
  sys.persist();
  return res.status(200).send(feature.get());
});
```

6. **Add State Endpoint** (`web/services/state/State.ts`)
```typescript
app.put('/state/waterFeature/:id/setState', async (req, res) => {
  const feature = sys.waterFeatures.getItemById(req.params.id);
  await sys.board.setWaterFeatureAsync(feature, req.body.state);
  return res.status(200).send({ id: feature.id, state: req.body.state });
});
```

---

### Adding a New Controller Board

**Example: Adding support for a new controller**

1. **Create Board Class** (`controller/boards/NewControllerBoard.ts`)
```typescript
import { SystemBoard } from './SystemBoard';

export class NewControllerBoard extends SystemBoard {
  constructor(system: PoolSystem) {
    super(system);
    // Initialize controller-specific settings
  }
  
  public async setCircuitStateAsync(id: number, state: boolean) {
    // Implement controller-specific circuit control
    const msg = new OutboundCircuitMessage();
    msg.dest = 16;
    msg.source = 0;
    msg.action = 0x86;
    msg.setPayload(id, state ? 1 : 0);
    await conn.queueSendMessage(msg);
  }
  
  // Override other required methods...
}
```

2. **Add to BoardFactory** (`controller/boards/BoardFactory.ts`)
```typescript
export class BoardFactory {
  public static fromControllerType(type: ControllerType): SystemBoard {
    switch (type) {
      case ControllerType.NewController:
        return new NewControllerBoard(sys);
      // ... existing cases
      default:
        return new SystemBoard(sys);
    }
  }
}
```

3. **Add to ControllerType enum** (`controller/Constants.ts`)
```typescript
export enum ControllerType {
  IntelliCenter = 0,
  IntelliTouch = 1,
  // ... existing types
  NewController = 10,
  Unknown = 255
}
```

---

### Adding an External Interface

**Example: Adding Telegram Bot integration**

1. **Create Interface Class** (`web/interfaces/telegramInterface.ts`)
```typescript
import { InterfaceServer } from './baseInterface';

export class TelegramInterfaceBindings extends InterfaceServer {
  private bot: any;
  
  public async connect() {
    const TelegramBot = require('node-telegram-bot-api');
    this.bot = new TelegramBot(this.cfg.token, { polling: true });
    
    // Handle commands
    this.bot.onText(/\/status/, (msg) => {
      const status = state.getExtended();
      this.bot.sendMessage(msg.chat.id, JSON.stringify(status));
    });
    
    // Subscribe to state changes
    state.emitter.on('circuit', (circuit) => {
      this.emitData('circuit', circuit);
    });
  }
  
  public async disconnect() {
    if (this.bot) {
      await this.bot.stopPolling();
    }
  }
  
  public emitData(event: string, data: any) {
    // Send notification to Telegram
    const msg = `${event}: ${JSON.stringify(data)}`;
    // Send to configured chat IDs
  }
}
```

2. **Register in Server.ts** (`web/Server.ts`)
```typescript
import { TelegramInterfaceBindings } from './interfaces/telegramInterface';

// In initInterfaces() method
switch (iface.type) {
  case 'telegram':
    srv = new TelegramInterfaceBindings(iface);
    break;
  // ... existing cases
}
```

3. **Create Binding Template** (`web/bindings/telegram.json`)
```json
{
  "context": {
    "name": "Telegram Bot",
    "options": {}
  },
  "events": [
    {
      "name": "circuit",
      "description": "Circuit state changes"
    },
    {
      "name": "temp",
      "description": "Temperature updates"
    }
  ],
  "config": {
    "token": "YOUR_BOT_TOKEN",
    "chatIds": []
  }
}
```

---

## Debugging

### Enable Debug Logging

**config.json:**
```json
{
  "log": {
    "app": {
      "level": "debug"  // or "silly" for maximum verbosity
    }
  }
}
```

### Packet Logging

**Enable packet capture:**
```json
{
  "log": {
    "packet": {
      "enabled": true,
      "logToFile": true,
      "logToConsole": true
    }
  }
}
```

**View packet log:**
```bash
tail -f logs/packetLog*.log
```

### Using VS Code Debugger

**launch.json:**
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Launch Program",
      "skipFiles": ["<node_internals>/**"],
      "program": "${workspaceFolder}/dist/app.js",
      "preLaunchTask": "npm: build",
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "sourceMaps": true
    }
  ]
}
```

**Set breakpoints** in TypeScript source files, then press F5

### Common Issues

**Port Already in Use:**
```bash
# Find process using port 4200
lsof -i :4200
# Kill process
kill -9 <PID>
```

**Serial Port Permission Denied:**
```bash
# Add user to dialout group (Linux)
sudo usermod -a -G dialout $USER
# Reboot required
```

**TypeScript Compilation Errors:**
```bash
# Clean and rebuild
rm -rf dist/
npm run build
```

---

## Testing

### Manual Testing with Mock

1. **Enable mock mode** (`config.json`)
```json
{
  "controller": {
    "comms": {
      "mockPort": true
    }
  }
}
```

2. **Start application**
```bash
npm start
```

3. **Test API endpoints**
```bash
# Get all circuits
curl http://localhost:4200/config/circuits

# Turn circuit on
curl -X PUT http://localhost:4200/state/circuit/1/setState \
  -H "Content-Type: application/json" \
  -d '{"state": true}'

# Get current state
curl http://localhost:4200/state/circuit/1
```

### Testing with Real Hardware

1. **Configure RS-485 port** (`config.json`)
```json
{
  "controller": {
    "comms": {
      "rs485Port": "/dev/ttyUSB0",
      "mockPort": false
    }
  }
}
```

2. **Enable packet logging**
```json
{
  "log": {
    "packet": {
      "enabled": true,
      "logToFile": true
    }
  }
}
```

3. **Monitor packets**
```bash
tail -f logs/packetLog*.log
```

4. **Test commands and verify** equipment responds

---

## Code Quality

### ESLint

**Run linter:**
```bash
eslint .
```

**Auto-fix:**
```bash
eslint . --fix
```

### Code Style

**Indentation:** 4 spaces (or 2, be consistent)

**Naming:**
- Classes: PascalCase (`CircuitCollection`)
- Functions: camelCase (`setCircuitState`)
- Constants: UPPER_SNAKE_CASE (`MAX_CIRCUITS`)
- Private members: _leadingUnderscore (`_cfg`)

**Comments:**
```typescript
// Single-line comment

/**
 * Multi-line JSDoc comment
 * @param id - Circuit ID
 * @param state - On/off state
 * @returns Promise resolving to new state
 */
async function setCircuitState(id: number, state: boolean): Promise<boolean> {
  // Implementation
}
```

---

## Git Workflow

### Branching Strategy

- `master` - Stable production code
- `feature/new-feature` - Feature branches
- `bugfix/fix-issue` - Bug fix branches

### Commit Messages

**Format:**
```
<type>: <subject>

<body>

<footer>
```

**Types:**
- `feat` - New feature
- `fix` - Bug fix
- `docs` - Documentation
- `style` - Code style (formatting)
- `refactor` - Code refactoring
- `test` - Tests
- `chore` - Maintenance

**Example:**
```
feat: Add support for water feature equipment

Implements WaterFeature class and REST API endpoints
for controlling decorative water features.

Closes #123
```

### Pull Request Process

1. Fork repository
2. Create feature branch
3. Make changes
4. Test thoroughly
5. Commit with descriptive messages
6. Push to your fork
7. Create pull request
8. Respond to review feedback
9. Merge when approved

---

## Building & Deployment

### Building for Production

```bash
# Clean build
rm -rf dist/
npm run build

# Verify
ls -l dist/
```

### Docker Build

```bash
# Build image
docker build -t njspc .

# Run container
docker run -d \
  --name njspc \
  -p 4200:4200 \
  -v /dev/ttyUSB0:/dev/ttyUSB0 \
  -v $(pwd)/config.json:/app/config.json \
  njspc
```

### Systemd Service

**Create service file** `/etc/systemd/system/njspc.service`:
```ini
[Unit]
Description=nodejs-poolController
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/nodejs-poolController
ExecStart=/usr/bin/npm start
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**Enable and start:**
```bash
sudo systemctl enable njspc
sudo systemctl start njspc
sudo systemctl status njspc
```

---

## Resources

### Documentation
- [Main README](../README.md)
- [Architecture Overview](./architecture.md)
- [Contributing Guidelines](../CONTRIBUTING.md)
- [Wiki](https://github.com/tagyoureit/nodejs-poolController/wiki)

### Community
- [GitHub Discussions](https://github.com/tagyoureit/nodejs-poolController/discussions)
- [Issue Tracker](https://github.com/tagyoureit/nodejs-poolController/issues)

### External
- [Pentair Protocol Documentation](http://www.sdyoung.com/home/decoding-the-pentair-easytouch-rs-485-protocol)
- [RS-485 Basics](https://en.wikipedia.org/wiki/RS-485)
- [Socket.IO Documentation](https://socket.io/docs/)
- [Express.js Documentation](https://expressjs.com/)

---

## Tips & Tricks

### Rapid Iteration

Use two terminal windows:
```bash
# Terminal 1: Watch mode
npm run watch

# Terminal 2: Run (restart manually)
npm run start:cached
# Ctrl+C, up arrow, enter to restart quickly
```

### Testing Protocol Changes

1. Enable packet capture
2. Make change
3. Restart application
4. Reproduce scenario
5. Stop capture
6. Review packets in ZIP file

### Debugging State Issues

Add temporary logging:
```typescript
state.emitter.on('circuit', (circuit) => {
  console.log('Circuit changed:', circuit);
});
```

### Configuration Experiments

Keep backup:
```bash
cp config.json config.json.backup
# Experiment
# Restore if needed
cp config.json.backup config.json
```

---

## Contribution Checklist

Before submitting a pull request:

- [ ] Code compiles without errors (`npm run build`)
- [ ] No ESLint warnings (`eslint .`)
- [ ] Tested with mock equipment
- [ ] Tested with real equipment (if applicable)
- [ ] Documentation updated
- [ ] Commit messages are clear
- [ ] No sensitive data in code
- [ ] Follows project coding style
- [ ] Related issues referenced in PR

---

## Getting Help

If you're stuck:
1. Check documentation in `/docs`
2. Search GitHub issues
3. Ask in GitHub Discussions
4. Join community chat (if available)
5. Create detailed issue with logs
