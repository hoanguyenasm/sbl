# MIMXRT1176 Flashing Guide for SBL and Application Firmware

This document provides a comprehensive guide for flashing the Secondary Boot Loader (SBL) and application firmware in single image mode for the MIMXRT1176 platform using MCUBootUtility.

## Memory Map Overview

### Flash Memory Layout
```
┌─────────────────────┬─────────────┬────────────────────────────────────┐
│ Memory Address      │ Size        │ Description                        │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x30000000          │ 0x400       │ Reserved                           │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x30000400          │ 0xC00       │ SBL Flash Configuration            │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x30001000          │ 0x1000      │ SBL Image Vector Table (IVT)       │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x30002000          │ 0x400       │ SBL Interrupt Vector Table         │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x30002400          │ ~0xFDC00    │ SBL Code (.text)                   │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x30100000          │ 0x400       │ Application Header                 │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x30100400          │ Remaining   │ Application Code                   │
└─────────────────────┴─────────────┴────────────────────────────────────┘
```

### RAM Memory Layout (Non-XIP Mode)
```
┌─────────────────────┬─────────────┬────────────────────────────────────┐
│ Memory Address      │ Size        │ Description                        │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x20000000          │ 0x40000     │ SRAM_DTC_cm7 (RAM for application) │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x20240000          │ 0x80000     │ SRAM_OC1                           │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x202C0000          │ Variable    │ RPMSG Shared Memory                │
├─────────────────────┼─────────────┼────────────────────────────────────┤
│ 0x202C0000+RPMSG    │ Remaining   │ SRAM_OC2                           │
└─────────────────────┴─────────────┴────────────────────────────────────┘
```

## SBL Configuration Requirements

Before building the SBL, ensure the following configuration settings in `sblconfig.h`:

```c
/* Required SBL Configuration */
#define SINGLE_IMAGE               // Enable single image mode
#define COMPONENT_MCU_ISP          // Enable MCU ISP support
#define COMPONENT_FLASHIAP         // Enable flash in application programming

/* Disabled Features */
// #define VERIFY_IMAGE            // Disable ROM to verify SBL
// #define MCUBOOT_IMAGE_SECURITY  // Disable image security function
// #define COMPONENT_SDRAM         // Disable SDRAM support
```

Ensure these settings are correctly configured in your SBL build to match your requirements.

## Flashing Process Using MCUBootUtility

### Prerequisites
- NXP MCUBootUtility (latest version)
- MCUXpresso IDE or SDK
- USB or UART connection to the board
- SBL binary
- Application binary (properly formatted with header)

### Step 1: Configure SBL

1. **Single Image Mode Configuration**
   - Ensure the SBL is configured according to the requirements above
   - Verify flash map configuration:
     ```c
     #define BOOT_FLASH_BASE      0x30000000
     #define BOOT_FLASH_HEADER    0x30010000
     #define BOOT_FLASH_ACT_APP   0x30100000
     #define BOOT_FLASH_CAND_APP  0x30200000
     #define BOOT_FLASH_CUSTOMER  0x303f0000
     ```

2. **Build SBL**
   - Compile the SBL project to generate the binary file

### Step 2: Configure Application Firmware

1. **Configure XIP Setting**
   - For non-XIP mode (application runs from RAM):
     - Set `XIP_ENABLE = 0` in the application project
     - Ensure the linker script is configured for RAM execution with flash loading

   - For XIP mode (application runs directly from flash):
     - Set `XIP_ENABLE = 1` in the application project
     - Ensure the linker script is configured for flash execution

2. **Configure Memory Settings**
   - If using MCUXpresso IDE, set memory configuration:
     ```xml
     <memoryInstance id='BOARD_FLASH' derived_from='Flash' 
       location='0x30100400' size='0x4000000' driver='MIMXRT1170_SFDP_QSPI.cfx' />
     ```
   - This configuration places your application code correctly in the flash area after the header

3. **Build Application**
   - Set `XIP_BOOT_HEADER_ENABLE = 0` in the application project
   - This ensures the application firmware relies on the SBL for booting
   - Compile the application project to generate the binary file

### Step 3: Prepare Application Image with Header

1. **Generate Application Header**
   - Use NXP's signing tool (e.g., elftosb, imgtool) to generate the header
   - Example command:
     ```bash
     imgtool sign --key key.pem --align 4 --version 1.0.0 --header-size 0x400 \
       --pad-header --slot-size 0x100000 application.bin signed_application.bin
     ```

2. **Verify Header**
   - The header should be 0x400 bytes and contain information about:
     - Application size
     - Entry point (Reset_Handler)
     - Image validation metadata

### Step 4: Flash Using MCUBootUtility

1. **Launch MCUBootUtility**
   - Open MCUBootUtility application
   - Select the appropriate device (MIMXRT1176)

2. **Configure Serial Port**
   - Select the correct COM port connected to your board
   - Configure baud rate (typically 115200)
   - Click "Connect" to establish connection with the board

3. **Enter Boot Mode**
   - Set the board to serial downloader mode (typically by holding BOOT mode button during reset)
   - MCUBootUtility should detect the device in boot mode

4. **Flash SBL**
   - In the MCUBootUtility interface:
     - Select the "Load File" option
     - Browse and select the SBL binary
     - Set the destination address to `0x30000400`
     - Click "Load to target" to begin flashing

5. **Flash Application**
   - In the MCUBootUtility interface:
     - Select the "Load File" option
     - Browse and select the signed application binary (with header)
     - Set the destination address to `0x30100000`
     - Click "Load to target" to begin flashing

6. **Verify Flashing**
   - MCUBootUtility should display successful programming messages
   - You can optionally read back the memory to verify the flashing was successful

7. **Reset the Board**
   - Click the "Reset" button in MCUBootUtility or manually reset the board
   - The board should now boot into the SBL, which will then load the application

> ⚠️ **Important**: Always flash the complete application image (including header) at `0x30100000`. Do not flash only the application code at `0x30100400` as the SBL requires the header information to locate and load the application.

## Execution Flow

### Non-XIP Mode (XIP_ENABLE = 0)
1. ROM bootloader loads the SBL from flash to RAM and executes it
2. SBL initializes hardware and locates the application image at `0x30100000`
3. SBL reads the application header to determine memory layout and entry point
4. SBL copies the application code from flash (`0x30100400+`) to RAM (`0x20000000+`)
5. SBL jumps to the application entry point in RAM
6. Application executes from RAM

### XIP Mode (XIP_ENABLE = 1)
1. ROM bootloader loads the SBL from flash to RAM and executes it
2. SBL initializes hardware and locates the application image at `0x30100000`
3. SBL reads the application header to determine memory layout and entry point
4. SBL initializes the `.data` section (if needed)
5. SBL jumps to the application entry point in flash
6. Application executes directly from flash

## Verification and Debugging

1. **SBL Boot Messages**
   - When correctly set up, the SBL should output something like:
     ```
     hello sbl.
     Bootloader Version 1.1.0
     Bootloader chainload address offset: 0x100000
     Reset_Handler address offset: 0x100400
     Jumping to the image
     ```

2. **Common Issues and Solutions**
   - **"Unable to find bootable image"**:
     - Verify application image is correctly flashed at `0x30100000`
     - Ensure application header is correctly formatted
     
   - **Application doesn't start after jump**:
     - Check if the Reset_Handler offset matches the one expected by the SBL
     - Verify application is built for the correct memory mode (XIP or non-XIP)

   - **MCUBootUtility connection issues**:
     - Ensure the board is in boot mode before connecting
     - Try different USB ports or cables
     - Reset the board and MCUBootUtility connection

## References

- [NXP MIMXRT1170 Reference Manual](https://www.nxp.com/docs/en/reference-manual/IMXRT1170RM.pdf)
- [MCUBoot Documentation](https://www.mcuboot.com/)
- [NXP MCUXpresso SDK](https://www.nxp.com/design/software/development-software/mcuxpresso-software-and-tools/mcuxpresso-software-development-kit-sdk:MCUXpresso-SDK)
- [MCUBootUtility User Guide](https://www.nxp.com/docs/en/user-guide/MCUBOOTUTIUG.pdf)

---

This document provides a comprehensive guide for flashing and configuring the SBL and application firmware on the MIMXRT1176 platform in single image mode using MCUBootUtility. Always refer to the latest NXP documentation for any platform-specific updates or changes.
