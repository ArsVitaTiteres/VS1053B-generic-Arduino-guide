# VS1053B Generic Module + Arduino Uno
## Complete Guide: SFEMP3Shield with AliExpress modules

> **By Ars-Vita Títeres — Published for the Arduino community**

---

Most documentation for the VS1053B assumes you have the original SparkFun MP3 Player Shield. This guide covers what happens when you have a **generic AliExpress module** — and specifically documents two undocumented problems that will block you if you don't know about them.

---

## Table of Contents

1. [The Key Finding](#1-the-key-finding)
2. [Hardware & Pinout](#2-hardware--pinout)
3. [Library Installation](#3-library-installation)
4. [The 0x43 SD Error — Explained](#4-the-0x43-sd-error--explained)
5. [Plugins — Extending VS1053B Capabilities](#5-plugins--extending-vs1053b-capabilities)
6. [Supported Audio Formats](#6-supported-audio-formats)
7. [API Quick Reference](#7-api-quick-reference)
8. [Troubleshooting Table](#8-troubleshooting-table)
9. [References](#9-references)

---

## 1. The Key Finding

> ✅ **The generic AliExpress VS1053B module is fully compatible with SFEMP3Shield without modifying any pins**, as long as the module is plugged directly onto the Arduino Uno. The pinout matches the original SparkFun Shield.

Two undocumented problems will block you:

- **The 0x43 error**: Pin 9 is shared between the VS1053 XCS and the SD card CS. Trying to initialize the SD alone (without the VS1053) fails because the VS1053 interferes on the SPI bus.
- **SdFat 2.x incompatibility**: The current SdFat in Arduino's Library Manager removes `SdFatUtil.h` and breaks compilation. You need the version bundled inside the SFEMP3Shield repository.

---

## 2. Hardware & Pinout

### Module pinout vs. Arduino Uno

| Module pin | Arduino pin | Function | Library macro |
|---|---|---|---|
| DREQ | 2 (INT0) | Data Request — chip asks for more data | `MP3_DREQ` |
| XCS | 6 | Control Chip Select (SCI registers) | `MP3_XCS` |
| XDCS | 7 | Data Chip Select (audio stream) | `MP3_XDCS` |
| X_RESET | 8 | Reset (active low) | `MP3_RESET` |
| CS / SD_CS | 9 | **Shared** — VS1053 XCS + SD card CS | `SD_SEL` |
| MOSI | 11 | SPI data out | — |
| MISO | 12 | SPI data in | — |
| SCK | 13 | SPI clock | — |
| VCC | 5V | Power | — |
| GND | GND | Ground | — |

> **Note:** MOSI, MISO, SCK are the hardware SPI pins on the Uno. When you plug the shield directly onto the Arduino, these connect automatically by physical position.

### SD card requirements

- Format: **FAT32** (not exFAT, not NTFS)
- Filenames: **8.3 format** — max 8 characters before the dot, 3 after, no spaces or special characters
- Recommended formatter: [SD Memory Card Formatter](https://www.sdcard.org/downloads/formatter/)

---

## 3. Library Installation

### Why SdFat 2.x breaks everything

The current SdFat in Arduino's Library Manager is version 2.x, which removed `SdFatUtil.h`. Compiling with it gives:

```
fatal error: SdFatUtil.h: No such file or directory
```

The fix: use the SdFat version **bundled inside the SFEMP3Shield repository** — it's the correct 1.x version.

### Step-by-step installation

**Step 1** — If you have SdFat installed from Library Manager, uninstall it:
`Tools → Manage Libraries → search "SdFat" → Uninstall`

**Step 2** — Download the full SFEMP3Shield repository as ZIP:
```
https://github.com/madsci1016/Sparkfun-MP3-Player-Shield-Arduino-Library/archive/refs/heads/master.zip
```

**Step 3** — Unzip it. You'll see this structure:
```
Sparkfun-MP3-Player-Shield-Arduino-Library-master/
  ├── SdFat/          ← copy this folder
  ├── SFEMP3Shield/   ← copy this folder
  └── plugins/        ← copy the .053 files to your SD card root
```

**Step 4** — Copy `SdFat` and `SFEMP3Shield` to your Arduino libraries folder:
```
# Windows
C:\Users\{youruser}\Documents\Arduino\libraries\

# Linux / macOS
~/Arduino/libraries/
```

**Step 5** — Copy all `.053` files from the `plugins` folder to the **root of your SD card**. These are needed for FLAC, real-time MIDI, and other advanced features.

**Step 6** — Restart Arduino IDE. Verify: `File → Examples → SFEMP3Shield`

> ⚠️ Do **not** reinstall SdFat from Library Manager after this. The bundled version is the correct one.

---

## 4. The 0x43 SD Error — Explained

If you try to initialize the SD card alone (for example, with a diagnostic sketch using only SdFat), you'll get:

```
begin() failed
Do not reformat the SD.
No card, wrong chip select pin, or wiring error?
SdError: 0X43,0X5
```

### Why this happens

Pin 9 serves two purposes on this module: it's the VS1053 XCS (chip select for control commands) **and** the SD card CS. When you try to initialize the SD without first initializing the VS1053, the VS1053 remains active on the SPI bus and corrupts the SD communication.

The error `0x43` means timeout — the SD tries to communicate but receives corrupted responses from the VS1053. This is visible because other CS pins return `0x01/0xFF` (no response at all), but pin 9 returns `0x43/0x05`, confirming something responds but incorrectly.

### The fix

> ✅ **Always initialize both devices together using `MP3player.begin()`**, which handles the correct activation sequence internally. Never call `sd.begin()` alone on this module.

```cpp
// CORRECT
if (!sd.begin(SD_SEL, SPI_FULL_SPEED)) sd.initErrorHalt();
if (!sd.chdir("/")) sd.errorHalt("sd.chdir");
MP3player.begin();  // initializes VS1053 and takes control of the shared SPI bus

// WRONG — fails with 0x43 on this module
// sd.begin(9);  // VS1053 still active, interferes on bus
```

---

## 5. Plugins — Extending VS1053B Capabilities

### What are plugins?

The VS1053B contains a programmable DSP called VS_DSP. Its factory capabilities cover the most common formats, but can be extended by loading small programs into its internal RAM. These are called **plugins** or **patches**.

**Plugins are volatile**: they live in the chip's RAM, which is erased on every reset or power-off. They must be reloaded in `setup()` after every `MP3player.begin()`. They are stored on the SD card as `.053` files and loaded with `VSLoadUserCode()`.

> ⚠️ **Plugins can only be loaded when the player is NOT playing** (`isPlaying() == 0`). Loading during playback returns error 1.

### Plugin catalog

Copy all these `.053` files from the repository's `plugins/` folder to the **root of your SD card**:

| File | Category | Function |
|---|---|---|
| `patches.053` | General patches | Accumulated firmware fixes. Improves decoding of various formats. **Recommended to always load.** |
| `patchesf.053` | Decoding | **Enables FLAC decoding.** Without this, VS1053B cannot play FLAC files. |
| `rtmidi.053` | MIDI | Activates the real-time MIDI synthesizer. Allows sending MIDI notes from Arduino; the chip synthesizes them directly without audio files. |
| `pcm.053` | Audio | Enables feeding raw PCM data directly to the chip's DAC, bypassing the format decoder. |
| `admxster.053` | ADC Mixer | Routes both ADC channels (analog input) to left and right outputs. |
| `admxswap.053` | ADC Mixer | Swaps left and right ADC channels on outputs. |
| `admxleft.053` | ADC Mixer | Routes MIC/LINE1 to both outputs. |
| `admxrght.053` | ADC Mixer | Routes LINE2 to both outputs. |
| `admxmono.053` | ADC Mixer | Mixes both input channels to mono on both outputs. |

> **Note:** The `admx*.053` mixer plugins are for modules with microphone or line input. The generic AliExpress module has a 3.5mm mic jack, so these plugins are functional.

### Loading a plugin — `VSLoadUserCode()`

```cpp
// Load general patches (recommended in every project)
uint8_t result = MP3player.VSLoadUserCode("patches.053");
if (result != 0) {
  Serial.print("Plugin load error: ");
  Serial.println(result);
}
```

**Return codes for `VSLoadUserCode()`:**

| Code | Meaning |
|---|---|
| 0 | Success — plugin loaded into VS1053B RAM |
| 1 | Player is busy playing. Call `stopTrack()` first |
| 2 | `.053` file not found on SD. Check name and root location |
| 3 | VS1053B is in reset state. Wait for `begin()` to complete |

### Use cases

#### FLAC playback

```cpp
void setup() {
  sd.begin(SD_SEL, SPI_FULL_SPEED);
  sd.chdir("/");
  MP3player.begin();

  // Load FLAC plugin
  if (MP3player.VSLoadUserCode("patchesf.053") != 0) {
    Serial.println("FLAC plugin not found on SD");
  }
}

void loop() {
  MP3player.playMP3("musica.fla");  // .fla extension (8.3 format)
}
```

> **Note:** FLAC files must use the `.fla` extension (not `.flac`) to comply with SdFat's 8.3 filename requirement. `isFnMusic()` recognizes `.fla` as a valid audio extension.

#### General patches (best practice)

```cpp
void setup() {
  sd.begin(SD_SEL, SPI_FULL_SPEED);
  sd.chdir("/");
  MP3player.begin();
  MP3player.VSLoadUserCode("patches.053");  // always recommended
}
```

#### ADC Mixer (microphone/line input)

```cpp
// Load stereo mixer (ADC input → L and R outputs)
MP3player.ADMixerLoad("admxster.053");

// Set input attenuation: -3 to -31 dB
// Values out of range disable the mixer
MP3player.ADMixerVol(-10);  // -10 dB input attenuation
```

> **Note:** ADC Mixer plugins use `ADMixerLoad()` instead of `VSLoadUserCode()`. This function also configures the input mode (microphone or line).

### Important plugin rules

- Plugins are **volatile**: lost on every reset. Always load them in `setup()`.
- Only **one plugin active at a time** in chip RAM. Loading a second plugin overwrites the first.
- All `.053` files must be in the **SD card root**, not in subfolders.
- **Do not load plugins while audio is playing** (`isPlaying() == 1`). Stop first with `stopTrack()`.
- The `admx*.053` plugins are specifically for analog input and do not affect file playback.

---

## 6. Supported Audio Formats

| Format | SD extension | Requires plugin | Notes |
|---|---|---|---|
| MP3 | `.mp3` | No (native) | CBR and VBR. Up to 320 kbps, 48 kHz. |
| WAV | `.wav` | No (native) | PCM and IMA ADPCM. Mono and stereo. |
| OGG Vorbis | `.ogg` | No (native) | High quality at low bitrates. Patent-free. |
| AAC | `.aac` | No (native) | ISO/IEC 13818-7 and 14496-3. |
| WMA | `.wma` | No (native) | Windows Media Audio versions 2, 7, 8, 9. |
| MIDI | `.mid` | No (native) | General MIDI and SP-MIDI format 0. |
| FLAC | `.fla` | Yes — `patchesf.053` | Must use `.fla` extension (8.3). Load plugin first. |

`isFnMusic()` automatically recognizes: `.mp3`, `.aac`, `.wma`, `.wav`, `.fla`, `.mid`, `.ogg`

---

## 7. API Quick Reference

### Minimum required structure

```cpp
#include <SPI.h>
#include <SdFat.h>
#include <SdFatUtil.h>
#include <SFEMP3Shield.h>

SdFat sd;               // exact name required
SFEMP3Shield MP3player; // exact name required

void setup() {
  if (!sd.begin(SD_SEL, SPI_FULL_SPEED)) sd.initErrorHalt();
  if (!sd.chdir("/")) sd.errorHalt("sd.chdir");
  MP3player.begin();
  MP3player.VSLoadUserCode("patches.053"); // recommended
}
```

### Playback functions

| Function | Returns | Description |
|---|---|---|
| `playMP3(char* name, uint32_t ms=0)` | uint8_t | Play by filename. ms = start offset in milliseconds. |
| `playTrack(uint8_t num, uint32_t ms=0)` | uint8_t | Play by number (track001.mp3 = 1). |
| `stopTrack()` | void | Stop playback and flush buffer. |
| `pauseMusic()` | void | Pause keeping position. |
| `resumeMusic()` | bool | Resume from where paused. |
| `isPlaying()` | uint8_t | 1 if playing, 0 if not. |
| `skip(int32_t ms)` | uint8_t | Skip forward (+) or backward (-) in ms. |
| `skipTo(uint32_t ms)` | uint8_t | Jump to absolute position in ms. |
| `VSLoadUserCode(char* file)` | uint8_t | Load a `.053` plugin from SD root. |
| `ADMixerLoad(char* file)` | uint8_t | Load an `admx*.053` mixer plugin. |
| `ADMixerVol(int8_t dB)` | void | Input attenuation -3 to -31 dB. |

**playMP3() return codes:**

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Player busy — call `stopTrack()` first |
| 2 | Error opening file |
| 4 | File not found on SD |
| 6 | Patch file not found (not critical for MP3) |

### Volume

```cpp
MP3player.setVolume(0, 0);     // maximum volume
MP3player.setVolume(40, 40);   // -20 dB
MP3player.setVolume(254, 254); // silence
// 0 = max, 254 = silence, each unit = 0.5 dB attenuation
// 0xFFFF = analog power-down mode
```

### Equalization

| Function | Range | Description |
|---|---|---|
| `setBassFrequency(uint16_t hz)` | 20–150 Hz | Bass enhancement cutoff frequency (VSBE). |
| `setBassAmplitude(uint8_t dB)` | 0–15 dB | Bass gain. |
| `setTrebleFrequency(uint16_t hz)` | 0–15000 Hz | Treble enhancement start frequency. |
| `setTrebleAmplitude(int8_t dB)` | -8 to +7 dB | Treble gain. Negative = attenuation. |

### List files on SD

```cpp
SdFile file;
char filename[13];
sd.chdir("/", true);
while (file.openNext(sd.vwd(), O_READ)) {
  file.getName(filename, sizeof(filename));
  if (isFnMusic(filename)) {
    Serial.println(filename);
  }
  file.close();
}
```

---

## 8. Troubleshooting Table

| Symptom | Error / code | Cause and fix |
|---|---|---|
| Won't compile | `SdFatUtil.h not found` | SdFat 2.x installed. Use the version bundled in the SFEMP3Shield repository. |
| SD won't init alone | `0x43, 0x05` | VS1053 interferes on SPI bus. Always use `MP3player.begin()`. |
| SD no response on any pin | `0x01, 0xFF` | SD not inserted or damaged. Check card and FAT32 format. |
| `begin()` returns 6 | — | `patches.053` missing from SD. Not critical for MP3 but recommended. |
| `playMP3()` returns 4 | — | File not found. Check 8.3 filename with no spaces. |
| `playMP3()` returns 1 | — | Player busy. Call `stopTrack()` first. |
| FLAC won't play | — | `patchesf.053` missing from SD or not loaded with `VSLoadUserCode()`. |
| `VSLoadUserCode()` returns 1 | — | Tried to load plugin while playing. Stop first. |
| `VSLoadUserCode()` returns 2 | — | `.053` file not found. Must be in SD root, not a subfolder. |
| No audio output | — | Volume at silence. Call `setVolume(20, 20)`. |

---

## 9. References

- [VS1053B Datasheet — VLSI Solution](https://www.vlsi.fi/en/products/vs1053.html)
- [VS1053B Patches and Plugins — VLSI Solution](https://www.vlsi.fi/en/support/software/vs10xxpatches.html)
- [SFEMP3Shield Repository](https://github.com/madsci1016/Sparkfun-MP3-Player-Shield-Arduino-Library)
- [SFEMP3Shield API Documentation](https://mpflaga.github.io/Sparkfun-MP3-Player-Shield-Arduino-Library)
- [SparkFun MP3 Player Shield Hookup Guide](https://learn.sparkfun.com/tutorials/mp3-player-shield-hookup-guide-v15)
- [SD Memory Card Formatter](https://www.sdcard.org/downloads/formatter/)

---

*This guide was built from hands-on experience with real hardware. If you find errors or have corrections, contributions are welcome.*
