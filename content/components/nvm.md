---
title: NVM (Non-Volatile Memory)
description: Store data in external Non-Volatile Memory like FRAM or EEPROM
---

The `nvm` component provides a unified interface for Non-Volatile Memory devices like FRAM (Ferroelectric RAM) and EEPROM.
It supports partitioning the memory into logical sections for different purposes.

```yaml
# Example configuration
nvm:
  - platform: fram_i2c
    id: my_fram
    address: 0x50
    model: MB85RC256
    partitions:
      - id: pref_store
        type: preferences
        size: 4KB
      - id: sensor_cache
        type: raw
        size: 8KB
```

## Benefits of External NVM

Using external NVM (like FRAM) instead of internal flash for preferences offers several advantages:

- **Unlimited write cycles** - FRAM supports 10^14 write cycles vs ~10,000 for flash
- **Fast write speed** - FRAM writes at bus speed (no erase cycles needed)
- **Reduced flash wear** - Preserves internal flash for firmware updates
- **Larger storage** - External NVM can provide more storage than internal flash

## Configuration

This component requires an I²C bus. See [I²C](/components/i2c) for configuration.

## Configuration Variables

- **platform** (**Required**, string): The NVM platform to use. Currently only `fram_i2c`.
- **id** (**Required**, ID): Manually specify the ID for this NVM device.
- **address** (*Optional*, int): I²C address of the FRAM device. Defaults to `0x50` when using a known `model`. **Required** when using custom `size` (different FRAM devices have different default addresses). All supported models use address range `0x50`-`0x57` (configurable via A0-A2 pins).
- **model** (*Optional*, string): The FRAM model. One of:
  
  **Fujitsu MB85RC FRAM Series:**
  - `MB85RC64` - 64 Kbit (8 KB)
  - `MB85RC128` - 128 Kbit (16 KB)
  - `MB85RC256` - 256 Kbit (32 KB)
  - `MB85RC512` - 512 Kbit (64 KB)
  - `MB85RC1M` - 1 Mbit (128 KB)
  - `MB85RC2M` - 2 Mbit (256 KB)
  - `MB85RC4M` - 4 Mbit (512 KB)
  
  **Infineon/Cypress FRAM Series:**
  - `FM24CL64B` - 64 Kbit (8 KB)
  - `FM24CL256B` - 256 Kbit (32 KB)
  - `CY15B104QSN` - 4 Mbit (512 KB)
  - `CY15B108QSN` - 8 Mbit (1 MB)

- **size** (*Optional*, int or string): Custom FRAM size in bytes. Use this for non-standard FRAM devices. Either `model` or `size` must be specified. When using custom size, `address` is required. Can be specified as:
  - Integer bytes: `16384`
  - String with suffix: `16KB`
- **partitions** (*Optional*, list): List of partitions to create. See [Partition Configuration](#partition-configuration).

## Multiple NVM Devices

This component supports multiple NVM devices (MULTI_CONF). You can configure multiple FRAM chips on different I²C addresses or buses:

```yaml
# Multiple FRAM devices on same I²C bus
nvm:
  - platform: fram_i2c
    id: fram1
    address: 0x50
    model: MB85RC256
    partitions:
      - id: pref_store
        type: preferences
        size: 4KB

  - platform: fram_i2c
    id: fram2
    address: 0x51
    model: MB85RC64
    partitions:
      - id: cache
        type: raw
        size: 8KB
```

## Custom Size Example

For non-standard FRAM devices, specify a custom size. Note that `address` is required when using custom size:

```yaml
nvm:
  - platform: fram_i2c
    id: my_fram
    address: 0x50  # Required for custom size
    size: 16KB  # Custom 16KB FRAM
    partitions:
      - id: pref_store
        type: preferences
        size: 4KB
```

## Partition Configuration

Partitions divide the NVM device into logical sections. Each partition has:

- **id** (**Required**, ID): Unique identifier for this partition.
- **type** (**Required**, string): Type of partition. One of:
  - `preferences` - ESPHome preferences backend
  - `raw` - Raw byte storage
  - `key_value` - Key-value store
- **size** (**Required**, int or string): Size of the partition. Can be specified as:
  - Integer bytes: `4096`
  - String with suffix: `4KB`, `1MB`
- **offset** (*Optional*, int): Offset within NVM device. Auto-calculated if not specified.

### Preferences Partition

The `preferences` partition type integrates with ESPHome's preferences system, allowing global variables and other
preferences to be stored in external NVM instead of internal flash.

```yaml
nvm:
  - platform: fram_i2c
    id: my_fram
    model: MB85RC256
    partitions:
      - id: pref_store
        type: preferences
        size: 4KB

# Global variables stored in FRAM
globals:
  - id: total_runtime
    type: uint32_t
    restore_value: true
```

When a preferences partition is configured, all preferences (like `globals.restore_value: true`) will be stored in the
FRAM instead of internal flash.

### Raw Partition

The `raw` partition type provides direct byte-level access to the storage. You can read and write arbitrary data at any
offset within lambdas.

```yaml
nvm:
  - platform: fram_i2c
    id: my_fram
    model: MB85RC256
    partitions:
      - id: sensor_cache
        type: raw
        size: 8KB
```

Access in lambdas:

```cpp
// Write data
uint8_t data[3] = {0x01, 0x02, 0x03};
id(sensor_cache)->write(0, data, 3);

// Read data
uint8_t buffer[3];
id(sensor_cache)->read(0, buffer, 3);
```

### Key-Value Partition

The `key_value` partition type provides a simple key-value store where values can be stored and retrieved by string
keys.

```yaml
nvm:
  - platform: fram_i2c
    id: my_fram
    model: MB85RC256
    partitions:
      - id: config
        type: key_value
        size: 2KB
```

Access in lambdas:

```cpp
// Store values
id(config)->set_string("device_name", "Living Room Sensor");
uint32_t counter = 42;
id(config)->set("counter", reinterpret_cast<uint8_t*>(&counter), sizeof(counter));

// Retrieve values
std::string name = id(config)->get_string("device_name", "Default Name");
uint32_t counter = 0;
id(config)->get("counter", reinterpret_cast<uint8_t*>(&counter), sizeof(counter));
```

## Complete Example

```yaml
i2c:
  sda: GPIOXX
  scl: GPIOXX

nvm:
  - platform: fram_i2c
    id: my_fram
    address: 0x50
    model: MB85RC256
    partitions:
      - id: pref_store
        type: preferences
        size: 4KB
      - id: sensor_cache
        type: raw
        size: 8KB

globals:
  - id: total_runtime
    type: uint32_t
    restore_value: true

sensor:
  - platform: template
    name: "Total Runtime"
    lambda: return id(total_runtime);
    update_interval: 60s
    on_value:
      - lambda: id(total_runtime) += 60;
```

## See Also

- [I²C Component](/components/i2c)
- [Globals Component](/components/globals)
- [Safe Mode Component](/components/safe_mode)
