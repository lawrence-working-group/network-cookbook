+++
title = "Flashing Dell PowerEdge T30 to Precision T3620"
date = "2021-05-27"
tags = ["Dell", "Firmware"]
+++

Every PowerEdge T30 user hates Dell for taking away functionality with firmware updates. This article provides instructions on flashing PowerEdge T30 to Precision T3620 to get full functionality.

# Tools & Materials

- [Dell Precision T3620 BIOS region dump (ver. 1.2.0)](https://mega.nz/file/jt4WDCDb#FWd-hxCOk-Xrkpgn2oQg5yGc2CtIIVkKiY6JuzTLyJI)
- [Intel CSME System Tools v11](https://www.win-raid.com/t596f39-Intel-Converged-Security-Management-Engine-Drivers-Firmware-and-Tools.html)
- [Intel CSME 11.0 Repository](https://www.win-raid.com/t832f39-Intel-CS-ME-CS-TXE-CS-SPS-GSC-PMC-PCHC-PHY-amp-OROM-Firmware-Repositories.html)

# Firmware Dump

In normal mode, you cannot dump the entire firmware. There is a 2-pin jumper plug plugged into the PSWD jumper. Shutdown your T30, remove the jumper plug and plug it into SERVICE_MODE. Boot into Windows, you should now be able to dump the firmware using Flash Programming Tool included in CSME System Tools.

```
.\FPTW64.exe -D DUMP.bin
```

# Mod

## Generate Config File

Open DUMP.bin with Intel Flash Image Tool. Go to `Build -> Build Settings`, set `Generate Intermediate Files` to No. Save the config file to `CONFIG.xml`. Close Flash Image Tool.

## Replace BIOS and ME region

Rename the downloaded BIOS region dump file to `BIOS Region.bin` and replace the file with the same name under `DUMP/Decomp` inside the Flash Image Tool directory.

Go to downloaded CSME 11.0 Repository, find `11.0.0.1183_COR_H_D_PRD_RGN.bin` and rename to `ME Region.bin`. Replace the file with the same name under `DUMP/Decomp` inside the Flash Image Tool directory. Ignore the size difference. It will work.

## Build the Image

Launch Flash Image Tool, open the saved `CONFIG.xml`. Then build the image. An `outimage.bin` will be generated inside the Flash Image Tool directory.

# Flash the Image

Copy `outimage.bin` to the Flash Programming Tool folder. Now you can flash the generated image.

```
.\FPTW64.exe -f .\outimage.bin
```

When completed, shutdown your T30.

# Reset Firmware Settings

Use the jumper plug to short `CMCLR` jumper. After that, plug it back to `PSWD`.

# Initialize ME

Power on your T30. Your T30 may fail to boot / shutdown / reboot multiple times. Retry until you boot into your OS. 

Use Flash Programming Tool to force re-initialize ME.

```
.\FPTW64.exe -greset
```

Your T30 should automatically reboot after that.

Note that if you have a "No bootable device found!" problem, you may checkout your firmware settings. It has something to do with AHCI / RAID or UEFI / Legacy boot.

# Update Firmware

The provided BIOS region dump is version 1.2.0. Go to the Dell website to download and execute the latest Precision T3620 firmware update. It is highly recommended to reset the BIOS settings after the update.
