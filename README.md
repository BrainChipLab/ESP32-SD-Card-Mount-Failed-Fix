# ESP32-SD-Card-Mount-Failed-Fix
Code snippets and wiring checklist for fixing ESP32 and ESP32-CAM "Card Mount Failed" SD card errors.


# ESP32 "Card Mount Failed" — SD Card Code & Wiring Checklist

Code snippets and wiring guide for fixing `"Card Mount Failed"` and `"No SD card attached"` on standard ESP32 boards (SPI) and ESP32-CAM (`SD_MMC`).

---

### 1. Standard SPI SD Card Test (with 1 MHz Lowered Speed for Long Wires)
Default SPI pins on classic ESP32 DevKit: `CS = 5`, `MOSI = 23`, `MISO = 19`, `SCK = 18`.

```cpp
#include "FS.h"
#include "SD.h"
#include "SPI.h"

const int SD_CS = 5;  // the CS pin you really wired

void setup() {
  Serial.begin(115200);
  // SD.begin(ssPin, spi, frequency): default frequency is 4 MHz
  if (!SD.begin(SD_CS, SPI, 1000000)) {  // 1 MHz for long wires
    Serial.println("Card Mount Failed");
    return;
  }
  if (SD.cardType() == CARD_NONE) {
    Serial.println("No SD card attached");
    return;
  }
  Serial.printf("Card size: %llu MB\n", SD.cardSize() / (1024 * 1024));
}

void loop() {}
```

---

### 2. Separate SPI Bus (`HSPI`) for Shared SPI Conflicts
If another SPI device or a cheap SD module is holding the `MISO` line, put the SD card on its own SPI bus (pins from the official `SPI_Multiple_Buses` example):

```cpp
#include "FS.h"
#include "SD.h"
#include "SPI.h"

SPIClass sdSPI(HSPI);

void setup() {
  Serial.begin(115200);
  sdSPI.begin(25, 26, 27, 32);          // SCK, MISO, MOSI, CS
  if (!SD.begin(32, sdSPI, 1000000)) {  // CS, bus, 1 MHz
    Serial.println("Card Mount Failed");
    return;
  }
  Serial.println("SD Card Mounted on HSPI!");
}

void loop() {}
```

---

### 3. ESP32-CAM (`SD_MMC` 1-Bit Mode)
The AI-Thinker ESP32-CAM slot uses `SD_MMC`, not `SD.h`. Passing `true` enables 1-bit mode and frees up `GPIO 4` (the bright onboard Flash LED):

```cpp
#include "FS.h"
#include "SD_MMC.h"

void setup() {
  Serial.begin(115200);
  // "/sdcard" = mount point, true = 1-bit mode (GPIO 4 / flash LED not used)
  if (!SD_MMC.begin("/sdcard", true)) {
    Serial.println("Card Mount Failed");
    return;
  }
  Serial.println("ESP32-CAM SD Card Mounted!");
}

void loop() {}
```

---

### 4. Quick Wiring & Formatting Checklist
* **Blue Module (with 3.3V regulator & level shifter):** `VCC -> 5V/VIN` | `GND -> GND` | `CS -> GPIO 5` | `MOSI -> GPIO 23` | `MISO -> GPIO 19` | `SCK -> GPIO 18`
* **Bare Micro-SD Adapter (no regulator):** `VCC -> 3V3 ONLY` (never 5V) | `GND -> GND` | Same 4 SPI pins.
* **Card Size & Format:** Use an **SDHC card (32 GB or smaller)** formatted as **FAT32** (cards 64 GB+ default to `exFAT`, which isn't supported by default).
* **See the Exact Error Reason:** In Arduino IDE, set `Tools -> Core Debug Level -> Verbose` before uploading.
