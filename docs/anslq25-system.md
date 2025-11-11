# Mock/Testing System Documentation (anslq25)

## Overview

The anslq25 folder contains mock/simulated pool equipment for testing nodejs-poolController without physical hardware. Named after a protocol action byte, this system enables development and testing of protocol implementations.

**Location:** `/anslq25` folder

---

## Purpose

1. **Hardware-Free Development**
   - Test protocol implementations without RS-485 adapter
   - Develop features without pool equipment
   - Safe experimentation with dangerous operations

2. **Protocol Testing**
   - Validate message encoding/decoding
   - Test edge cases and error conditions
   - Capture protocol sequences

3. **Continuous Integration**
   - Automated testing in CI/CD pipelines
   - Regression testing
   - Protocol compliance verification

4. **Training & Demos**
   - Learn system without equipment risk
   - Demonstrate features
   - Onboard new developers

---

## Files

### `MessagesMock.ts`
**Purpose:** Mock message generation and handling

**Class:** `MockMessage`

#### Key Features
- Simulates inbound messages from equipment
- Responds to outbound commands
- Timing-accurate responses
- Realistic protocol behavior

#### Key Methods

**`createResponse(outbound: Outbound): Inbound`**
- Generates appropriate response to command
- Mimics real equipment behavior
- Includes realistic delays

**`generateStatusMessage(msgType: number): Inbound`**
- Creates periodic status updates
- Simulates polling responses
- Equipment state reflection

---

## Boards

Mock board implementations simulate different controller types.

### `boards/MockSystemBoard.ts`
**Purpose:** Abstract base for mock boards

**Class:** `MockSystemBoard extends SystemBoard`

#### Key Features
- Overrides base `SystemBoard` methods
- Simulates equipment responses
- No actual RS-485 communication
- In-memory state management

#### Simulated Behaviors
- Circuit state changes
- Temperature readings
- Pump speed adjustments
- Heater activation
- Schedule execution
- Chemistry readings

#### Timing
- Uses `setTimeout` for realistic delays
- Simulates communication latency
- Gradual temperature changes
- Pump ramp-up/down

---

### `boards/MockBoardFactory.ts`
**Purpose:** Factory for instantiating mock boards

**Function:** `MockBoardFactory.fromControllerType(type: ControllerType)`

Similar to regular `BoardFactory` but returns mock implementations:
```typescript
switch(type) {
  case ControllerType.EasyTouch:
    return new MockEasyTouchBoard(sys);
  // ... etc
}
```

---

### `boards/MockEasyTouchBoard.ts`
**Purpose:** Mock EasyTouch controller

**Class:** `MockEasyTouchBoard extends MockSystemBoard`

#### Simulated Features
- 8 circuits (lights, pump, cleaner, etc.)
- 2 bodies (pool and spa)
- Single pump
- Gas heater
- Basic chlorinator

#### Response Patterns
- Acknowledges circuit commands
- Reports temperature changes
- Pump status updates
- Configuration responses

---

## Equipment Mocks

### `chemistry/MockChlorinator.ts`
**Purpose:** Simulated chlorinator behavior

**Class:** `MockChlorinator`

#### Simulated Features
- Salt level reading (2800-3400 ppm)
- Output percentage (0-100%)
- Cell status
- Flow detection
- Temperature compensation

#### Behaviors
- Responds to output commands
- Simulates salt consumption
- Reports low salt warnings
- Temperature-based adjustments

#### Methods

**`processSetOutput(percent: number)`**
- Validates percentage range
- Updates output state
- Generates status message

**`generateStatus(): Buffer`**
- Creates status message packet
- Includes current readings
- Realistic value fluctuations

---

### `pumps/MockPump.ts`
**Purpose:** Simulated variable speed pump

**Class:** `MockPump`

#### Simulated Features
- RPM control (400-3450 RPM)
- Power consumption calculation
- Flow rate estimation
- Run time tracking

#### Behaviors
- Ramps speed gradually
- Calculates watts from RPM
- Reports errors (priming, overload)
- Flow sensor readings

#### Power Curve
Realistic power consumption based on RPM:
```typescript
// Simplified curve
watts = (rpm / 3450) ^ 3 * maxWatts
```

#### Methods

**`setSpeed(rpm: number)`**
- Validates RPM range
- Simulates ramp time
- Updates power consumption

**`getStatus(): PumpStatus`**
- Current RPM
- Watts consumed
- Flow rate (GPM)
- Run time

---

## Configuration

### Enabling Mock Mode

**config.json:**
```json
{
  "controller": {
    "comms": {
      "mockPort": true,
      "rs485Port": "/dev/mockSerial"
    }
  }
}
```

**Result:**
- No physical RS-485 connection
- Mock board instantiated
- Simulated responses only

### Mock Equipment Configuration

Mock boards respect configuration in `poolConfig.json`:
- Number of circuits
- Pump models
- Heater types
- Body configurations

### Simulated Values

**Temperature:**
- Range: 50-104°F
- Change rate: ±0.5°F per minute
- Heater effect: +2°F per minute

**Salt Level:**
- Range: 2000-4000 ppm
- Consumption: -10 ppm per hour at 100% output
- Warning threshold: <2700 ppm

**Pump RPM:**
- Range: 400-3450 RPM
- Ramp rate: 100 RPM per second
- Flow: ~20 GPM at 1800 RPM

---

## Testing Scenarios

### Basic Circuit Control

```bash
# Start in mock mode
npm start

# Turn circuit on (via API)
curl -X PUT http://localhost:4200/state/circuit/1/setState \
  -d '{"state": true}'

# Verify state change
curl http://localhost:4200/state/circuit/1
# Response: {"id": 1, "isOn": true, "name": "Pool Light"}
```

### Temperature Simulation

```bash
# Set heat mode
curl -X PUT http://localhost:4200/state/body/1/heatMode \
  -d '{"heatMode": 1}'  # Heater

# Set setpoint
curl -X PUT http://localhost:4200/state/body/1/setPoint \
  -d '{"setPoint": 85}'

# Watch temperature rise (mock simulates gradual increase)
watch -n 1 'curl -s http://localhost:4200/state/body/1 | jq .temp'
```

### Pump Speed Control

```bash
# Set pump to 2400 RPM
curl -X PUT http://localhost:4200/state/pump/1/rpm \
  -d '{"rpm": 2400}'

# Monitor power consumption
curl http://localhost:4200/state/pump/1
# Response includes calculated watts based on RPM
```

### Schedule Testing

```bash
# Create schedule
curl -X POST http://localhost:4200/config/schedule \
  -d '{
    "circuit": 1,
    "startTime": "08:00",
    "endTime": "18:00",
    "days": [1,2,3,4,5]  # Weekdays
  }'

# Mock board executes schedule at configured times
# No actual time delay in fast testing mode
```

---

## Development Workflow

### 1. Protocol Development

When adding new message types:

```typescript
// 1. Add to Messages.ts
class NewConfigMessage extends Inbound {
  // ... decode logic
}

// 2. Add to MockSystemBoard.ts
processNewConfig(msg: NewConfigMessage) {
  // Generate mock response
  const response = new Response();
  // ... populate response
  this.sendMessage(response);
}

// 3. Test with mock
// No hardware needed!
```

### 2. Feature Testing

```typescript
// Test new feature in isolation
describe('Heater Control', () => {
  before(() => {
    // Start with mock board
    config.setSection('controller.comms.mockPort', true);
  });

  it('should activate heater', async () => {
    const body = sys.bodies.getItemById(1);
    await sys.board.setHeatModeAsync(body, 1);
    
    // Mock responds immediately
    const state = state.bodies.getItemById(1);
    expect(state.heatMode.val).to.equal(1);
  });
});
```

### 3. Regression Testing

```bash
# Run full test suite with mocks
npm test

# Tests run fast without hardware waits
# Predictable behavior
# Repeatable results
```

---

## Mock vs Real Hardware

### Advantages of Mock

✅ Fast response times  
✅ No hardware requirements  
✅ Repeatable behavior  
✅ Safe for destructive tests  
✅ Parallel test execution  
✅ CI/CD integration  
✅ No equipment wear  

### Limitations of Mock

❌ May not catch hardware-specific bugs  
❌ Timing might not match real equipment  
❌ Protocol quirks might be missed  
❌ Network latency not simulated  
❌ Hardware failures not represented  

### When to Use Mock

- Feature development
- Unit testing
- Protocol implementation
- Documentation examples
- Continuous integration
- Demonstrations

### When to Use Real Hardware

- Protocol validation
- Integration testing
- Performance testing
- Edge case discovery
- Final verification
- Bug reproduction

---

## Extending Mock System

### Adding New Mock Equipment

1. **Create Mock Class**
```typescript
// anslq25/equipment/MockHeatPump.ts
export class MockHeatPump {
  private _coeff: number = 3.5;  // COP
  
  processSetMode(mode: number) {
    // Simulate mode change
  }
  
  calculateOutput(ambientTemp: number): number {
    // Realistic heat pump behavior
    return this._coeff * 1000;  // watts
  }
}
```

2. **Integrate with Board**
```typescript
// MockSystemBoard.ts
processHeatPumpStatus() {
  const status = this.mockHeatPump.getStatus();
  // Generate status message
}
```

3. **Add Response Handling**
```typescript
// MessagesMock.ts
if (msg.action === 0x89) {  // Heat pump command
  return this.createHeatPumpResponse(msg);
}
```

### Simulating Errors

```typescript
// Simulate low salt condition
class MockChlorinator {
  generateStatus() {
    if (Math.random() < 0.1) {  // 10% chance
      return this.createLowSaltError();
    }
    return this.createNormalStatus();
  }
}
```

### Advanced Timing

```typescript
// Simulate realistic temperature change
class MockBody {
  private _temp: number = 75;
  private _setPoint: number = 82;
  private _heating: boolean = true;
  
  tick() {
    if (this._heating && this._temp < this._setPoint) {
      // 0.5°F per minute
      this._temp += 0.5 / 60;  // Per second
    }
  }
}
```

---

## File Location Summary

| File/Folder | Purpose |
|-------------|---------|
| `MessagesMock.ts` | Mock message generation |
| `boards/MockSystemBoard.ts` | Base mock board |
| `boards/MockBoardFactory.ts` | Mock board factory |
| `boards/MockEasyTouchBoard.ts` | EasyTouch simulation |
| `chemistry/MockChlorinator.ts` | Chlorinator simulation |
| `pumps/MockPump.ts` | Pump simulation |

---

## Troubleshooting

**Issue:** Mock not activating
- Verify `mockPort: true` in config
- Check `rs485Port` points to `/dev/mockSerial` or similar
- Restart application after config change

**Issue:** Mock responses incorrect
- Check message type handling in `MessagesMock.ts`
- Verify mock board implementation
- Enable packet logging to see exchanges

**Issue:** Timing issues in tests
- Use `await` for async operations
- Mock may respond faster than expected
- Add explicit waits if needed

**Issue:** State not updating
- Ensure mock board emits state changes
- Check event listeners are registered
- Verify state persistence is working

---

## Best Practices

1. **Keep Mocks Realistic**
   - Match real equipment behavior
   - Include realistic delays
   - Simulate errors and edge cases

2. **Maintain Parity**
   - Update mocks when adding features
   - Test with both mock and real hardware
   - Document differences

3. **Use for Development**
   - Develop features with mock first
   - Validate with real hardware later
   - Keep mock tests in CI

4. **Document Limitations**
   - Note what mock doesn't simulate
   - Explain when real hardware needed
   - Warn about mock-specific quirks

---

## Future Enhancements

Potential improvements to mock system:

- **Configurable Scenarios:** Load test scenarios from JSON
- **Fault Injection:** Simulate specific hardware failures
- **Network Latency:** Add configurable delays
- **State Persistence:** Save mock state between runs
- **Interactive Mode:** Manual control of mock equipment
- **Replay Mode:** Replay captured packet sequences
- **Visualization:** Web UI for mock equipment state
