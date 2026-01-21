# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

Fork of geoffdavis/esphome-mitsubishiheatpump maintained for personal use. Original project is in maintenance mode with recommended replacement at [echavet/MitsubishiCN105ESPHome](https://github.com/echavet/MitsubishiCN105ESPHome).

## Build Commands

This is an ESPHome external component. The config uses:
```yaml
external_components:
  - source: github://cohenam/esphome-mitsubishiheatpump
```

**Workflow:** Push changes to GitHub, then compile.

```bash
# Push changes
git add -A && git commit -m "message" && git push

# Clean cached component (forces re-download from GitHub)
rm -rf ~/.esphome/external_components

# Compile
esphome compile ~/src/cohenam/homekit/esphome/config/melcloud-omri.yaml

# Upload to device
esphome upload ~/src/cohenam/homekit/esphome/config/melcloud-omri.yaml

# View logs
esphome logs ~/src/cohenam/homekit/esphome/config/melcloud-omri.yaml
```

**For local testing** (without pushing), temporarily change config to:
```yaml
external_components:
  - source:
      type: local
      path: /Users/yogi/src/cohenam/esphome-mitsubishiheatpump/components
```

## Architecture

### Component Structure
```
components/mitsubishi_heatpump/
├── __init__.py              # ESPHome config schema and C++ code generation
├── climate.py               # Climate platform config and UART setup
├── espmhp.h                 # MitsubishiHeatPump class header
├── espmhp.cpp               # Main implementation (801 lines)
└── mitsubishi_ac_select.h   # Vane position select component
```

### Key Classes

**MitsubishiHeatPump** (`espmhp.h:45-187`)
- Inherits: `PollingComponent`, `climate::Climate`
- Core methods:
  - `setup()` - Initializes HeatPump library, verifies serial connection
  - `update()` - Polls heatpump (called every `update_interval`)
  - `control()` - Handles climate requests (mode, temp, fan, swing)
  - `traits()` / `config_traits()` - Defines supported features
  - `hpSettingsChanged()` / `hpStatusChanged()` - Callbacks from HeatPump library
  - `set_remote_temperature()` - External temperature sensor integration

**MitsubishiACSelect** (`mitsubishi_ac_select.h`)
- Horizontal/vertical vane position controls exposed as ESPHome select entities

### External Dependency

SwiCago/HeatPump Arduino library (commit `5d1e146771d2f458907a855bf9d5d4b9bf5ff033`)
- Handles serial protocol with Mitsubishi units via CN105 connector
- Automatically pulled by ESPHome external_components

## Key Technical Constraints

1. **Direct Hardware UART Required**: Cannot use ESPHome's `uart` component because Mitsubishi units require even parity, which is only available via Arduino's `HardwareSerial`

2. **ESP8266 Limitations**:
   - Only UART0 available
   - Must disable serial logging (`logger: baud_rate: 0`)

3. **ESP32 Options**:
   - Can use UART1 or UART2 to keep logging on UART0
   - Custom `rx_pin`/`tx_pin` supported

4. **Update Interval**: Max 9000ms due to HeatPump library limitations (default 500ms)

5. **Baud Rate**: Default 4800, some units need 2400 or 9600

## Mode Mappings

Climate modes → HeatPump: `HEAT_COOL`→AUTO, `COOL`→COOL, `HEAT`→HEAT, `DRY`→DRY, `FAN_ONLY`→FAN

Fan modes → HeatPump: `DIFFUSE`→QUIET, `LOW`→1, `MEDIUM`→2, `MIDDLE`→3, `HIGH`→4, `AUTO`→AUTO

## Temperature Persistence

Mode-specific setpoints stored via ESPHome preferences (`cool_storage`, `heat_storage`, `auto_storage`) - encoded as steps from MIN_TEMPERATURE (16°C) using TEMPERATURE_STEP (0.5°C).
