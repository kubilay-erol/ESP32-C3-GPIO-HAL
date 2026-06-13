# ESP32-C3 Bare-Metal Register DigIOReg Driver

A lightweight, zero-dependency, object-oriented C++ driver written from scratch to control the General Purpose Input/Output (GPIO) pins on the **Espressif ESP32-C3** (specifically tested on the SuperMini form factor) via direct register manipulation.

This project completely bypasses the standard Arduino HAL overhead (like `digitalWrite`) to interface directly with the silicon hardware stencils, providing deterministic, constant-time performance.

## 🚀 Key Features

* **True Object-Oriented Hardware Isolation:** Every physical pin gets its own dedicated class instance using a memory-safe `placement new` factory layout.
* **Direct Register Manipulation:** Bypasses framework abstraction layers by speaking directly to the ESP32-C3 memory map (`0x60004000` base).
* **Constant-Time Execution ($O(1)$):** Hardware initialization avoids slow, branch-predictive `if-else` block trees during data routing.
* **Zero Dynamic Allocation Overhead:** Uses a custom pre-allocated buffer array (`ledBuffer`) to guarantee safety in memory-constrained embedded environments.

---

## 🛠️ Hardware Architecture & Memory Mapping

The driver handles the two distinct execution phases required to control physical pins on the RISC-V ESP32-C3 core:

1. **The Setup Phase (Pin Activation & Routing):** * Modifies the individual configuration register (`pinconf`) at `0x60009004 + (4 * Pin#)`. Setting **Bit 12 (`MCU_SEL`) to 1** physically unhooks the pin from internal factory JTAG/testing modules and routes it to the digital engine.
* Flips the corresponding bit in the Master Direction register (`GPIO_ENABLE_REG` at `0x60004020`) to lock the pin into `Input` or `Output` modes.


2. **The Runtime Phase (Blazing-Fast Toggling):**
* Employs unique `pinmask` configurations (`1 << Pin#`) inside `SetHigh()` and `SetLow()` to isolate and change single bit structures inside the global Data Output Register (`GPIO_DATA_REG` at `0x60004004`) without corrupting adjacent pin states.



---

## 📦 Project File Structure

```text
├── include/
│   └── DigIOReg.h      # Register macros, Mode enums, and Class declarations
├── src/
│   ├── DigIOReg.cpp    # Direct register bit manipulation logic 
│   └── main.cpp        # Onboard blue LED (GPIO 8) blink implementation loop
└── platformio.ini      # Custom environment overrides optimized for C++17

```

---

## 💻 Code Quick-Start

To spin this up yourself on an ESP32-C3 SuperMini (where the onboard Blue LED is hardwired to **GPIO 8**), implement the driver initialization like this:

```cpp
#include "DigIOReg.h"

// Statically pre-allocate memory for the object instance
uint8_t ledBuffer[sizeof(DigIOReg)];

void setup() {
    // Instantiate Pin 8 as an Output using the placement new factory
    DigIOReg* led = DigIOReg::digioreg(
        DigIOReg::Mode::output, 
        GPIO8_MODE_REG,          // Pin 8 individual configuration register
        GPIO_DATA_REG,           // Shared master data register
        (1 << 8),                // Bitmask targeting Bit 8
        ledBuffer                // Memory array destination
    );

    // Bare-metal hardware execution loop
    while(true) {
        led->SetHigh(); 
        for(volatile uint32_t i = 0; i < 500000; i++); // Volatile timing delay

        led->SetLow();  
        for(volatile uint32_t i = 0; i < 500000; i++); 
    }
}

void loop() {
    // Unused: Hijacked by bare-metal loop in setup()
}

```

---

## ⚙️ Compiling & Uploading via VS Code + PlatformIO

This project uses the `framework = arduino` configuration purely as an automatic toolchain fetcher to establish cross-compilation environments for the RISC-V targets without using any Arduino functions.

1. Open this repository workspace inside **VS Code** with the **PlatformIO IDE** extension installed.
2. Connect your ESP32-C3 SuperMini using a dependable USB-C data cable.
3. Click the **PlatformIO: Build** ($\checkmark$) button at the bottom status bar to confirm compilation integrity under `gnu++17`.
4. Click **PlatformIO: Upload** ($\rightarrow$) to flash the binary image directly into the chip's internal flash.

> 💡 **Troubleshooting Boot / Download Mode:** If your machine fails to connect to the board's native CDC USB port during compiling, hold the **BOOT** button, tap the **RST** button once, and release **BOOT**. This forces the internal ROM directly into the factory bootloader mode for seamless flashing.
