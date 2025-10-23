# Configuration Notes - Jesus's Corne Keyboard

## Current Configuration

### Hardware
- **Controller**: nice!nano v2 (both sides)
- **Display**: nice!view with adapter
- **Keyboard**: Corne split (42 keys)

### Side Roles
- **Left**: CENTRAL/MASTER - Connects to computer
- **Right**: PERIPHERAL/SLAVE - Connects to left side

### Bluetooth Configuration

**Left Side (Master):**
- TX Power: `CONFIG_BT_CTLR_TX_PWR_0=y` (0 dBm)
- Reason: Battery saving, computer is usually close
- Estimated duration: 5-6 weeks

**Right Side (Slave):**
- TX Power: `CONFIG_BT_CTLR_TX_PWR_PLUS_4=y` (+4 dBm)
- Reason: Needs good transmission to left side
- Estimated duration: 7-10 weeks

**Important note:** The right side (slave) lasts LONGER on battery because it doesn't maintain the full BLE stack connection to the computer.

## File Structure

```
zmk-config-corneview/
├── config/
│   ├── corne_left.conf     # Left side specific config
│   ├── corne_right.conf    # Right side specific config
│   ├── corne.keymap        # Key layout (shared)
│   └── west.yml
├── build.yaml              # Defines what to compile in GitHub Actions
├── .github/workflows/build.yml
└── zephyr/module.yml
```

## Kconfig Settings

### Why NOT use config/corne.conf?

**Problem:** If `config/corne.conf` exists, ZMK IGNORES the side-specific files `corne_left.conf` and `corne_right.conf`.

**Solution:** Use ONLY side-specific files:
- `config/corne_left.conf`
- `config/corne_right.conf`

### Common Settings on Both Sides

```conf
# Deep sleep (10 minutes of inactivity)
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=600000

# Keyboard name
CONFIG_ZMK_KEYBOARD_NAME="Jesus's Corne"
```

### Difference between CONFIG_BT_CTLR_TX_PWR_* and CONFIG_BT_CTLR_TX_PWR_DBM

- **`CONFIG_BT_CTLR_TX_PWR_0=y`**: Boolean option that YOU configure
- **`CONFIG_BT_CTLR_TX_PWR_DBM=0`**: Numeric value that the SYSTEM calculates automatically

You should only configure the boolean options (those ending in `=y`).

## Available Bluetooth Power Levels

```conf
CONFIG_BT_CTLR_TX_PWR_MINUS_40=y  # -40 dBm (ultra low)
CONFIG_BT_CTLR_TX_PWR_MINUS_20=y  # -20 dBm
CONFIG_BT_CTLR_TX_PWR_MINUS_4=y   # -4 dBm
CONFIG_BT_CTLR_TX_PWR_0=y         # 0 dBm (default)
CONFIG_BT_CTLR_TX_PWR_PLUS_3=y    # +3 dBm
CONFIG_BT_CTLR_TX_PWR_PLUS_4=y    # +4 dBm
CONFIG_BT_CTLR_TX_PWR_PLUS_6=y    # +6 dBm
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y    # +8 dBm (maximum)
```

### Tested Configurations

| Config | Left | Right | Result | Battery Left | Battery Right |
|--------|------|-------|--------|--------------|---------------|
| **Original** | +8 dBm | +8 dBm | ✅ No delay | 3-4 wk | 5-7 wk |
| **First attempt** | 0 dBm | +4 dBm | ❌ Delay on right | - | - |
| **Current (works)** | 0 dBm | +4 dBm | ✅ No delay | 5-6 wk | 7-10 wk |

**Note:** The first attempt failed because `config/corne.conf` was overriding the side-specific configs.

## Bluetooth Profiles (Multi-device)

### Capacity
- ZMK supports up to **5 simultaneous profiles** (0-4)
- Only **1 active profile** at a time
- Quick switching between saved devices

### Control Keys (on layer 2 - RAISE)

```
BT_SEL 0: Select profile 0
BT_SEL 1: Select profile 1
BT_SEL 2: Select profile 2
BT_SEL 3: Select profile 3
BT_SEL 4: Select profile 4
BT_CLR:   Clear current profile
```

### Profile Security

**✅ Inactive profiles are NOT in pairing mode**
- Only the active profile can advertise
- If you're connected, you're not visible to other devices
- Other profiles are completely dormant

**Pairing mode only activates when:**
1. You switch to an empty profile (no saved device)
2. You clear a profile with BT_CLR
3. The active profile cannot reconnect to its device

### Recommended Profile Setup

```
Profile 0: Main computer
Profile 1: Samsung TV / Secondary device
Profile 2: Tablet/Laptop
Profile 3: Phone
Profile 4: [Empty for testing]
```

## Compilation with GitHub Actions

### Trigger
- Automatic on every `git push`
- Manual from GitHub Actions UI

### Process
1. Reads `build.yaml` to see what to compile
2. For each entry (left/right):
   - Reads `config/corne.conf` IF it exists (❌ bad)
   - Reads `config/corne_left.conf` or `corne_right.conf` IF corne.conf doesn't exist (✅ good)
   - Compiles the firmware
3. Generates `.uf2` files
4. Uploads them as artifacts (available for 90 days)

### Verify Correct Compilation

**In GitHub Actions logs you should see:**

```
# For corne_left:
Using NCS config: .../config/corne_left.conf
CONFIG_BT_CTLR_TX_PWR_0=y
CONFIG_BT_CTLR_TX_PWR_DBM=0

# For corne_right:
Using NCS config: .../config/corne_right.conf
CONFIG_BT_CTLR_TX_PWR_PLUS_4=y
CONFIG_BT_CTLR_TX_PWR_DBM=4
```

**If both show the same value → there's a problem** (probably `config/corne.conf` exists)

### Verify Compiled Binaries

```bash
# View file information
file firmware.uf2

# Extract text strings
strings firmware.uf2 | grep "Jesus"

# Calculate checksum (left and right must be different)
md5 corne_left.uf2
md5 corne_right.uf2
```

## Flashing Firmware

1. Connect nice!nano via USB
2. Press RESET twice quickly
3. Appears as USB drive (NICENANO)
4. Copy corresponding `.uf2` file
5. Reboots automatically

**Important:**
- Flash `corne_left...uf2` to LEFT side
- Flash `corne_right...uf2` to RIGHT side

## Battery Consumption

### Consumption Components

**Left Side (Master):**
- BLE Central stack: 5-8 mA (continuous)
- TX power overhead: varies by config
- **Estimated total:** 6-9 mA with 0 dBm

**Right Side (Slave):**
- BLE Peripheral simple: 2-4 mA (continuous)
- TX power overhead: varies by config
- **Estimated total:** 3-5 mA with +4 dBm

**That's why the slave ALWAYS lasts longer than the master**

### Estimated Duration (110 mAh battery)

| TX Power | TX Consumption | Master | Slave |
|----------|----------------|--------|-------|
| 0 dBm | +5.5 mA | 5-6 wk | 7-10 wk |
| +4 dBm | +7.5 mA | 4-5 wk | 6-8 wk |
| +6 dBm | +9 mA | 3-4 wk | 5-7 wk |
| +8 dBm | +10.5 mA | 3-4 wk | 5-7 wk |

## Common Problems and Solutions

### Key Delay

**Cause:** TX power too low on the slave
**Solution:** Increase power on the right side (slave)

### Both Sides with Same Config

**Cause:** `config/corne.conf` exists and overrides
**Solution:** Delete `config/corne.conf`, use only `corne_left.conf` and `corne_right.conf`

### Won't Connect to Samsung TV (or other Smart TVs)

**Possible causes:**
- TV doesn't support BLE HID (only Bluetooth Classic)
- Device whitelist restriction
- Using wrong menu (must be Input Device Manager, not Sound Output)

**Solution:** Try in Settings → Input Device Manager → Bluetooth Keyboard

### Won't Compile in GitHub Actions

**Common errors:**
- TX_PWR conflict (two different values)
- Incorrect syntax in `.conf`
- `.conf` file incorrectly named

## Alternative Configurations to Try

### Maximum Battery
```
Left:  CONFIG_BT_CTLR_TX_PWR_0=y           # 0 dBm
Right: CONFIG_BT_CTLR_TX_PWR_PLUS_3=y      # +3 dBm
```

### Optimal Balance
```
Left:  CONFIG_BT_CTLR_TX_PWR_PLUS_4=y      # +4 dBm
Right: CONFIG_BT_CTLR_TX_PWR_PLUS_6=y      # +6 dBm
```

### Maximum Stability (original)
```
Left:  CONFIG_BT_CTLR_TX_PWR_PLUS_8=y      # +8 dBm
Right: CONFIG_BT_CTLR_TX_PWR_PLUS_8=y      # +8 dBm
```

## Useful References

- [ZMK Documentation](https://zmk.dev)
- [ZMK Discord](https://zmk.dev/community/discord/invite)
- [nice!nano Pinout](https://nicekeyboards.com/docs/nice-nano/pinout-schematic)
- [ZMK Repository](https://github.com/zmkfirmware/zmk)

## Change History

- **2025-10-22**: Initial configuration with separate profiles (0 dBm / +4 dBm)
- **2025-10-22**: Fix: Removed `config/corne.conf` that was overriding specific configs
- **2025-10-22**: Confirmed working without delay

---

**Last updated:** October 22, 2025
**Active configuration:** 0 dBm (left) / +4 dBm (right)
**Status:** ✅ Working correctly
