# OpenBMC Learning Series --- Day 29

# Linux Kernel in OpenBMC --- How the BMC Runs Linux

> **Focus:** Linux Kernel, kernel configuration, drivers, Device Tree
> relationship, Yocto/OpenBMC integration, boot flow, and debugging.
> **Prerequisite:** Day 28 --- Yocto & BitBake

------------------------------------------------------------------------

## 1. Introduction

A BMC is a dedicated computer inside a server platform. It has its own
processor/SoC, RAM, flash, networking, GPIO, I²C/SMBus, SPI, UART,
watchdog and other platform-management hardware.

OpenBMC uses the Linux kernel as the operating-system kernel that
manages this hardware.

``` text
+------------------------------------------------------+
|                  Management Clients                  |
|       Redfish / IPMI / SSH / Web / Automation       |
+------------------------------------------------------+
                         |
                         v
+------------------------------------------------------+
|                  OpenBMC Services                    |
|  bmcweb | phosphor services | Entity Manager | etc.|
+------------------------------------------------------+
                         |
                         v
+------------------------------------------------------+
|                       D-Bus                          |
+------------------------------------------------------+
                         |
                         v
+------------------------------------------------------+
|                 Linux Kernel                         |
| Drivers | Scheduler | Memory | Networking | I/O     |
+------------------------------------------------------+
                         |
                         v
+------------------------------------------------------+
|                  BMC Hardware                        |
| CPU | I2C | GPIO | SPI | UART | Ethernet | Sensors |
+------------------------------------------------------+
```

The kernel is therefore the critical boundary between OpenBMC userspace
software and physical BMC hardware.

## 2. What Is the Linux Kernel?

The Linux kernel is the privileged core of the Linux operating system.

It manages:

-   Process scheduling
-   Memory
-   Interrupts
-   Device drivers
-   Networking
-   Filesystems
-   Timers
-   Hardware access
-   Kernel logging
-   Security mechanisms

For BMC work, drivers and hardware subsystems are especially important.

``` text
Linux Kernel
     |
     +-- Drivers
     +-- Memory
     +-- Networking
     +-- I/O
     +-- Scheduler
     |
     +-- I2C
     +-- GPIO
     +-- SPI
     +-- UART
     +-- HWMON
     +-- Watchdog
```

## 3. Linux Kernel vs OpenBMC

These are not the same thing.

### Linux Kernel

Provides:

``` text
Hardware abstraction
Drivers
Scheduling
Memory management
Networking
Filesystem support
Device management
```

### OpenBMC

Provides a complete BMC software stack around Linux:

``` text
OpenBMC
  |
  +-- Linux kernel
  +-- U-Boot
  +-- systemd
  +-- D-Bus
  +-- Sensor services
  +-- Entity Manager
  +-- phosphor components
  +-- bmcweb / Redfish
  +-- IPMI
  +-- MCTP / PLDM
  +-- Other userspace software
```

So:

``` text
Linux Kernel = operating-system core

OpenBMC = complete BMC software/firmware stack
           built around Linux
```

## 4. Why Does a BMC Need Linux?

A modern BMC needs to perform many tasks concurrently:

-   Monitor sensors
-   Communicate over Ethernet
-   Provide Redfish
-   Support IPMI
-   Manage host power
-   Record events
-   Handle storage
-   Manage hardware buses
-   Run D-Bus services
-   Provide SSH
-   Run firmware-update mechanisms
-   Handle watchdogs
-   Manage processes and logs

Linux provides the operating-system infrastructure for these components.

A typical request path can be:

``` text
Redfish request
      |
      v
bmcweb
      |
      v
D-Bus
      |
      v
OpenBMC service
      |
      v
Linux kernel driver
      |
      v
Physical hardware
```

## 5. Where the Kernel Fits

``` text
Management Clients
        |
        v
OpenBMC Applications
        |
        v
D-Bus / system services
        |
        v
Linux userspace interfaces
        |
        v
Linux Kernel
        |
        v
SoC peripherals
        |
        v
Physical BMC hardware
```

The kernel is the operating-system and hardware-control layer underneath
OpenBMC services.

## 6. Complete BMC Boot Flow

``` text
Power Applied
     |
     v
Boot ROM
     |
     v
U-Boot / Bootloader
     |
     +-- Initialize DRAM
     +-- Select boot image
     +-- Load kernel
     +-- Load Device Tree
     +-- Load initramfs when used
     |
     v
Linux Kernel
     |
     +-- CPU initialization
     +-- Memory initialization
     +-- Interrupt subsystem
     +-- Driver initialization
     +-- Device discovery
     +-- Network initialization
     +-- Root filesystem
     |
     v
systemd / init
     |
     v
OpenBMC services
     |
     +-- D-Bus
     +-- Entity Manager
     +-- Sensors
     +-- bmcweb
     +-- IPMI
     +-- MCTP / PLDM
     |
     v
Running BMC
```

OpenBMC documentation describes U-Boot as the bootloader and explains
that it loads the kernel, initrd and Device Tree before transferring
control to the kernel.

## 7. Kernel Initialization

A simplified initialization sequence is:

``` text
Kernel Entry
     |
Architecture Initialization
     |
Memory Initialization
     |
Interrupt Initialization
     |
Scheduler Initialization
     |
Driver / Subsystem Initialization
     |
Device Discovery
     |
Root Filesystem
     |
init / systemd
```

The exact sequence depends on architecture, configuration and whether
drivers are built in or modular.

## 8. Linux Kernel Source Tree

A simplified kernel source tree:

``` text
linux/
|
+-- arch/
|    +-- arm/
|    +-- arm64/
|
+-- drivers/
|    +-- i2c/
|    +-- gpio/
|    +-- hwmon/
|    +-- net/
|    +-- spi/
|    +-- tty/
|    +-- watchdog/
|
+-- include/
+-- kernel/
+-- mm/
+-- net/
+-- fs/
+-- block/
+-- init/
+-- scripts/
+-- Documentation/
+-- Makefile
```

For BMC development, `arch/` and `drivers/` are especially important.

## 9. OpenBMC Linux Kernel

OpenBMC maintains a kernel tree for the project.

The OpenBMC kernel-development guidance follows an **upstream-first**
philosophy:

``` text
New driver / feature
        |
        v
Upstream Linux when practical
        |
        v
OpenBMC consumes upstream support
```

OpenBMC can carry patches when platform bring-up or other practical
constraints require them, but long-lived downstream patches increase
maintenance cost.

## 10. Kernel Configuration

Linux is highly configurable.

The resulting kernel configuration is commonly represented by:

``` text
.config
```

Source configuration can be maintained as a `defconfig`.

Example options:

``` text
CONFIG_I2C=y
CONFIG_HWMON=y
CONFIG_NET=y
CONFIG_WATCHDOG=y
```

The exact configuration depends on the SoC, board, peripherals and
OpenBMC platform.

## 11. Built-In Drivers vs Modules

A driver can be built into the kernel:

``` text
CONFIG_DRIVER=y
```

or as a module:

``` text
CONFIG_DRIVER=m
```

If it is not enabled, the feature is unavailable.

A feature needed very early in boot may need to be built into the kernel
rather than loaded later.

## 12. Device Drivers

A device driver is software that allows the kernel to communicate with
hardware.

``` text
Hardware             Linux subsystem / driver
------------------------------------------------
I2C controller       I2C
GPIO controller      GPIO
Ethernet MAC         Network driver
SPI controller       SPI
UART                 Serial driver
Temperature sensor   HWMON / sensor driver
Watchdog             Watchdog subsystem
EEPROM               EEPROM subsystem
```

The general path is:

``` text
Application
    |
Kernel interface
    |
Driver
    |
Hardware
```

## 13. Why Drivers Matter in a BMC

Consider a temperature sensor connected over I²C:

``` text
Temperature Sensor
        |
      I2C bus
        |
        v
Linux I2C controller driver
        |
        v
Sensor driver / HWMON
        |
        v
OpenBMC userspace
```

Higher-level OpenBMC software can work through standard Linux interfaces
instead of directly manipulating SoC registers.

## 14. Common BMC Hardware Interfaces

### I²C / SMBus

Common uses:

-   Temperature sensors
-   Voltage monitors
-   Fan controllers
-   EEPROMs
-   Power-management devices

``` text
OpenBMC
  |
Linux I2C subsystem
  |
I2C controller driver
  |
I2C bus
  |
Device
```

### GPIO

Common uses:

-   LEDs
-   Reset
-   Presence
-   Power-good
-   Host status
-   Interrupts

``` text
OpenBMC service
      |
Linux GPIO subsystem
      |
GPIO controller driver
      |
GPIO pin
```

### SPI

Common uses:

-   Flash
-   Platform-specific peripherals

### UART

Common uses:

-   Serial console
-   Debugging
-   Host communication

### Ethernet

``` text
Application
   |
Socket API
   |
Linux networking stack
   |
Ethernet MAC driver
   |
MAC / PHY
   |
Network
```

### Watchdog

``` text
OpenBMC service
      |
Watchdog API
      |
Linux watchdog subsystem
      |
Watchdog driver
      |
Hardware watchdog
```

## 15. How Linux Represents Hardware

Important Linux interfaces include:

``` text
/sys
/dev
/proc
```

They expose different views of devices, kernel state and processes.

### `/sys`

``` bash
ls /sys
ls /sys/class
ls /sys/bus
ls /sys/devices
```

`sysfs` exposes the kernel device model and related attributes.

### `/dev`

Examples:

``` text
/dev/i2c-0
/dev/ttyS4
/dev/watchdog
```

The exact devices depend on the platform.

### `/proc`

Examples:

``` bash
cat /proc/cpuinfo
cat /proc/meminfo
cat /proc/interrupts
cat /proc/cmdline
```

If kernel configuration is exposed:

``` bash
zcat /proc/config.gz
```

## 16. Kernel → Userspace Boundary

``` text
+-------------------------------+
| OpenBMC Userspace             |
| bmcweb | sensors | services   |
+-------------------------------+
               |
       Linux interfaces
               |
+-------------------------------+
| Linux Kernel                  |
| drivers | networking | I/O    |
+-------------------------------+
               |
+-------------------------------+
| Hardware                      |
+-------------------------------+
```

Userspace normally accesses hardware through defined kernel interfaces
rather than arbitrary register access.

## 17. Kernel → D-Bus → OpenBMC

D-Bus is **not part of the Linux kernel**.

It is a userspace IPC mechanism.

``` text
Linux Kernel
     |
     | kernel interfaces
     v
OpenBMC userspace service
     |
     | D-Bus
     v
Other OpenBMC services
```

This distinction is fundamental:

``` text
Kernel = low-level OS + hardware

D-Bus = userspace communication between services
```

## 18. Example --- Temperature Sensor

``` text
Temperature Sensor
       |
       | I2C
       v
BMC I2C Controller
       |
       v
Linux I2C Driver
       |
       v
Sensor Driver / HWMON
       |
       v
OpenBMC Userspace
       |
       v
D-Bus
       |
       v
bmcweb
       |
       v
Redfish
```

Redfish does not directly read the physical sensor.

The request travels through the software layers.

## 19. Example --- GPIO

``` text
Power Hardware
     |
PWR_GOOD signal
     |
     v
BMC GPIO pin
     |
     v
GPIO controller
     |
     v
Linux GPIO subsystem
     |
     v
OpenBMC userspace
     |
     v
D-Bus
     |
     v
Power-management service
```

## 20. Example --- EEPROM

``` text
EEPROM
  |
 I2C
  |
I2C Controller
  |
Linux I2C subsystem
  |
EEPROM interface
  |
Userspace
  |
OpenBMC service
```

## 21. Example --- Network Interface

``` text
Redfish / SSH / IPMI
        |
        v
Socket API
        |
        v
Linux Networking Stack
        |
        v
Ethernet MAC Driver
        |
        v
MAC / PHY
        |
        v
Ethernet
```

Linux provides the networking stack used by OpenBMC applications.

## 22. Linux Kernel and Device Tree

For embedded platforms, the Linux kernel often uses a **Device Tree** to
describe hardware that cannot simply be discovered automatically.

``` text
Device Tree Source (.dts)
          |
          v
Device Tree Blob (.dtb)
          |
          v
Bootloader
          |
          v
Linux Kernel
          |
          v
Device / Driver Matching
```

A Device Tree can describe:

-   CPUs
-   Memory
-   UARTs
-   I²C controllers
-   GPIO controllers
-   SPI
-   Ethernet
-   Interrupts
-   Sensors
-   Board-specific connections

Day 30 will go deeper into this.

## 23. Device Tree and Drivers Work Together

A Device Tree does not implement a driver.

``` text
Device Tree
     |
     | describes hardware
     v
Kernel Device Model
     |
     | matching
     v
Driver
     |
     v
Hardware
```

In simple terms:

-   Device Tree: **What hardware exists and how is it connected?**
-   Driver: **How does Linux operate it?**

## 24. Kernel Configuration + Device Tree + Driver

These three concepts are different.

### Kernel configuration

> Is this kernel functionality enabled?

### Device Tree

> What hardware exists on this platform, and how is it
> connected/configured?

### Driver

> How does Linux operate this hardware?

Together:

``` text
Kernel Configuration
        +
Device Tree
        +
Driver
        |
        v
Working Hardware Support
```

## 25. OpenBMC Machine Configuration

OpenBMC uses Yocto machine configuration to select platform-specific
build settings.

These can include:

-   Kernel Device Tree
-   U-Boot configuration
-   SoC family
-   Serial console
-   Flash size
-   Platform-specific options

A current OpenBMC AST2600 EVB machine file, for example, contains:

``` bitbake
KERNEL_DEVICETREE = "aspeed/aspeed-ast2600-evb.dtb"
```

This connects the machine build to the appropriate DTB.

## 26. Kernel Recipe in OpenBMC

The kernel is built through Yocto/BitBake.

Conceptually:

``` text
Machine
  |
  v
Kernel recipe
  |
  +-- Source
  +-- Configuration
  +-- Patches
  +-- Device Tree
  |
  v
BitBake
  |
  v
Linux kernel build
```

The recipe name depends on the OpenBMC branch and platform. Current
OpenBMC documentation notes that Aspeed-based builds use `linux-aspeed`
in current revisions.

## 27. Kernel `.bbappend`

A platform layer can customize an existing kernel recipe using a
`.bbappend`.

Example:

``` text
meta-myplatform/
|
+-- recipes-kernel/
    |
    +-- linux/
        |
        +-- linux-aspeed_%.bbappend
        |
        +-- files/
            |
            +-- myboard.cfg
            +-- patches/
```

This is preferable to copying the entire kernel recipe when only
platform-specific changes are required.

## 28. Kernel Configuration Fragments

A platform can add configuration fragments.

Conceptually:

``` text
Base kernel configuration
          +
Platform configuration
          +
Custom configuration fragment
          |
          v
Final .config
```

Example:

``` text
CONFIG_I2C=y
CONFIG_HWMON=y
CONFIG_WATCHDOG=y
```

The exact integration mechanism depends on the OpenBMC/Yocto branch.

## 29. Kernel Patches

If required support is not available in the selected kernel version,
OpenBMC may carry a patch.

``` text
Upstream Linux
      |
      +-- existing functionality
      |
      v
OpenBMC kernel
      |
      +-- platform patch when needed
      |
      v
Build
```

The long-term preferred direction is:

``` text
Local change
    ↓
Upstream Linux
    ↓
OpenBMC consumes upstream support
```

## 30. Kernel Development with `devtool`

OpenBMC documentation supports Yocto `devtool` for kernel development.

Typical workflow:

``` bash
. setup <machine>
devtool modify linux-aspeed
```

This creates a development workspace for the kernel recipe.

After changes:

``` bash
devtool build linux-aspeed
```

Always verify the recipe name for the OpenBMC revision being used.

## 31. Kernel Configuration with `menuconfig`

A documented workflow is:

``` bash
bitbake linux-aspeed -c menuconfig
```

After changing options:

``` bash
bitbake linux-aspeed -c savedefconfig
```

This can save the reduced configuration for reuse.

## 32. Inspecting Resolved Yocto Variables

Yocto variables can be affected by multiple layers and overrides.

Useful commands:

``` bash
bitbake -e linux-aspeed
```

and:

``` bash
bitbake-getvar MACHINE -r obmc-phosphor-image
```

These help determine:

-   Kernel source/revision
-   Applied configuration
-   Patches
-   Selected Device Tree
-   Active overrides
-   Machine settings
-   Toolchain information

## 33. Kernel Build Flow Inside BitBake

``` text
Kernel Recipe
     |
     v
do_fetch
     |
     v
do_unpack
     |
     v
do_patch
     |
     v
Configuration
     |
     v
do_compile
     |
     +-- Kernel
     +-- Device Tree
     +-- Modules
     |
     v
do_install
     |
     v
Packaging / Deployment
     |
     v
OpenBMC image creation
```

This directly connects to Day 28.

## 34. From Kernel Build to BMC Image

``` text
OpenBMC Layers
      |
      v
Machine Configuration
      |
      +--------------------+
      |                    |
      v                    v
Linux Kernel            U-Boot
      |                    |
      +---------+----------+
                |
                v
         Root Filesystem
                |
                v
        OpenBMC Services
                |
                v
         Image Generation
                |
                v
        BMC Firmware Image
```

The final artifacts depend on the machine and current build
configuration.

## 35. FIT Images

OpenBMC's kernel-development documentation describes FIT-based images.

A FIT image can package related boot components:

``` text
FIT Image
   |
   +-- Kernel
   +-- Device Tree
   +-- Initramfs
```

The exact artifact name and format vary by machine and branch.

## 36. Kernel + DTB + Initramfs

``` text
             Boot Image
                 |
      +----------+----------+
      |          |          |
      v          v          v
    Kernel      DTB      Initramfs
      |          |          |
      +----------+----------+
                 |
                 v
             Linux Boot
```

Kernel = Linux operating-system code.

DTB = hardware description.

Initramfs = initial filesystem when used by the boot configuration.

## 37. Kernel Logs

Useful commands:

``` bash
dmesg
dmesg | less
dmesg | grep -i i2c
dmesg | grep -i gpio
dmesg | grep -i eth
dmesg | grep -i watchdog
```

These can reveal:

-   Driver initialization
-   Device probing
-   Hardware errors
-   Interrupt issues
-   Network initialization
-   I²C failures
-   Missing devices
-   Boot problems

## 38. `journalctl` and Kernel Messages

On systemd-based systems:

``` bash
journalctl -k
```

Useful examples:

``` bash
journalctl -k -b
journalctl -k | grep -i i2c
journalctl -k | grep -i error
```

Conceptually:

``` text
dmesg
   |
Kernel message buffer

journalctl -k
   |
Kernel messages collected by systemd journal
```

## 39. Checking Kernel Modules

``` bash
lsmod
modinfo <module>
modprobe <module>
```

Whether a driver is modular depends on kernel configuration.

## 40. Checking Kernel Version

``` bash
uname -a
uname -r
```

When debugging an OpenBMC platform, identify the actual kernel version
running on the BMC rather than assuming it from the source tree.

## 41. Checking Hardware from Userspace

``` bash
ls /sys
ls /sys/class
ls /sys/bus
ls /sys/devices
ls /dev
```

I²C:

``` bash
ls /dev/i2c*
```

Hardware monitoring:

``` bash
ls /sys/class/hwmon
```

Serial:

``` bash
ls /dev/tty*
```

Network:

``` bash
ip link
```

## 42. Debugging a Missing I²C Device

``` text
Sensor missing
     |
     v
Is I2C controller enabled?
     |
     +-- No --> Kernel config / Device Tree
     |
     v
Is I2C bus present?
     |
     +-- No --> Driver / Device Tree / hardware
     |
     v
Is device described?
     |
     +-- No --> Device Tree
     |
     v
Does driver probe?
     |
     +-- No --> Driver / compatible / address / power
     |
     v
Does userspace see it?
     |
     +-- No --> Userspace integration
     |
     v
D-Bus object exists?
     |
     +-- No --> OpenBMC service/configuration
     |
     v
Redfish resource exists?
```

## 43. Debugging a Missing GPIO

``` text
GPIO not working
      |
      v
GPIO controller enabled?
      |
      v
GPIO described in Device Tree?
      |
      v
Pinmux correct?
      |
      v
Correct GPIO consumer?
      |
      v
Kernel errors?
      |
      v
Expected state visible in userspace?
```

## 44. Debugging a Network Interface

``` text
Ethernet missing
      |
      v
MAC driver loaded?
      |
      v
Device Tree correct?
      |
      v
PHY configured?
      |
      v
Interface exists?
      |
      v
Link state?
      |
      v
IP configuration?
      |
      v
Connectivity
```

Useful commands:

``` bash
ip link
ip addr
ip route
dmesg | grep -i eth
```

## 45. Kernel Debugging Mindset

Always identify the layer where the failure occurs:

``` text
Hardware
   ↓
Device Tree
   ↓
Kernel configuration
   ↓
Driver
   ↓
Kernel interface
   ↓
Userspace service
   ↓
D-Bus
   ↓
bmcweb / IPMI / PLDM
```

If the kernel cannot see the hardware, changing Redfish code will not
solve the hardware problem.

## 46. Kernel and OpenBMC Services

``` text
Linux Kernel
    |
    +--> hwmon
    +--> I2C
    +--> GPIO
    +--> networking
    +--> watchdog
    |
    v
OpenBMC userspace
    |
    +--> Sensor services
    +--> Entity Manager
    +--> Power services
    +--> Host-state services
    +--> bmcweb
```

## 47. Kernel and Sensor Architecture

``` text
Physical Sensor
      |
      v
Bus / Controller
      |
      v
Kernel Driver
      |
      v
Kernel Interface
      |
      v
OpenBMC Sensor Component
      |
      v
D-Bus
      |
      v
bmcweb
      |
      v
Redfish
```

The exact userspace component depends on the sensor type and OpenBMC
implementation.

## 48. Kernel and Power Management

Power management can involve:

-   GPIO
-   I²C
-   PMBus
-   ADC
-   Power controllers
-   Watchdog

Conceptually:

``` text
Power Hardware
      |
      v
Kernel drivers
      |
      v
OpenBMC power-management services
      |
      v
D-Bus
      |
      v
Redfish / IPMI / Host management
```

## 49. Kernel and Networking

``` text
Redfish client
       |
       v
Ethernet
       |
       v
NIC / MAC / PHY
       |
       v
Linux network driver
       |
       v
Linux networking stack
       |
       v
Socket
       |
       v
bmcweb
```

## 50. Kernel and Watchdog

``` text
OpenBMC service
       |
       v
Watchdog interface
       |
       v
Linux watchdog subsystem
       |
       v
Hardware watchdog
```

## 51. Kernel Build Artifacts

After an OpenBMC build, machine-specific deployment files are normally
under:

``` text
build/tmp/deploy/images/<machine>/
```

Depending on the platform, this can contain:

``` text
u-boot
kernel image
device tree
initramfs
FIT image
root filesystem
flash image
```

Exact filenames and formats differ between machines and revisions.

## 52. Kernel Development Workflow

``` text
Identify problem
       |
Determine hardware
       |
Check Device Tree
       |
Check kernel configuration
       |
Check driver
       |
Check kernel logs
       |
Make change
       |
Build
       |
Boot / test
       |
Inspect logs
       |
Validate userspace integration
```

For new platform work:

``` text
Hardware schematic
       |
       +-- GPIO
       +-- I2C
       +-- UART
       +-- Ethernet
       +-- SPI / Flash
       +-- Other peripherals
       |
       v
Device Tree
       |
       v
Kernel configuration
       |
       v
Drivers
       |
       v
Machine configuration
       |
       v
OpenBMC image
       |
       v
Boot + validation
```

## 53. Testing Kernel Changes

OpenBMC kernel-development documentation recommends boot testing on QEMU
platforms and on available physical hardware before submitting changes.

A useful progression is:

``` text
Build
  ↓
Validation
  ↓
QEMU boot test
  ↓
Physical BMC test
  ↓
Hardware-specific validation
```

QEMU can shorten development cycles because repeatedly flashing physical
BMC flash can be slow.

## 54. Network Boot Testing

OpenBMC documentation describes loading a FIT image through TFTP and
booting it from U-Boot.

Conceptually:

``` text
Kernel Build
     |
     v
FIT Image
     |
     v
TFTP Server
     |
     v
U-Boot
     |
     v
RAM
     |
     v
bootm
     |
     v
Linux
```

The exact memory addresses and commands must match the target platform.

## 55. New Hardware Bring-Up

A new BMC platform commonly requires:

``` text
Hardware schematic
       |
       +-- GPIO information
       +-- I2C buses/devices
       +-- UART
       +-- Ethernet
       +-- SPI / Flash
       +-- other peripherals
       |
       v
Device Tree
       |
       v
Kernel configuration
       |
       v
Drivers
       |
       v
Machine configuration
       |
       v
OpenBMC image
       |
       v
Boot + validate
```

OpenBMC's new-system documentation specifically describes adding a
machine Device Tree for GPIOs, I²C buses/devices and other peripherals,
then selecting the appropriate DTB with `KERNEL_DEVICETREE`.

## 56. Day 28 → Day 29 Connection

Day 28:

``` text
Yocto
  ↓
BitBake
  ↓
Recipes
  ↓
Tasks
  ↓
Packages
  ↓
Root filesystem
  ↓
BMC image
```

Day 29 opens one of those major components:

``` text
BitBake
   |
   +----------------------+
   |                      |
   v                      v
Linux Kernel            U-Boot
   |
   +-- Drivers
   +-- Device Tree
   +-- Kernel config
   |
   v
BMC Hardware Support
```

Therefore:

``` text
DAY 28
How OpenBMC is BUILT

        ↓

DAY 29
How Linux CONTROLS the hardware

        ↓

DAY 30
How Device Tree DESCRIBES the hardware
```

## 57. Complete Day 29 Architecture

``` text
                         MANAGEMENT
                             |
              +--------------+--------------+
              |              |              |
           Redfish         IPMI         SSH/Other
              |              |              |
              +--------------+--------------+
                             |
                             v
                     OpenBMC Userspace
                             |
              +--------------+--------------+
              |              |              |
           bmcweb       Sensor Services   Other
              |              |              |
              +--------------+--------------+
                             |
                           D-Bus
                             |
                             v
                    Linux Userspace APIs
                             |
                             v
                     Linux Kernel
                             |
       +----------+----------+----------+----------+
       |          |          |          |          |
      I2C       GPIO        SPI       UART       NET
       |          |          |          |          |
       +----------+----------+----------+----------+
                             |
                             v
                      BMC SoC / Hardware
                             |
       +----------+----------+----------+----------+
       |          |          |          |          |
    Sensors    EEPROM      Fans       Power      Host
```

## 58. Important Commands --- Quick Reference

### Kernel

``` bash
uname -a
uname -r
dmesg
journalctl -k
```

### Hardware

``` bash
ls /sys
ls /sys/class
ls /sys/bus
ls /sys/devices
ls /dev
ls /dev/i2c*
ls /dev/tty*
ip link
```

### Modules

``` bash
lsmod
modinfo <module>
modprobe <module>
```

### Yocto / OpenBMC

``` bash
bitbake -e linux-aspeed
bitbake-getvar MACHINE -r obmc-phosphor-image
bitbake linux-aspeed -c menuconfig
bitbake linux-aspeed -c savedefconfig
devtool modify linux-aspeed
```

Use the recipe name appropriate for the OpenBMC revision being built.

## 59. Common Mistakes

### Mistake 1 --- Treating OpenBMC and Linux as the same thing

OpenBMC is the complete BMC software stack; Linux is its
operating-system kernel.

### Mistake 2 --- Treating Device Tree as a driver

Device Tree describes hardware. The driver implements the software that
operates it.

### Mistake 3 --- Treating D-Bus as a kernel mechanism

D-Bus is userspace IPC.

### Mistake 4 --- Debugging Redfish first

If the kernel cannot see the hardware, changing bmcweb will not fix the
hardware problem.

### Mistake 5 --- Changing kernel configuration without checking the final result

Multiple Yocto layers and overrides can affect the final `.config`.

### Mistake 6 --- Assuming identical image names on every platform

Artifacts and image formats vary by machine and OpenBMC revision.

### Mistake 7 --- Ignoring upstream Linux

Long-term maintenance benefits from upstreaming drivers and platform
support whenever practical.

## 60. Practical Mental Model

Remember these four questions:

### What hardware exists?

``` text
Device Tree / platform description
```

### Is the required functionality enabled?

``` text
Kernel configuration
```

### How does Linux communicate with it?

``` text
Kernel driver
```

### How does OpenBMC use it?

``` text
Userspace → D-Bus → OpenBMC services → Redfish/IPMI/etc.
```

Together:

``` text
Hardware
   ↓
Device Tree
   ↓
Kernel Configuration
   ↓
Driver
   ↓
Kernel Interface
   ↓
OpenBMC Userspace
   ↓
D-Bus
   ↓
Management Protocol
```

## 61. Final Summary

The Linux kernel is the critical software layer connecting OpenBMC
userspace with BMC hardware.

The core architecture is:

``` text
              OpenBMC
                 |
            Userspace
                 |
              D-Bus
                 |
          Linux Kernel
                 |
        +--------+--------+
        |        |        |
       I2C      GPIO     Network
        |        |        |
        +--------+--------+
                 |
            BMC Hardware
```

For new BMC platform bring-up:

``` text
Hardware
   ↓
Device Tree
   ↓
Kernel Configuration
   ↓
Drivers
   ↓
Linux Kernel
   ↓
OpenBMC Userspace
   ↓
D-Bus
   ↓
Redfish / IPMI / PLDM / MCTP
```

This establishes the foundation for understanding how OpenBMC interacts
with the physical server platform.

## 62. Official References

-   OpenBMC Kernel Development:
    https://github.com/openbmc/docs/blob/master/kernel-development.md
-   OpenBMC Development Cheatsheet:
    https://github.com/openbmc/docs/blob/master/cheatsheet.md
-   Adding a New OpenBMC System:
    https://github.com/openbmc/docs/blob/master/development/add-new-system.md
-   OpenBMC Development Environment:
    https://github.com/openbmc/docs/blob/master/development/dev-environment.md
-   OpenBMC Yocto Development:
    https://github.com/openbmc/docs/blob/master/yocto-development.md
-   OpenBMC Linux Kernel: https://github.com/openbmc/linux
-   OpenBMC Source: https://github.com/openbmc/openbmc

------------------------------------------------------------------------

# Day 29 Complete

``` text
DAY 28 → Yocto & BitBake
       ↓
DAY 29 → Linux Kernel
       ↓
DAY 30 → Device Tree
       ↓
DAY 31 → Linux Kernel Drivers
       ↓
DAY 32 → U-Boot & BMC Boot Flow
```
