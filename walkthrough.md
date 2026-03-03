# Enabling and Optimizing TFT SD Printing for RepRap Firmware (RRF)

## Context
The user wanted to enable printing from the TFT's SD card for RepRap Firmware (RRF). Previously, this feature was disabled, and a hack was used to force it. However, the print performance was poor (stuttering).

## Investigation
1.  **Issue Analysis**: We suspected that the heavy `M408` (JSON status response) queries used by default for RRF were overwhelming the serial connection or the TFT's processing power during the high-speed G-code streaming required for printing from the TFT SD.
2.  **OctoPrint Comparison**: To confirm best practices, we analyzed the source code of OctoPrint (specifically `src/octoprint/plugins/serial_connector/serial_comm.py`). We found that OctoPrint **does not use `M408`** at all. instead, it relies on the standard `M105` command for temperature polling and `M27` for SD status, proving that `M408` is not strictly necessary for successful printing on RRF.

## Solution
We decided to align the TFT's behavior with OctoPrint's proven approach for RRF printing:
-   **Switch to M105**: When printing from the TFT SD card (even on RRF), the firmware now uses `loopCheckHeater()` (which sends `M105`) instead of `rrfStatusQuery()` (which sends `M408`). This significantly reduces the overhead on the serial line.
-   **Formalize Feature**: We cleaned up the `Print.c` file by removing the commented-out code that previously blocked this feature, officially enabling the "Print from TFT SD" menu for RRF.

## Changes Made

### 1. Optimize Polling Loop
**File**: `TFT/src/User/API/Mainboard_FlowControl.c`

We modified the `loopBackEnd` function to use `loopCheckHeater()` when printing from the TFT, regardless of the firmware type.

```c
// Before
if (infoMachineSettings.firmwareType != FW_REPRAPFW)
  loopCheckHeater();  // temperature monitor
else
  rrfStatusQuery();  // query RRF status

// After
if (infoMachineSettings.firmwareType != FW_REPRAPFW || isPrintingFromTFT())
  loopCheckHeater();  // temperature monitor
else
  rrfStatusQuery();  // query RRF status
```

### 2. Enable Menu Option
**File**: `TFT/src/User/Menu/Print.c`

We removed the code block that was previously commented out by the user to force-enable the menu.

```c
// Removed the following block:
// if (infoMachineSettings.firmwareType == FW_REPRAPFW)
// {
//   list_mode = infoSettings.files_list_mode;
//   infoFile.source = FS_ONBOARD_MEDIA;
//   ...
// }
```

## Next Steps
-   Compile the firmware.
-   Flash the firmware to the TFT.
-   Test printing a file from the TFT SD card on an RRF machine to verify that the stuttering issue is resolved.
