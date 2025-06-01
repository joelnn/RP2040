# PicoBOB G540 Build Configuration for grblHAL

This document describes the build configuration for the PicoBOB G540 board, a Raspberry Pi Pico-based CNC controller designed to work with the Gecko G540 stepper driver.

## Table of Contents
- [Overview](#overview)
- [Hardware Configuration](#hardware-configuration)
- [Build Methods](#build-methods)
- [Configuration Details](#configuration-details)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Overview

The PicoBOB G540 is a specialized board configuration that interfaces the Raspberry Pi Pico with the Gecko G540 4-axis stepper driver. This configuration provides:

- USB serial communication
- PWM spindle control with enable signal
- Shared limit switches for X, Y, and Z axes
- Probe input support
- E-stop and feed hold controls
- Coolant control output

Board Information:
- **Board Name**: PicoBOB_G540
- **Project URL**: https://github.com/Expatria-Technologies/PicoBOB
- **Compatible With**: Gecko G540 4-axis stepper driver

## Hardware Configuration

### Pin Mappings

The PicoBOB G540 uses the following pin configuration (defined in `boards/picobob_g540_map.h`):

#### Step/Direction Signals
- **Step Pins**: GPIO 17-20 (via PIO state machine)
  - X Step: GPIO 17
  - Y Step: GPIO 18
  - Z Step: GPIO 19
  - A Step: GPIO 20 (if 4th axis enabled)
- **Direction Pins**:
  - X Direction: GPIO 9
  - Y Direction: GPIO 10
  - Z Direction: GPIO 11
  - A Direction: GPIO 12 (if 4th axis enabled)

#### Limit Switches
- **Shared Limit Pin**: GPIO 3 (X, Y, and Z axes share this pin)
- **A Limit**: GPIO 2 (if 4th axis enabled)

#### Spindle Control
- **PWM Output**: GPIO 16
- **Spindle Enable**: GPIO 14
- **Note**: No direction signal (not supported by Mach3 BOB)

#### Control Inputs
- **Probe**: GPIO 4
- **Reset/E-Stop**: GPIO 5
- **Feed Hold**: GPIO 1

#### Outputs
- **Coolant Flood**: GPIO 13

### Important Limitations
- Trinamic drivers are not supported with this configuration
- No spindle direction control (hardware limitation)
- Stepper enable signal is replaced with coolant control

## Build Methods

### Method 1: Docker Build (Recommended)

The Docker build method ensures compatibility regardless of your host system's GLIBC version.

```bash
# Make the script executable
chmod +x docker_build_picobob_g540.sh

# Run the Docker build
./docker_build_picobob_g540.sh
```

This script will:
1. Build a Docker image with Ubuntu 22.04 and all required tools
2. Initialize git submodules
3. Configure the build for PicoBOB G540
4. Build the firmware
5. Output `grblHAL.uf2` in the `build/` directory

### Method 2: Native Build

If your system has the required dependencies:

```bash
# Install dependencies (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y cmake gcc-arm-none-eabi \
    libnewlib-arm-none-eabi libstdc++-arm-none-eabi-newlib

# Make the script executable
chmod +x build_picobob_g540.sh

# Run the build
./build_picobob_g540.sh
```

**Note**: This method requires GLIBC 2.32+ for the Pico SDK tools to work properly.

### Method 3: GitHub Actions CI

The repository includes a CI workflow that automatically builds the firmware:

1. Push to the `picobob-g540-build` branch
2. The workflow will trigger automatically
3. Download artifacts from the Actions tab when complete

## Configuration Details

### my_machine.h Configuration

The build scripts automatically generate the following configuration:

```c
// Enable PicoBOB G540 board
#define BOARD_PICOBOB_G540

// Configuration
#define USB_SERIAL_CDC          1 // Serial communication via native USB

// Control signals
#define CONTROL_ENABLE          (CONTROL_HALT|CONTROL_FEED_HOLD)

// Spindle configuration
#define DRIVER_SPINDLE_ENABLE   (SPINDLE_PWM|SPINDLE_ENA)

// Default axis configuration is 3 axes (X, Y, Z)
// Uncomment to enable 4th axis (A)
//#define N_ABC_MOTORS 1
```

### Optional Features

To enable additional features, modify `my_machine.h` before building:

```c
// Enable SD card support
#define SDCARD_ENABLE           2

// Enable probe input (enabled by default)
#define PROBE_ENABLE            1

// Enable coolant control
#define COOLANT_ENABLE          3
```

## Verification

### 1. Build Output Verification

After a successful build, you should see these files in the `build/` directory:

```
grblHAL.uf2  - Main firmware file (approx. 430-450KB)
grblHAL.bin  - Binary format (approx. 215-230KB)
grblHAL.hex  - Intel HEX format (approx. 600-640KB)
```

### 2. Firmware Content Verification

Verify the correct board configuration is included:

```bash
# Check for board name in binary
strings build/grblHAL.bin | grep -i "picobob"
# Should output: PicoBOB_G540

# Check for board URL
strings build/grblHAL.bin | grep -i "expatria"
# Should output: https://github.com/Expatria-Technologies/PicoBOB
```

### 3. Configuration Verification in Code

The build system verifies:
- `BOARD_PICOBOB_G540` is defined in `my_machine.h`
- `driver.h` includes `boards/picobob_g540_map.h` when this is defined
- The correct pin mappings are applied during compilation

### 4. Runtime Verification

After flashing the firmware:

1. Connect via USB serial (115200 baud)
2. Send `$I` command to check build info
3. Send `$$` to view all settings
4. Send `$pins` to verify pin assignments match the PicoBOB G540 configuration

## Troubleshooting

### GLIBC Version Error

If you see errors about GLIBC versions when building natively:
```
/lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.32' not found
```

**Solution**: Use the Docker build method instead.

### Submodules Not Found

If the build fails with missing CMakeLists.txt errors:
```
include could not find load file: grbl/CMakeLists.txt
```

**Solution**: Initialize git submodules:
```bash
git submodule update --init --recursive
```

### Wrong Board Configuration

If the firmware doesn't behave as expected:

1. Verify `my_machine.h` contains `#define BOARD_PICOBOB_G540`
2. Clean and rebuild:
   ```bash
   rm -rf build
   ./docker_build_picobob_g540.sh
   ```

### Spindle Not Working

The PicoBOB G540 does not support spindle direction control due to hardware limitations. Ensure your spindle only requires:
- PWM speed control (M3 Sxxx)
- Enable/disable (M3/M5)

## Flashing the Firmware

1. Hold the BOOTSEL button on your Raspberry Pi Pico
2. Connect the Pico to your computer via USB
3. Release the BOOTSEL button
4. The Pico will appear as a mass storage device (RPI-RP2)
5. Copy `build/grblHAL.uf2` to the RPI-RP2 drive
6. The Pico will automatically reboot with the new firmware

## Support

For issues specific to:
- **PicoBOB Hardware**: https://github.com/Expatria-Technologies/PicoBOB
- **grblHAL Firmware**: https://github.com/grblHAL/core
- **This Build Configuration**: Create an issue in this repository

## License

This configuration follows the grblHAL license (GPLv3). See the main grblHAL repository for details.