# **⌨️ How I Built a 16-Key Wireless Bluetooth Macro Pad with the Raspberry Pi Pico 2 W**

Commercial macro keypads and stream decks are handy, but they are often quite expensive and locked into bloated, closed-source software. When the new Raspberry Pi Pico 2 W (RP2350) launched with built-in Wi-Fi and Bluetooth, I immediately knew what my next weekend project would be: a fully standalone, wireless Bluetooth BLE macro pad featuring 16 mechanical keys.

In this article, I will take you through the complete build process: from smartly expanding GPIO pins using the MCP23017, to avoiding some tricky software pitfalls on the new RP2350 chip.

📺 Watch the Full Project Video on YouTube: https://youtu.be/69uvluCrlgI

A great tool for building electronic circuits and [**PCB layout**](https://design.jlcpcb.com/?from=makerio_layout) workflows.

## **💡 The Concept & Architecture**

The goal was simple: a compact 16-key macro keyboard that connects directly to my PC via Bluetooth Low Energy (BLE). No USB cables cluttering my desk, and no special background software required on Windows; the device simply acts as a fully compliant Bluetooth HID keyboard.

### **Why the MCP23017 Port Expander?**

The Raspberry Pi Pico 2 W has plenty of pins, but to keep your wiring clean and manageable, you definitely want to avoid a tangled mess of 16 individual signal wires running to your microcontroller.

This is where the MCP23017 comes in:

- It communicates via the I²C protocol.
- It reads 16 digital inputs using just two data lines (SDA and SCL).
- It features internal $100k\Omega$ pull-up resistors, allowing us to wire all 16 buttons directly between the chip and ground (GND). No hassle with loose resistors on your breadboard!

## **🛠️ Required Components**

- Raspberry Pi Pico 2 W (RP2350 with built-in Bluetooth)
- MCP23017 I²C 16-Bit GPIO Expander Module
- 16× Push buttons (mechanical switches or tactile buttons)
- Breadboard & jumper wires
- Micro-USB cable (for initial programming)

## **🔌 Wiring Schematic & Pinout**

The wiring is split into two logical parts: communication between the Pico 2 W and the MCP23017, and hooking up the 16 physical buttons.

### **Table 1: Connection between Pico 2 W and MCP23017**

**⚠️ Critical Hardware Note:**

On the Pico 2 W, pins GP6 and GP7 are physically tied to the internal hardware controller i2c1. Therefore, your code **must** use Wire1 instead of Wire.

| **MCP23017 Pin** | **Pico 2 W Connection** | **Physical Pin** | **Function**                                |
| ---------------- | ----------------------- | ---------------- | ------------------------------------------- |
| **VCC / VDD**    | 3V3(OUT)                | Pin 36           | 3.3V Power supply for the module            |
| ---              | ---                     | ---              | ---                                         |
| **GND / VSS**    | GND                     | Pin 38           | Common ground                               |
| ---              | ---                     | ---              | ---                                         |
| **SDA**          | GP6                     | Pin 9            | I²C Data (i2c1 via Wire1)                   |
| ---              | ---                     | ---              | ---                                         |
| **SCL**          | GP7                     | Pin 10           | I²C Clock (i2c1 via Wire1)                  |
| ---              | ---                     | ---              | ---                                         |
| **A0, A1, A2**   | GND                     | Pin 38           | Address selection $\to$ I²C Address: 0x20 |
| ---              | ---                     | ---              | ---                                         |
| **RESET**        | 3V3(OUT)                | Pin 36           | Active-low; tied to 3.3V to remain active   |
| ---              | ---                     | ---              | ---                                         |

### **Table 2: Connection and Function Mapping of the 16 Buttons**

Each button has one leg connected to the specified MCP23017 pin, and the other leg connected to the common ground (GND):

| **Button**    | **MCP23017 Pin** | **Assigned Action** | **Function / Description** |
| ------------- | ---------------- | ------------------- | -------------------------- |
| **Button 1**  | GPA0             | KEY_F13             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 2**  | GPA1             | KEY_F14             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 3**  | GPA2             | KEY_F15             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 4**  | GPA3             | KEY_F16             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 5**  | GPA4             | KEY_F17             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 6**  | GPA5             | KEY_F18             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 7**  | GPA6             | KEY_F19             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 8**  | GPA7             | KEY_F20             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 9**  | GPB0             | KEY_F21             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 10** | GPB1             | KEY_F22             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 11** | GPB2             | KEY_F23             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 12** | GPB3             | KEY_F24             | Virtual macro key          |
| ---           | ---              | ---                 | ---                        |
| **Button 13** | GPB4             | Ctrl + C            | Copy (Shortcut)            |
| ---           | ---              | ---                 | ---                        |
| **Button 14** | GPB5             | Ctrl + V            | Paste (Shortcut)           |
| ---           | ---              | ---                 | ---                        |
| **Button 15** | GPB6             | Ctrl + Z            | Undo (Shortcut)            |
| ---           | ---              | ---                 | ---                        |
| **Button 16** | GPB7             | Ctrl + S            | Save (Shortcut)            |
| ---           | ---              | ---                 | ---                        |

## **⚠️ Three Technical Pitfalls (and How They Were Solved)**

During development, I ran into three interesting technical challenges:

### **1\. The I²C Pitfall: Wire vs. Wire1**

The Pico features two separate hardware I²C controllers: i2c0 and i2c1. Pins GP6 and GP7 belong to i2c1. If you simply call standard Wire.begin() in Arduino, the I²C bus locks up immediately and freezes the Serial Monitor. By explicitly using Wire1 and setting the bus speed to a safe $50kHz$, communication over breadboard jumper wires runs flawlessly and noise-free.

### **2\. The Mystery of the "Stuck" Keys (Bluetooth Timing)**

With a wired USB keyboard, you fire off a press signal and a release signal in a split second. Bluetooth Low Energy, however, relies on so-called **Connection Intervals** (typically around 30–50 ms under Windows).

If your code fires Ctrl press $\to$ V press $\to$ V release $\to$ Ctrl release within a few milliseconds, the Bluetooth buffer can get overwhelmed. Windows then misses the "release" packet, making the computer think you are holding the key down indefinitely (resulting in an endless stream of zzzzzzzz). By building in a deliberate pause of 50 to 60 ms between each action and always ending with a bundled releaseAll(), every shortcut operates 100% reliably.

### **3\. The Secret Power of F13 through F24**

Standard keyboards have keys from F1 to F12. However, Windows has long supported virtual keys F13 through F24. These keys are absolute gold for streamers and gamers: you can assign them in applications like Discord (Push-to-Talk) or OBS Studio without ever conflicting with existing shortcuts in your browser or game.

## **🛠️ Configuring Software & Arduino IDE**

To compile the code for the new Pico 2 W, you need the community-standard Earle Philhower Arduino-Pico core:

1. **Add Board Manager URL:** In the Arduino IDE, go to **File > Preferences** and add Earle Philhower's RP2040/RP2350 JSON URL:  
   \[<https://github.com/earlephilhower/arduino-pico/releases/download/global/package_rp2040_index.json\>](<https://github.com/earlephilhower/arduino-pico/releases/download/global/package_rp2040_index.json>)
2. **Install Package:** Open the **Boards Manager** and install the **Raspberry Pi Pico/RP2040/RP2350** package (version 6.3.0 or newer).
3. **Correct Settings:**
   - **Board:** Raspberry Pi Pico 2 W (Do not select the standard Pico or Pico W).
   - **IP/Bluetooth Stack:** Pico W Bluetooth (Crucial for the Bluetooth HID library).

### **Flashing via BOOTSEL**

Hold down the small white **BOOTSEL** button on the Pico while plugging the USB cable into your computer. The Pico will mount as a mass storage drive (RP2350). Simply click **Upload** in the Arduino IDE; the software will automatically convert the code into a .uf2 file and flash the board.

### **Pairing with Windows**

1. In Windows, go to **Settings > Bluetooth & devices**.
2. Click on **Add device > Bluetooth**.
3. Select **Pico** from the list and click **Connect**. As soon as the status changes to "Connected", your macro pad is fully operational!

## **📦 Source Code & Building Your Own**

The complete, ready-to-use source code (.ino) featuring all registers, debounce algorithms, and macro settings is neatly documented and available on GitHub:

👉 \[View the project and download the code on GitHub\](Insert your repository link here)

Do you have questions about the wiring or want to 3D print a custom enclosure? Feel free to leave a comment or open an Issue on GitHub!
