# ESP32-C3 Custom Register-Level GPIO Driver

Instead of relying on heavy framework functions like `digitalWrite()`, this project talks directly to the chip's raw memory registers to toggle pins. It's basically a mini custom Hardware Abstraction Layer (HAL) built from scratch to see how the silicon works under the hood.

## 🤔 How It Works

To get a pin to do anything on this chip, you have to deal with two different register areas:

1. **The Setup (`pinconf`):** By default, the ESP32-C3 locks the physical pins for factory debugging tools. The driver flips **Bit 12** on the pin's config register to unlock it and route it to the digital I/O engine. Then, it updates the global master direction register (`GPIO_ENABLE_REG`) to set it as an input or output.
2. **The Runtime (`SetHigh / SetLow`):** To change the pin state later, the driver uses a bitmask to toggle only the specific bit assigned to that pin inside the shared output data register (`GPIO_DATA_REG`), leaving all other pins completely untouched.

To keep things efficient and completely avoid dynamic memory fragmentation, the driver uses standard C++ object-oriented structure but initializes the instances inside a pre-allocated static buffer using `placement new`.

