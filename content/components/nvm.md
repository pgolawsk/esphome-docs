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
        size: 4kB
      - id: sensor_cache
        type: raw
        size: 8kB
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
  - String with suffix: `16kB`
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
        size: 4kB

  - platform: fram_i2c
    id: fram2
    address: 0x51
    model: MB85RC64
    partitions:
      - id: cache
        type: raw
        size: 8kB
```

## Custom Size Example

For non-standard FRAM devices, specify a custom size. Note that `address` is required when using custom size:

```yaml
nvm:
  - platform: fram_i2c
    id: my_fram
    address: 0x50  # Required for custom size
    size: 16kB  # Custom 16kB FRAM
    partitions:
      - id: pref_store
        type: preferences
        size: 4kB
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
  - String with suffix: `4kB`, `1MB`
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
        size: 4kB

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
        size: 8kB
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
        size: 2kB
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

## Partition Resizing and Data Preservation

When resizing partitions, understanding how offsets work is crucial for preserving existing data.

### Partition Creation Logging

At startup, partition configuration is logged at DEBUG level:

```
[D][nvm:112] Configured partition 'pref_store': type=preferences, offset=0x0000, size=4000 bytes
[D][nvm:112] Configured partition 'sensor_cache': type=raw, offset=0x1000, size=8000 bytes
```

When a partition is initialized for the first time (empty FRAM or after factory reset), an INFO level message is logged:

```
[I][nvm:573] Created partition 'pref_store': type=preferences, size=4000 bytes
[I][nvm:262] Created partition 'keys_data': type=key_value, size=2000 bytes
```

This helps identify when partitions are actually created vs. just configured from existing data.

### Removing Partitions

When a partition is removed from the YAML configuration:

- **Data remains on FRAM**: The FRAM retains all written data; it's not erased
- **No automatic cleanup**: The storage space is not reclaimed or zeroed
- **Offset implications**: If you add a new partition later at the same offset, it may read old data

To explicitly clear a partition before removal, use a lambda in your config:

```yaml
# Before removing, clear the partition once
on_boot:
  - lambda: |-
      id(my_partition)->clear();  // If clear() method exists
```

Or simply accept that the old data will remain until overwritten by a new partition.

### Factory Reset Behavior

When a factory reset is triggered (via the `factory_reset` component or safe mode):

| Partition Type | Cleared by Factory Reset |
|----------------|--------------------------|
| `preferences` | ✅ Yes - pool is zeroed and reinitialized |
| `raw` | ❌ No - data remains unchanged |
| `key_value` | ❌ No - data remains unchanged |

Raw and key_value partitions are independent storage areas that retain their data across factory resets.

This design allows you to:
- Preserve calibration data or sensor caches through a factory reset by storing them in `raw` partitions
- Keep device configuration in `key_value` storage that survives factory resets
- Use `preferences` for user-modifiable settings that should be cleared on factory reset

### Automatic vs Explicit Offsets

By default, partitions are placed automatically starting at offset 0, with each subsequent partition placed immediately
after the previous one:

```yaml
# Auto-calculated offsets
partitions:
  - id: pref_store
    type: preferences
    size: 4kB    # offset: 0x0000 (auto)
  - id: sensor_cache
    type: raw
    size: 8kB    # offset: 0x1000 (auto - starts right after pref_store)
```

### Data Preservation When Resizing

The preferences partition has built-in logic to handle size changes:

- **Increasing size**: Data is always preserved
- **Decreasing size**: Data is preserved if it fits within the new size, otherwise the pool is cleared

However, **other partitions are not automatically migrated** when offsets change:

| Scenario | Preferences Data | Other Partitions |
|----------|------------------|------------------|
| Increase first partition | ✅ Preserved | ❌ Lost (offset shifts) |
| Decrease first partition | ✅ Preserved | ❌ Lost (offset shifts) |
| Any change with explicit offsets | ✅ Preserved | ✅ Preserved |

### Best Practice: Use Explicit Offsets

To safely resize partitions while preserving data in other partitions, use explicit offsets:

```yaml
partitions:
  - id: pref_store
    type: preferences
    size: 4kB
    # offset: 0 (implicit - first partition always at 0)
  
  - id: sensor_cache
    type: raw
    size: 8kB
    offset: 0x2000   # Explicit offset - won't shift if pref_store changes
```

With explicit offsets, you can safely change `pref_store` size without affecting `sensor_cache`:

```yaml
# After resizing pref_store from 4kB to 6kB
partitions:
  - id: pref_store
    type: preferences
    size: 6kB        # Increased
  
  - id: sensor_cache
    type: raw
    size: 8kB
    offset: 0x0000   # Changed offset - data preserved
```

### Resizing Recommendations

1. **Plan ahead**: Leave gaps between partitions for future growth
2. **Use explicit offsets**: For all partitions after the first one
3. **Place preferences first**: Since it's at offset 0, it can grow without affecting others (if they have explicit offsets)
4. **Monitor usage**: The preferences partition logs pool usage at startup and warns when approaching capacity:

```
[D][nvm:470]: Pool usage: 149/4000 bytes (3.7%)
[W][nvm:474]: Pool is 95% full! Consider increasing partition size
```

#### Warning Thresholds

The preferences and key_value partitions monitor usage and generate warnings at these thresholds:

| Threshold | Default | Description |
|-----------|---------|-------------|
| L1        | 80%     | Warning logged - pool approaching capacity |
| L2        | 90%     | Warning logged - pool nearly full |

| Usage | Timing | Message | Repeat |
|-------|--------|---------|--------|
| > 80% | Startup | "Pool/Partition is X% full. Consider increasing partition size soon" | Once per boot |
| > 80% | Runtime | "Pool/Partition is X% full (Y/Z bytes). Consider increasing partition size" | Once per boot |
| > 90% | Startup | "Pool/Partition is X% full! Consider increasing partition size" | Once per boot |

The runtime warning is triggered when data is written (preferences saved or key_value set) and usage exceeds 80%.
It only fires once per boot session (tracked by internal flag) to avoid log spam.

> **Note**: Usage monitoring is available for `preferences` and `key_value` partitions. `raw` partitions do not
> have automatic usage tracking since they provide direct byte-level access without a storage format.

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
        size: 4kB
      - id: sensor_cache
        type: raw
        size: 8kB

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
