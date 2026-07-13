# MeshCore — LilyGo T-Beam T22 V1.1 Fork

## Fork
- Upstream: https://github.com/meshcore-dev/MeshCore
- Fork: https://github.com/william67/MeshCore
- Working branch: `tbeam-sx1276-fixes`

## Board
- LilyGo T-Beam T22 V1.1, SX1276, AXP192, SH1106 OLED
- MAC suffix: DFD8A8
- PlatformIO environment: `Tbeam_SX1276_repeater`

## Build & Flash

```powershell
# One-time setup per PowerShell session:
. "C:\Espressif\v5.3.2\Microsoft.PowerShell_profile.ps1"

cd "C:\Apps\MeshCore"
pio run -e Tbeam_SX1276_repeater
pio run -e Tbeam_SX1276_repeater -t upload --upload-port COM16
```

## Fixes Applied (vs upstream)

See `tbeam-sx1276-fixes.md` for full details.

1. **`src/helpers/esp32/TBeamBoard.cpp`** — AXP2101 init wrapped in `#ifdef TBEAM_SUPREME_SX1262`; prevents AXP2101 driver falsely claiming the AXP192 chip and leaving LoRa radio unpowered (error -2)
2. **`variants/lilygo_tbeam_SX1276/platformio.ini`** — switched from `SSD1306Display` to `SH1106Display` (correct OLED driver for this board)
3. **`variants/lilygo_tbeam_SX1276/target.h`** — include changed to match display driver

## After First Flash

```
set radio 869.6179809,62.5,7,5
reboot
```

Default firmware builds with SF8; local MeshCore network uses SF7.

## Deployment

Repeater is deployed at 8m height, window open, facing 7221 (~10km NLOS).
Transport keys pushed via MeshCore companion app (admin login) to enable FLOOD+TRANSPORT relay.
