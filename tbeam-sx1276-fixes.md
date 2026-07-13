# MeshCore Repeater — LilyGo T-Beam T22 V1.1 (SX1276)

## Hardware

- Board: LilyGo T-Beam T22 V1.1
- Radio: SX1276 (868 MHz)
- PMIC: AXP192
- MAC suffix: DFD8A8
- COM port: COM16

## Repeater ID

```
975B0EB655310BA8D583221E2341D4B6C92859ACFD511510E8A92CAEF1822238
```

## Source

MeshCore open source: https://github.com/meshcore-dev/MeshCore  
Fork: https://github.com/william67/MeshCore (branch: `tbeam-sx1276-fixes`)  
Cloned to: `C:\Apps\MeshCore`

PlatformIO environment: `Tbeam_SX1276_repeater`  
Variant config: `C:\Apps\MeshCore\variants\lilygo_tbeam_SX1276\platformio.ini`

## Bug Fix — AXP192 not powered (radio init failed: -2)

### Root cause

`C:\Apps\MeshCore\src\helpers\esp32\TBeamBoard.cpp` — `power_init()`

On non-Supreme T-Beam boards, the code first tried `XPowersAXP2101::init()`. The problem: both AXP192 and AXP2101 share I2C address `0x34`. The AXP2101 driver incorrectly returns `true` when probing an AXP192 chip. It then writes to the AXP2101 ALDO2 register (LoRa power rail on AXP2101) instead of the AXP192 LDO2 register, leaving the radio unpowered. RadioLib sees no SPI response → error `-2` (`RADIOLIB_ERR_CHIP_NOT_FOUND`).

### Fix applied

Wrapped the non-Supreme AXP2101 init block in `#ifdef TBEAM_SUPREME_SX1262` so it only runs on the T-Beam Supreme (which actually has AXP2101). All other T-Beam boards fall through to the AXP192 init.

**Before** (`TBeamBoard.cpp`, `power_init()`):
```cpp
if (!PMU) {
    #ifdef TBEAM_SUPREME_SX1262
      PMU = new XPowersAXP2101(PMU_WIRE_PORT, PIN_BOARD_SDA1, PIN_BOARD_SCL1, I2C_PMU_ADD);
    #else
      PMU = new XPowersAXP2101(PMU_WIRE_PORT, PIN_BOARD_SDA, PIN_BOARD_SCL, I2C_PMU_ADD);
    #endif
    if (!PMU->init()) {
        MESH_DEBUG_PRINTLN("Warning: Failed to find AXP2101 power management");
        delete PMU;
        PMU = NULL;
    } else {
        MESH_DEBUG_PRINTLN("AXP2101 PMU init succeeded, using AXP2101 PMU");
    }
}
```

**After**:
```cpp
#ifdef TBEAM_SUPREME_SX1262
if (!PMU) {
    PMU = new XPowersAXP2101(PMU_WIRE_PORT, PIN_BOARD_SDA1, PIN_BOARD_SCL1, I2C_PMU_ADD);
    if (!PMU->init()) {
        MESH_DEBUG_PRINTLN("Warning: Failed to find AXP2101 power management");
        delete PMU;
        PMU = NULL;
    } else {
        MESH_DEBUG_PRINTLN("AXP2101 PMU init succeeded, using AXP2101 PMU");
    }
}
#endif
// AXP192 init follows unconditionally for all non-Supreme boards:
if (!PMU) {
    PMU = new XPowersAXP192(PMU_WIRE_PORT, PIN_BOARD_SDA, PIN_BOARD_SCL, I2C_PMU_ADD);
    ...
}
```

### Confirmed by working reference project

`D:\Arduino\Projects\LilyGo T-Beam T22 V1.1 868MHz\LoRa2MQTTBasestation\LoRa2MQTTBasestation_v4\LoRaBoards.cpp`  
— AXP2101 detection block fully commented out; only `XPowersAXP192` used directly. This was the evidence that confirmed AXP2101 must be skipped on T22 V1.1.

### AXP192 LDO mapping (T-Beam T22 V1.1)

| Rail | Channel | Purpose |
|---|---|---|
| LDO2 | `XPOWERS_LDO2` | LoRa radio (SX1276) |
| LDO3 | `XPOWERS_LDO3` | GPS |
| DCDC1 | `XPOWERS_DCDC1` | OLED |

## Build & Flash

```powershell
cd C:\Apps\MeshCore

# Build SX1276 repeater:
pio run -e Tbeam_SX1276_repeater

# Flash (COM16, close serial monitor first):
pio run -e Tbeam_SX1276_repeater -t upload --upload-port COM16
```

## Serial Commands (115200 baud)

```
set name <name>            set repeater name
get name
set tx <dBm>               set TX power
get tx
advert                     broadcast advertisement immediately
ver                        firmware version
board                      board info
stats-packets
stats-radio
stats-core
neighbors
reboot
poweroff / shutdown
```

GPS commands are **top-level** (not `set` subcommands — `set gps off` returns "unknown config"):

```
gps off                    disable GPS in software (persists across reboot)
gps on                     re-enable GPS
gps                        show GPS status (e.g. "on, deactivated, no fix, 0 sats")
gps sync                   force RTC time sync from GPS
gps setloc                 save current GPS fix as static prefs lat/lon
gps advert                 show/set location advertise policy (none/share/prefs)
```

Note: `gps off` deactivates GPS in software but the hardware power rail (LDO3) stays on — the GPS chip remains powered by AXP192.

### Radio params — COMMA separators

`get radio` and `set radio` use **commas** (not spaces) between parameters:

```
get radio
  -> 869.6179809,62.5,8,5     (freq,bw,sf,cr)

set radio 869.6179809,62.5,7,5
  -> OK - reboot to apply
```

**Important**: the default firmware builds with `LORA_SF=8` (from `platformio.ini` `esp32_base` section). The local MeshCore network uses SF7 — different spreading factors are completely orthogonal and invisible to each other. After first flash, always run:

```
set radio 869.6179809,62.5,7,5
reboot
```

Then confirm with `get radio` → should show `...,7,5`.

### Backspace corrupts commands

The serial input handler echoes backspace so the terminal *looks* correct, but it does NOT remove the previous character from the internal buffer — it just appends `\b` (ASCII 8) into the command string. `strtof` then sees a corrupted value (e.g. `869.617\x08...`) and parses a wrong float.

**Rule: never use backspace when typing commands in PuTTY.** If you mistype, press Enter to send the bad command (it will error or parse wrong), then retype the full command cleanly.

## Radio chip — confirmed SX1276

Some internet sources (and TTN forum posts) claim the T-Beam V1.1 has an SX1262. This board has an **SX1276**.

Confirmed by: the ESP-IDF MeshCore sniffer firmware (uses RadioLib `SX1276` class) ran successfully on this board and received live MeshCore packets. The SX1262 variant of the repeater firmware failed even after the AXP192 fix, because the SX1262 and SX1276 have different SPI register maps — wrong chip = immediate init failure.

Always use the `Tbeam_SX1276_repeater` PlatformIO environment for this board.

## Display fix — SH1106 vs SSD1306

The T-Beam T22 V1.1 has an **SH1106** OLED (I2C 0x3C, SDA=21, SCL=22). The MeshCore firmware defaults to `SSD1306Display` — using the wrong driver produces garbage/dots on screen.

Two files changed in `C:\Apps\MeshCore\variants\lilygo_tbeam_SX1276\`:

**`platformio.ini`** (`[LilyGo_TBeam_SX1276]` section):
```ini
# build_flags:
-D DISPLAY_CLASS=SH1106Display        # was SSD1306Display

# build_src_filter:
+<helpers/ui/SH1106Display.cpp>       # was SSD1306Display.cpp

# lib_deps:
adafruit/Adafruit SH110X              # was adafruit/Adafruit SSD1306 @ ^2.5.13
```

**`target.h`**:
```cpp
#ifdef DISPLAY_CLASS
  #include <helpers/ui/SH1106Display.h>   // was SSD1306Display.h
```

MeshCore already has `SH1106Display` built-in (`src/helpers/ui/SH1106Display.cpp`) using `Adafruit_SH1106G` — no custom code needed.

## Antenna Placement — Critical for Range

The T-Beam is deployed at **8m height with the window open**, facing the direction of the nearest network repeater (7221, ~10km NLOS).

Signal levels measured at this location:

| Position | RSSI from 7221 | Margin above SF7 (-123 dBm) |
|---|---|---|
| Low indoors | -121 dBm | 2 dB (marginal, unreliable) |
| 8m, window open | -110 dBm | 13 dB (solid, reliable) |

The 11 dB improvement is from reduced building attenuation and better Fresnel zone clearance. The link is NLOS over 10km through suburban houses (~12m). At low height the 2 dB margin caused the uplink to 7221 to fail silently. At 8m the link closed reliably.

**Lesson**: for NLOS LoRa at 10km, antenna height matters more than TX power. A 1W transmitter would have helped (~+10 dB) but antenna placement achieved similar gains for free.

## Network Relay — How It Works

MeshCore uses FLOOD routing. When the repeater receives a packet it re-broadcasts it after a random delay (0–5× estimated air time). Loop detection prevents relaying the same packet twice. Hop count (path hash count) limits propagation depth.

**Confirmed relay chain** (both directions):

```
Device 1 (inside house, can't reach 7221 directly)
    → T-Beam (hops+1, last_hop=975b)
        → 7221 (10km, hops+1, last_hop=7221)
            → MeshCore network
```

You can confirm the chain is working in the sniffer data by looking for a path that contains `97 5b` followed by `72 21`:
```json
"path": "97 72"        // T-Beam first, then 7221 — uplink working
"path": "72 21 97 5b"  // 7221 first, then T-Beam — inbound working
```

**Signal at the sniffer** when 7221 relays the T-Beam's packet: rssi ≈ -116 dBm (received from 7221 at 10km).

### Packet types

| Route type | Relay behaviour |
|---|---|
| `FLOOD` | Relayed by default (wildcard region) |
| `FLOOD+TRANSPORT` | Only relayed if the repeater has the matching transport key configured |

## Transport Keys — Enabling FLOOD+TRANSPORT Relay

Most MeshCore network traffic uses `FLOOD+TRANSPORT` (regionally scoped with a transport key derived from the channel password). Without the transport keys loaded, the repeater silently drops all `FLOOD+TRANSPORT` packets and only relays plain `FLOOD`.

**To relay the #public channel and other channels:**

1. Open the MeshCore companion app
2. Connect to the T-Beam repeater as admin (default password: `password`)
3. Use the repeater management / channel configuration section to push channel subscriptions to the repeater
4. The app derives transport keys from the channel passwords you already have and loads them into the repeater's region map over the air

Once configured, `FLOOD+TRANSPORT` packets for those channels will be relayed. Confirmed working: after pushing the #public channel key, public messages from the MeshCore network are relayed through the T-Beam.

**Verify in sniffer data**: after configuration, packets with `"route":"FLOOD+TRANSPORT"` should appear with `last_hop=975b` (T-Beam relaying inbound) and 7221 should relay the T-Beam's outbound FLOOD+TRANSPORT packets.

## Notes

- T-Beam T22 V1.1 has no dedicated BOOT button (buttons: PWR, RST, IO38). Web flasher at flasher.meshcore.co.uk cannot auto-reset for flashing — use PlatformIO CLI or esptool.py.
- Both ends of the 10km link are standard 20 dBm nodes — 7221 is NOT a 1W repeater. The -110 dBm received signal at 8m height is consistent with 20 dBm TX at 10km NLOS suburban (Hata model: ~131 dB path loss).
- This fix should ideally be submitted upstream to the MeshCore project.
