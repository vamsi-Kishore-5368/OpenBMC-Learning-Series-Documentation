# OpenBMC Learning Series — Day 9

## Deep Dive into Linux Device Tree

### Understanding How Linux Knows What Hardware Exists

Welcome to **Day 9** of the OpenBMC Learning Series.

In Day 8, we learned how:

> **OpenBMC → Linux Kernel → Subsystems / Drivers → Hardware**

Today, we answer the next important question:

> **How does Linux know what hardware is present on an embedded BMC platform?**

The answer, on many embedded platforms, is:

> **Device Tree**

---

## 1. What Is Device Tree?

A **Device Tree** is a data structure used to describe hardware to the operating system.

It can describe things such as:

- I²C controllers and devices
- GPIO controllers
- SPI devices
- UARTs
- Ethernet controllers
- Interrupts
- Memory regions
- Clocks
- Other platform-specific hardware

The important point is:

> **Device Tree describes the hardware. It does not itself control the hardware.**

A simplified view is:

```text
Device Tree
     ↓
Describes Hardware
     ↓
Linux Kernel
     ↓
Driver / Subsystem
     ↓
Hardware
```

---

## 2. Why Is Device Tree Needed?

An embedded board can contain many peripherals.

```text
BMC SoC
 ├── I²C Controller
 │    ├── Temperature Sensor
 │    └── EEPROM
 │
 ├── GPIO Controller
 │    ├── Power Control
 │    └── Status Signals
 │
 ├── SPI Controller
 │    └── SPI Device
 │
 └── UART
      └── Debug Console
```

The Linux kernel needs a way to know how these devices are connected and what resources they use.

Device Tree provides that hardware description.

---

## 3. DTS vs DTSI

When working with Device Tree, you will commonly encounter:

### `.dts`

**Device Tree Source**

Usually contains a board or platform-specific hardware description.

### `.dtsi`

**Device Tree Source Include**

Used to share common hardware descriptions between multiple Device Trees.

A simplified relationship:

```text
Common SoC Description
        │
        ↓
      .dtsi
        │
        ↓
Board-specific Description
        │
        ↓
      .dts
        │
        ↓
   Device Tree Blob
       (.dtb)
```

The `.dtb` is the compiled binary representation passed to the Linux kernel.

---

## 4. Device Tree Is Organized as Nodes

Device Tree is organized into a hierarchy of **nodes**.

Conceptually:

```text
/
├── soc
│   ├── i2c-controller
│   │   └── temperature-sensor
│   │
│   ├── gpio-controller
│   │
│   └── uart
│
└── memory
```

A node represents a hardware component or logical hardware entity.

Nodes contain **properties** that describe that component.

---

## 5. Important Device Tree Properties

Some properties appear frequently when working with embedded Linux.

### `compatible`

Identifies the device or hardware implementation so that the kernel can match it with an appropriate driver.

Conceptually:

```text
compatible
     ↓
Driver Matching
     ↓
Driver
```

### `reg`

Describes the address/resource information associated with a device. The exact interpretation depends on the bus and the device-tree binding.

For example, an I²C device may use:

```text
reg = <0x48>;
```

### `status`

Controls whether a device is enabled.

```text
status = "okay";
```

or:

```text
status = "disabled";
```

### `interrupts`

Describes interrupt information for hardware that can generate interrupts.

### `gpios`

Describes GPIO connections used by a device, such as reset, power-enable or presence signals. The exact syntax depends on the device-tree binding.

---

## 6. The Most Important Property: `compatible`

One of the most important concepts to understand is **driver matching**.

```text
Device Tree
     │
     │ compatible
     ↓
Linux Device
     │
     ↓
Driver Matching
     │
     ↓
Driver
```

A driver typically contains a table of supported `compatible` strings.

When Linux processes the Device Tree, it can use this information to associate a device with the appropriate driver.

This is one of the key connections between:

> **Device Tree → Linux Driver**

---

## 7. A Simple I²C Example

Consider a temperature sensor connected to an I²C controller.

A simplified, illustrative Device Tree representation might look like:

```dts
&i2c1 {
    status = "okay";

    temperature_sensor@48 {
        compatible = "vendor,temp-sensor";
        reg = <0x48>;
    };
};
```

### What does this mean?

```text
&i2c1
```

→ We are describing an I²C controller.

```text
status = "okay";
```

→ Enable the controller.

```text
temperature_sensor@48
```

→ A child device exists at I²C address `0x48`.

```text
compatible = "vendor,temp-sensor";
```

→ Identifies the device for driver matching.

```text
reg = <0x48>;
```

→ Specifies the device's address on the I²C bus.

> **Important:** This is a simplified teaching example. Real Device Tree nodes must follow the binding defined for the specific hardware and kernel driver.

---

## 8. What Happens During Boot?

A simplified sequence is:

```text
BMC Boot
   ↓
Bootloader
   ↓
Linux Kernel + DTB
   ↓
Kernel parses Device Tree
   ↓
Devices are described/created
   ↓
Drivers are matched
   ↓
Drivers initialize hardware
   ↓
Userspace / OpenBMC Services
```

The exact boot implementation can vary between platforms.

OpenBMC documentation describes the bootloader passing the Device Tree to the kernel, with the kernel using information from the Device Tree during initialization.

---

## 9. Device Tree → Driver → Hardware

This is the concept we want to remember from Day 9.

```text
             Device Tree
                  ↓
          Hardware Description
                  ↓
             Linux Kernel
                  ↓
           Driver Matching
                  ↓
               Driver
                  ↓
              Hardware
```

So:

> **Device Tree tells Linux what the hardware looks like.**

> **The driver provides the software support needed to operate/support that hardware.**

---

## 10. Device Tree and OpenBMC

Device Tree becomes particularly important when bringing up a new BMC platform.

An OpenBMC system can have machine-specific Device Tree information describing things such as:

- GPIOs
- I²C buses and devices
- UARTs
- Ethernet
- Other platform peripherals

OpenBMC's developer documentation gives examples of adding machine-specific Device Tree descriptions for GPIOs, I²C devices, UARTs and other hardware.

The OpenBMC machine configuration also selects the Device Tree Blob to be built for the target platform.

For example, an OpenBMC machine configuration can specify:

```text
KERNEL_DEVICETREE = "aspeed-ast2600-evb.dtb"
```

The `.dtb` is generated from the corresponding Device Tree source as part of the kernel build.

---

## 11. Device Tree and Sensors

Device Tree can become directly relevant to OpenBMC sensor handling.

For example:

```text
I²C Controller
      ↓
Temperature Sensor
      ↓
Device Tree
      ↓
Linux Driver / hwmon
      ↓
OpenBMC Sensor Component
      ↓
D-Bus
      ↓
Redfish
```

OpenBMC's sensor architecture documentation shows how hardware-monitoring sensor configuration can correspond to paths derived from the Linux Device Tree.

This is where the concepts from several previous days begin to connect.

---

## 12. Device Tree Is Not the Driver

This distinction is extremely important.

### Device Tree

Describes:

> **What hardware exists and how it is connected.**

### Driver

Provides:

> **The software logic required to operate/support that hardware.**

So:

```text
Device Tree
     ↓
Description
```

while:

```text
Driver
     ↓
Hardware Support
```

Together:

```text
Device Tree + Driver
          ↓
      Linux Kernel
          ↓
       Hardware
```

---

## 13. Device Tree vs Hardware Schematic

They describe the system from different perspectives.

### Hardware Schematic

Shows the actual electrical design:

- Connections
- Signals
- Power
- Components
- Pull-ups
- Clocks
- Physical wiring

### Device Tree

Provides the operating system with the relevant hardware description needed for software to use the platform.

For example:

```text
Hardware Schematic
        ↓
Understand physical connection
        ↓
Create correct Device Tree description
        ↓
Linux Kernel
        ↓
Driver
```

The Device Tree should reflect the actual hardware design and the relevant Linux bindings.

---

## 14. How Device Tree Helps During Bring-Up

Suppose a new BMC board contains:

```text
I²C Bus
 ├── Temperature Sensor
 ├── EEPROM
 └── Voltage Monitor
```

During board bring-up, we may need to describe those devices correctly in Device Tree.

If the Device Tree is incorrect, Linux may not create the expected devices or bind the expected drivers.

This makes Device Tree debugging an important part of embedded Linux and OpenBMC platform bring-up.

---

## 15. A Practical Mental Model

When debugging a BMC hardware problem, think in this order:

```text
Is the hardware actually connected?
             ↓
Is it described correctly in Device Tree?
             ↓
Does Linux create the device?
             ↓
Does the correct driver bind?
             ↓
Does the driver communicate with hardware?
             ↓
Does OpenBMC expose/use the device?
```

This is a useful troubleshooting mindset.

---

## 16. Day 9 Mental Model

```text
┌──────────────────────────────┐
│       OpenBMC Services       │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│        Linux Kernel          │
│                              │
│   Device Tree + Subsystems   │
│             ↓                │
│          Drivers             │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│         BMC Hardware         │
│   I²C / GPIO / SPI / UART    │
│   Sensors / EEPROM / etc.    │
└──────────────────────────────┘
```

The key relationship is:

> **Device Tree describes the hardware → Linux matches drivers → Drivers communicate with the hardware.**

---

## 17. Key Takeaways

### ✅ 1.
Device Tree is a hardware description used by Linux on many embedded platforms.

### ✅ 2.
`.dts` files commonly describe board/platform-specific hardware.

### ✅ 3.
`.dtsi` files are commonly used for reusable/common Device Tree descriptions.

### ✅ 4.
Device Tree is compiled into a `.dtb`.

### ✅ 5.
Properties such as `compatible`, `reg`, `status`, `interrupts` and `gpios` are important concepts.

### ✅ 6.
`compatible` plays an important role in matching devices with drivers.

### ✅ 7.
Device Tree describes hardware; it does not replace the driver.

### ✅ 8.
OpenBMC uses Linux, so Device Tree becomes an important part of BMC platform bring-up.

### ✅ 9.
A simplified relationship is:

> **Device Tree → Kernel → Driver → Hardware**

### ✅ 10.
When debugging embedded hardware, Device Tree is one of the important layers to inspect.


## References

- OpenBMC Kernel Development — https://github.com/openbmc/docs/blob/master/kernel-development.md
- OpenBMC — Adding a New System — https://github.com/openbmc/docs/blob/master/development/add-new-system.md
- OpenBMC — Sensor Architecture — https://github.com/openbmc/docs/blob/master/architecture/sensor-architecture.md
- OpenBMC — Yocto Development — https://github.com/openbmc/docs/blob/master/yocto-development.md
- Linux Kernel Documentation — https://docs.kernel.org/
- Devicetree Specification — https://www.devicetree.org/specifications/
