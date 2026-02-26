# VS1053B Generic Module — Arduino Guide

**Making the AliExpress VS1053B work with SFEMP3Shield on Arduino Uno**

> Documented and tested by [Ars-Vita Títeres](https://github.com/ArsVitaTiteres) — professional puppet theater integrating Arduino-based audio systems.

---

## Key Findings

- **Full pinout compatibility**: The generic AliExpress VS1053B module is 100% compatible with SFEMP3Shield when mounted directly on Arduino Uno. No pin remapping needed.
- **SdFat 2.x incompatibility**: SdFat 2.x changed its API in ways that break SFEMP3Shield — `chdir()` lost its boolean parameter and `vwd()` was made private. Use the SdFat 1.x bundled inside the SFEMP3Shield repository.
- **SD error 0x43**: Caused by calling `sd.begin()` independently. Pin 9 is shared between VS1053 XCS and SD CS — always use `MP3player.begin()` exclusively.
- **VSLoadUserCode() is private**: The method is declared `private` in the library. Requires a one-line change in `SFEMP3Shield.h` to access plugins from sketches.
- **FLAC does not work on generic modules**: `patchesf.053` loads without error but the decoder never activates (`HDAT1=0x0000`, `SM_RESET` stays set). Likely a firmware version mismatch between the plugin and the chip on generic modules.

---

## Hardware

The generic module plugs directly onto Arduino Uno. Pinout matches SparkFun MP3 Shield exactly:

| Arduino Pin | VS1053B Signal | Function |
|-------------|----------------|----------|
| 2 | DREQ | Data Request |
| 6 | XCS | SCI Chip Select |
| 7 | XDCS | SDI Chip Select |
| 8 | RESET | Hardware Reset |
| 9 | SD_CS | SD Card CS (shared with XCS) |
| 11 | MOSI | SPI data out |
| 12 | MISO | SPI data in |
| 13 | SCK | SPI clock |

> **Pin 9 is shared** between VS1053 XCS and SD CS. Never call `sd.begin()` independently.

---

## Installation

### Why SdFat 2.x breaks everything

SdFat 2.x changed its API incompatibly with SFEMP3Shield. Two specific breaking changes:

- `chdir()` no longer accepts the second boolean parameter
- `vwd()` was declared private, breaking file iteration with `openNext()`

Compiling with SdFat 2.x gives these two errors:

```
no matching function for call to 'SdFat32::chdir(const char [2], bool)'
'FatFile* FatVolume::vwd()' is private within this context
```

**Fix:** Use the SdFat 1.x version bundled inside the SFEMP3Shield repository.

### Step-by-step

1. Uninstall SdFat from Library Manager if installed
2. Download the full SFEMP3Shield repo ZIP:
   `https://github.com/madsci1016/Sparkfun-MP3-Player-Shield-Arduino-Library/archive/refs/heads/master.zip`
3. Extract and copy `SdFat/` and `SFEMP3Shield/` to your Arduino `libraries/` folder
4. Copy all `.053` files from the `plugins/` folder to the **root of your SD card**
5. Restart Arduino IDE

---

## Error 0x43 — Why sd.begin() Fails

Pin 9 is shared between the SD card CS and the VS1053 XCS. When the VS1053 is active on the SPI bus, it interferes with SD initialization.

**Fix:** Use only `MP3player.begin()` — it initializes both devices in the correct order.

```cpp
#include <SPI.h>
#include <SdFat.h>
#include <SFEMP3Shield.h>

SdFat sd;
SFEMP3Shield MP3player;

void setup() {
  uint8_t result = MP3player.begin();
  // 0 = OK (patches.053 loaded)
  // 6 = OK but patches.053 not found on SD
  MP3player.setVolume(20, 20); // 0=max, 255=silence
}
```

---

## Plugin System

### Required modification: make VSLoadUserCode() public

In the repository version of SFEMP3Shield, `VSLoadUserCode()` is declared `private`. Move it to `public` in `SFEMP3Shield.h`:

```cpp
// In the public: section, after ADMixerLoad:
uint8_t ADMixerLoad(char*);
void ADMixerVol(int8_t);
uint8_t VSLoadUserCode(char*);  // ← move here from private:
```

### Additional fix: waitForReset()

Some plugins activate `SM_RESET` internally during loading. Add this to `SFEMP3Shield.h` (private section) and `SFEMP3Shield.cpp`:

```cpp
// SFEMP3Shield.h — private section:
void waitForReset(uint16_t timeout_ms);

// SFEMP3Shield.cpp — add before VSLoadUserCode(), call at end of it:
void SFEMP3Shield::waitForReset(uint16_t timeout_ms) {
  unsigned long t = millis();
  while ((Mp3ReadRegister(SCI_MODE) & SM_RESET) && (millis() - t < timeout_ms));
}

// At the end of VSLoadUserCode(), before return 0:
  track.close();
  waitForReset(500);
  return 0;
```

### Verified plugin status

All `.053` files must be copied to the root of the SD card.

| Plugin | Access method | Function | Status |
|--------|--------------|----------|--------|
| `patches.053` | Automatic in `begin()` | General firmware patches | ✔ Verified |
| `admxster.053` | `ADMixerLoad()` | Stereo ADC mixer | ✔ Verified |
| `admxleft.053` | `ADMixerLoad()` | Left-channel ADC output | ✔ Verified (loads) |
| `admxrght.053` | `ADMixerLoad()` | Right-channel ADC output | ✔ Verified (loads) |
| `admxmono.053` | `ADMixerLoad()` | Mono ADC output | ✔ Verified (loads) |
| `admxswap.053` | `ADMixerLoad()` | Swap L/R ADC channels | ✔ Verified (loads) |
| `patchesf.053` | `VSLoadUserCode()`* | FLAC decoder | ✘ Not working on generic modules |

> **FLAC limitation**: `patchesf.053` loads without error but `HDAT1=0x0000` and `SM_RESET` never clears during playback. The chip consumes the file data without producing audio. Likely a firmware version mismatch between the plugin and the VS1053B chip on generic AliExpress modules.

### ADMixer usage example

```cpp
void setup() {
  MP3player.begin();          // loads patches.053 automatically
  MP3player.setVolume(20, 20);

  uint8_t r = MP3player.ADMixerLoad("admxster.053");
  if (r == 0) {
    MP3player.ADMixerVol(-15); // -15 dB input gain
  }
  MP3player.playMP3("track001.mp3");
}
```

---

## Supported Audio Formats

| Format | Plugin required | Notes |
|--------|----------------|-------|
| MP3 | None (native) | Verified |
| WAV (PCM/ADPCM) | None (native) | Verified |
| OGG Vorbis | None (native) | Per datasheet, not tested |
| AAC / WMA | None (native) | Per datasheet, not tested |
| MIDI | None (native) | Per datasheet, not tested |
| FLAC | `patchesf.053` | Not working on generic modules |

---

## API Reference

| Function | Description |
|----------|-------------|
| `begin()` | Init SD + VS1053. Loads patches.053. Returns 0=OK, 6=no patches.053 |
| `playMP3(char* file)` | Play file. Returns 0=OK, 1=busy, 2=SD fail, 3=reset, 4=not found |
| `stopTrack()` | Stop playback and close file |
| `isPlaying()` | Returns 1 if actively playing |
| `setVolume(L, R)` | 0=max, 255=silence. setVolume(20,20) = -10 dB |
| `ADMixerLoad(char*)` | Load admx*.053 plugin. Public by design |
| `ADMixerVol(int8_t)` | ADC input gain in dB |
| `VSLoadUserCode(char*)` | Load generic plugin. Requires moving to public in .h |
| `vs_init()` | Reset chip without reinitializing SD |
| `getAudioInfo()` | Print internal chip registers via serial (diagnostic) |

---

## Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `chdir: no matching function` + `vwd() is private` | SdFat 2.x installed. Remove it, use SdFat 1.x from SFEMP3Shield repo |
| `VSLoadUserCode is private within this context` | Move VSLoadUserCode() from private to public in SFEMP3Shield.h |
| FLAC loads but no audio | patchesf.053 incompatible with generic module. Not fixable without firmware update |
| SD error 0x43 | sd.begin() called independently. Use only MP3player.begin() |

---

## References

- SFEMP3Shield repository: https://github.com/madsci1016/Sparkfun-MP3-Player-Shield-Arduino-Library
- VS1053B Datasheet: VLSI Solution Oy
- SdFat 1.x: included in SFEMP3Shield repository

---

*Ars-Vita Títeres — Cuautepec de Hinojosa, Hidalgo, México*  
*License: CC0 1.0 Universal (Public Domain)*
