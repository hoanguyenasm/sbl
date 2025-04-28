# RT1176 Firmware Update Guide using SBL Remap Functionality

This guide documents the process for implementing firmware updates using the Secondary Boot Loader (SBL) remap functionality on NXP MIMXRT1176 platforms.

## Overview

The SBL remap functionality provides a safe and reliable method for firmware updates by allowing the system to switch between active and candidate firmware images without physically copying the firmware. This approach offers:

- Reduced update time
- Lower risk of update failures
- Fallback capability if new firmware is corrupt

## Memory Map

```
┌─────────────────────┬─────────────┬────────────────────────────────────┐
│ Memory Address      │ Size        │ Description                        │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x30000400          │ Variable    │ SBL (Secondary Boot Loader)        │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x30100000          │ 0x100000    │ Active Firmware Slot               │
│                     │             │ (BOOT_FLASH_ACT_APP)               │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x30200000          │ 0x100000    │ Candidate Firmware Slot            │
│                     │             │ (BOOT_FLASH_CAND_APP)              │
└─────────────────────┴─────────────┴────────────────────────────────────┘
```

## Prerequisites

1. SBL compiled with the following configuration:
   - `SOC_REMAP_ENABLE` defined
   - `COMPONENT_MCU_ISP` defined for In-System Programming support
   - `COMPONENT_FLASHIAP` defined for flash programming support

2. Development tools:
   - MCUXpresso IDE
   - MCUBootUtility for flashing
   - J-Link or compatible debugger

## Step 1: Initial Setup

### 1.1 Configure and Build SBL

Ensure the following in `sblconfig.h`:

```c
/* Required configuration for remap-based updates */
#define SOC_REMAP_ENABLE
#define COMPONENT_MCU_ISP
#define COMPONENT_FLASHIAP

/* Flash Memory Map */
#define BOOT_FLASH_BASE 0x30000000
#define BOOT_FLASH_HEADER 0x30010000
#define BOOT_FLASH_ACT_APP 0x30100000
#define BOOT_FLASH_CAND_APP 0x30200000
```

Build the SBL with these configurations.

### 1.2 Configure Application Firmware

Configure the application firmware:
- Set `XIP_ENABLE = 1` for Execute-In-Place mode
- Set `XIP_BOOT_HEADER_ENABLE = 0` (no ROM boot header, SBL handles this)
- Ensure the linker script is configured for flash execution

Build the initial application firmware.

### 1.3 Initial Deployment

1. Flash the SBL at address `0x30000400` using MCUBootUtility.
2. Sign the application and flash it at `0x30100000`.

## Step 2: Firmware Update Process

### 2.1 Prepare the New Firmware

1. Build the new firmware version.
2. Sign the new firmware binary to add the header:
   ```bash
   imgtool sign --key key.pem --align 4 --version 1.0.0 --header-size 0x400 \
     --pad-header --slot-size 0x100000 new_firmware.bin signed_new_firmware.bin
   ```

### 2.2 Flash the New Firmware to Candidate Slot

Using MCUBootUtility:
1. Connect to the target device.
2. Flash the signed new firmware to the candidate slot at address `0x30200000`.

### 2.3 Trigger Remap and Update

1. Reset the device to boot into the SBL.
2. When prompted with "Remap type:", respond with 'Y' or 'y' to enable remap.
3. The SBL will call `SBL_EnableRemap(BOOT_FLASH_ACT_APP, BOOT_FLASH_ACT_APP+FLASH_AREA_IMAGE_1_SIZE, FLASH_AREA_IMAGE_1_SIZE)` which:
   - Sets up a memory remap redirecting accesses from the active slot to the candidate slot
   - Loads the new firmware without physically moving it

4. The system now boots into the new firmware.

## Step 3: Verifying and Finalizing the Update

### 3.1 Verify New Firmware

1. Test the new firmware functionality to ensure it operates correctly.
2. If testing is successful, you can proceed to finalize the update.
3. If any issues are found, you can reboot and select 'N' at the remap prompt to revert to the original firmware.

### 3.2 Finalize the Update (Optional)

After successful verification, you may want to physically update the active slot:

1. Use the flash programming capabilities in your application to:
   - Erase the active slot (`BOOT_FLASH_ACT_APP`)
   - Copy the firmware from candidate slot (`BOOT_FLASH_CAND_APP`) to active slot
   - Verify the copy operation

2. After physical copy is complete, disable the remap by resetting and selecting 'N' at the remap prompt.

## Troubleshooting

### Issues with Remap

1. **Remap Not Working**:
   - Ensure `SOC_REMAP_ENABLE` is defined in SBL configuration
   - Verify memory addresses in SBL configuration match your memory layout
   - Check that the `SBL_EnableRemap` function is correctly implemented

2. **System Crashes After Remap**:
   - The candidate firmware may be corrupt or incompatible
   - Reset and select 'N' at the remap prompt to boot the original firmware
   - Verify the new firmware is properly signed and formatted

### Flash Programming Issues

1. **Flash Programming Fails**:
   - Ensure `COMPONENT_FLASHIAP` is defined in SBL configuration
   - Verify flash drivers are correctly initialized
   - Check for write protection or security features that may be enabled

2. **Verification Errors**:
   - The flash content may be corrupted during programming
   - Try lowering the flash programming speed
   - Ensure power is stable during the update process

## Advanced Configuration / Future Implementation

### Automated Update Process

You can modify the SBL to automatically perform the update process:

1. Add code to check for a valid firmware in the candidate slot
2. If a valid candidate is found, automatically enable remap
3. After validation period, automatically finalize the update

### Failsafe Updates

Implement a failsafe mechanism:

1. Add a persistent flag to track update attempts
2. If the system fails to boot the new firmware multiple times, automatically revert to the original
3. Implement a watchdog to detect firmware crashes during startup
