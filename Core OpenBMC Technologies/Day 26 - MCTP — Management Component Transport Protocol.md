# OpenBMC Learning Series --- Day 26

# MCTP --- Management Component Transport Protocol

### From Host / BMC / Device Communication → MCTP → EID → Endpoint → PLDM

------------------------------------------------------------------------

## 1. Introduction

Day 25 covered traditional IPMI management:

``` text
Fault → SEL → phosphor-sel-logger → IPMI → ipmitool
```

Day 26 moves to the newer platform-management transport architecture:

``` text
Host / BMC / Device
        ↓
       MCTP
        ↓
   MCTP Endpoint
        ↓
       EID
        ↓
 Higher-Level Protocol
        ↓
       PLDM
```

**MCTP = Management Component Transport Protocol.**

The key idea is:

> **MCTP provides transport; higher-level protocols such as PLDM provide
> management semantics.**

OpenBMC's MCTP design was motivated partly by limitations of traditional
IPMI-based host/BMC communication and by the need for a reusable
transport layer across different hardware channels. citeturn0search1

------------------------------------------------------------------------

## 2. Why MCTP?

Traditional IPMI combines:

``` text
Transport
   +
Messaging / Management Model
```

MCTP separates them:

``` text
Physical Binding
       ↓
      MCTP
       ↓
Higher-Level Protocol
       ↓
Application / Management
```

This allows the same transport architecture to support different
hardware channels and higher-level protocols.

------------------------------------------------------------------------

## 3. MCTP vs PLDM

### MCTP

Handles transport-related functions such as:

-   endpoint addressing
-   packetization
-   reassembly
-   routing
-   forwarding
-   binding interaction

### PLDM

Provides higher-level platform-management messaging.

Therefore:

``` text
Application / Management
          ↓
         PLDM
          ↓
         MCTP
          ↓
Physical Transport Binding
```

OpenBMC's kernel design explicitly places higher-level protocols such as
PLDM above the kernel MCTP transport layer. citeturn0search0

------------------------------------------------------------------------

## 4. MCTP Stack

``` text
┌──────────────────────────────┐
│ Application / Management     │
├──────────────────────────────┤
│ PLDM / Other MCTP Protocol   │
├──────────────────────────────┤
│ MCTP Transport               │
├──────────────────────────────┤
│ MCTP Binding                 │
├──────────────────────────────┤
│ Physical Hardware Channel    │
└──────────────────────────────┘
```

Possible bindings include:

``` text
I²C / SMBus
PCIe
UART / Serial
LPC
```

The binding is separated from the common transport protocol.
citeturn0search1

------------------------------------------------------------------------

## 5. What Is an MCTP Endpoint?

An MCTP endpoint is a device/entity participating in an MCTP network.

Examples:

``` text
Host
BMC
PCIe device
Management controller
Power controller
Other platform device
```

Each endpoint is identified by an **Endpoint ID (EID)**.

------------------------------------------------------------------------

## 6. What Is an EID?

**EID = Endpoint ID**

Example:

``` text
Host   → EID 8
BMC    → EID 9
Device → EID 10
```

A request can therefore be:

``` text
Source EID      = 9
Destination EID = 10
```

An EID is not an IP address.

``` text
IP address ≠ MCTP EID
```

------------------------------------------------------------------------

## 7. MCTP Networks

An MCTP network defines an EID address space.

The kernel design models:

``` text
Interface / Link
        +
Network
        +
EID
        +
Routes
```

A network can contain multiple interfaces and EIDs. Different MCTP
networks can contain duplicate EIDs because EIDs are scoped to a
network. citeturn0search0

------------------------------------------------------------------------

## 8. Interface vs Network vs EID

### Interface / Link

Represents a physical MCTP transport binding.

### Network

Defines an MCTP EID address space.

### EID

Identifies an endpoint within that network.

``` text
Physical Interface
        ↓
      Network
        ↓
       EID
        ↓
    Endpoint
```

------------------------------------------------------------------------

## 9. Message vs Packet

A **message** is the higher-level data exchanged between endpoints.

A **packet** is the transport unit sent over the physical binding.

Large messages may be split:

``` text
Large Message
      ↓
Packet 1 + Packet 2 + Packet 3
      ↓
Physical Transport
      ↓
Reassembly
      ↓
Complete Message
```

OpenBMC's MCTP design explicitly distinguishes messages from packets and
describes packetization/reassembly around the physical MTU.
citeturn0search1

------------------------------------------------------------------------

## 10. MCTP Transport Responsibilities

MCTP can provide:

``` text
Addressing
Packetization
Reassembly
Routing
Forwarding
Tag handling
Binding interaction
```

The conceptual separation is:

``` text
MCTP
  ↓
"How do I deliver the message?"

PLDM
  ↓
"What does the message mean?"
```

------------------------------------------------------------------------

## 11. Physical Bindings

MCTP separates the common transport from the physical channel:

``` text
               MCTP
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
      I²C       PCIe      UART
       │         │         │
       ▼         ▼         ▼
    Hardware  Hardware  Hardware
```

This means higher layers do not need to be rewritten for every physical
transport.

------------------------------------------------------------------------

## 12. MCTP over I²C / SMBus

Conceptually:

``` text
BMC
 │
I²C Controller
 │
SMBus
 │
MCTP Endpoint
```

This is useful for management communication with devices attached to
management buses.

------------------------------------------------------------------------

## 13. MCTP over PCIe

Conceptually:

``` text
BMC / Host
    │
   PCIe
    │
MCTP Binding
    │
MCTP Endpoint
    │
Device
```

PCIe can provide a higher-bandwidth platform interconnect for management
communication.

------------------------------------------------------------------------

## 14. MCTP over UART / LPC

Other platform channels can carry MCTP depending on the platform
binding:

``` text
BMC
 │
UART / LPC
 │
MCTP
 │
Remote Endpoint
```

The exact hardware binding is platform-specific.

------------------------------------------------------------------------

## 15. Addressing

The Linux kernel MCTP socket API represents addressing using fields
including:

``` text
MCTP network
Destination EID
Message type
Tag
```

The destination address maps to the destination endpoint's EID.
citeturn0search0

------------------------------------------------------------------------

## 16. Routing and Forwarding

A BMC may connect multiple MCTP interfaces:

``` text
              BMC
          ┌─────┴─────┐
          ▼           ▼
        I²C Bus     PCIe Bus
          │           │
       Device A    Device B
```

MCTP can use routing/forwarding to reach remote EIDs.

The kernel design supports multiple interfaces in a network and
configurable forwarding/routing. citeturn0search0

------------------------------------------------------------------------

## 17. MCTP Control Protocol

MCTP includes a Control Protocol for transport-level management,
including functions related to:

``` text
Endpoint discovery
Endpoint information
EID management
Transport configuration
```

The kernel implementation uses a limited subset internally, while more
complex endpoint management can involve userspace control-protocol
handling. citeturn0search0

------------------------------------------------------------------------

## 18. Static vs Dynamic EIDs

A simple endpoint may use a static EID:

``` text
BMC    = 9
Device = 10
```

More complex environments can require dynamic EID allocation.

OpenBMC's kernel design notes that endpoints requiring dynamic
allocation or roles such as bus owners/bridges can use userspace MCTP
Control Protocol handling. citeturn0search0

------------------------------------------------------------------------

## 19. MCTP Tags

Tags help correlate request/response traffic.

Conceptually:

``` text
Request
Source EID = 9
Destination EID = 10
Tag = 3

       ↓

Response
Source EID = 10
Destination EID = 9
Tag = 3
```

The Linux MCTP socket design includes tag-owner behavior and tag
allocation for outgoing requests. citeturn0search0

------------------------------------------------------------------------

## 20. OpenBMC MCTP Implementations

OpenBMC documents two approaches.

### Userspace implementation

``` text
MCTP core library
       ↓
MCTP daemon
       ↓
Hardware bindings
       ↓
Applications
```

### Kernel implementation

``` text
Linux kernel MCTP
       ↓
AF_MCTP sockets
       ↓
Userspace applications
```

The current OpenBMC design recommends the **kernel implementation for
new designs**; the userspace implementation remains relevant to some
existing platforms. citeturn0search1

------------------------------------------------------------------------

## 21. Why Kernel MCTP?

The kernel implementation provides:

``` text
Standard Linux socket API
+
Kernel transport handling
+
Packetization
+
Routing
+
Physical bindings
```

Applications can use:

``` c
AF_MCTP
```

rather than implementing the complete transport stack themselves.
citeturn0search0

------------------------------------------------------------------------

## 22. AF_MCTP and sockaddr_mctp

A simplified socket concept is:

``` c
socket(AF_MCTP, SOCK_DGRAM, 0);
```

The MCTP socket address contains concepts such as:

``` text
Network
Destination EID
Message Type
Tag
```

This gives userspace applications a standard Linux-style communication
interface. citeturn0search0

------------------------------------------------------------------------

## 23. MCTP Message Type

The message type identifies the higher-level protocol.

Conceptually:

``` text
MCTP
  │
  ├── MCTP Control
  ├── PLDM
  └── Other defined/vendor protocols
```

The kernel socket API uses the message type as part of the MCTP
addressing model. citeturn0search0

------------------------------------------------------------------------

## 24. MCTP + PLDM

The relationship is:

``` text
              PLDM
                │
          PLDM Message
                │
                ▼
              MCTP
                │
        MCTP Packetization
                │
                ▼
       Physical Binding
                │
                ▼
             Device
```

MCTP transports the PLDM message; PLDM defines its meaning.

------------------------------------------------------------------------

## 25. MCTP + SPDM

MCTP is not limited to PLDM.

OpenBMC also documents SPDM-over-MCTP architectures:

``` text
SPDM
  ↓
MCTP
  ↓
PCIe / SMBus / Other Binding
  ↓
Security-Capable Device
```

This illustrates the value of a reusable transport layer.
citeturn0search4

------------------------------------------------------------------------

## 26. Userspace MCTP Architecture

The older architecture is conceptually:

``` text
Application
    │
Unix-domain socket
    │
MCTP Demux / Daemon
    │
libmctp
    │
Binding
    │
Hardware
```

The OpenBMC userspace design describes an MCTP core, binding
implementations and a demultiplexer daemon communicating with
applications over a Unix-domain socket. citeturn0search2

------------------------------------------------------------------------

## 27. libmctp

`libmctp` is a portable MCTP implementation.

It remains useful when:

-   kernel MCTP is unavailable,
-   an embedded device needs its own stack,
-   one application owns the MCTP stack,
-   custom hardware bindings are required.

For Linux systems with kernel MCTP support, the current project
documentation recommends using the kernel sockets interface instead.
citeturn0search5

------------------------------------------------------------------------

## 28. MCTP net_device

The kernel implementation represents MCTP physical interfaces using
Linux `net_device` infrastructure.

Conceptually:

``` text
Linux
 ├── Ethernet net_device
 ├── Wi-Fi net_device
 └── MCTP net_device
```

MCTP interfaces have their own semantics and are not ordinary Ethernet
interfaces.

------------------------------------------------------------------------

## 29. The `mctp` Utility

The kernel design defines a configuration utility:

``` bash
mctp
```

with subcommands including:

``` text
mctp link
mctp network
mctp address
mctp route
mctp stat
```

It is designed similarly to Linux `iproute2` tools. citeturn0search0

------------------------------------------------------------------------

## 30. `mctp link`

Examples documented by OpenBMC:

``` bash
mctp link set <link> up
mctp link set <link> down
mctp link set <link> network <network-id>
mctp link set <link> mtu <mtu>
mctp link set <link> bus-owner <hwaddr>
```

Exact support depends on the installed utility/kernel version.
citeturn0search0

------------------------------------------------------------------------

## 31. `mctp network`

``` bash
mctp network create <network-id>
mctp network set <network-id> forwarding on
mctp network set <network-id> default true
mctp network delete <network-id>
```

A network defines the EID address space and routing context.

------------------------------------------------------------------------

## 32. `mctp address`

Local EID management:

``` bash
mctp address add <eid> dev <link>
mctp address del <eid> dev <link>
```

Example:

``` bash
mctp address add 9 dev mctp0
```

------------------------------------------------------------------------

## 33. `mctp route`

Inspect routes:

``` bash
mctp route show
```

Add a route:

``` bash
mctp route add \
    net <network-id> \
    eid <eid> \
    via <link>
```

The kernel design also documents options such as hardware address, MTU
and metric. citeturn0search0

------------------------------------------------------------------------

## 34. `mctp stat`

Inspect MCTP socket status:

``` bash
mctp stat
```

Useful when debugging userspace applications using the kernel MCTP
sockets.

------------------------------------------------------------------------

## 35. Debugging MCTP

Always debug from the bottom upward:

``` text
Hardware
   ↓
Binding Driver
   ↓
MCTP Interface
   ↓
Network
   ↓
Local EID
   ↓
Route
   ↓
Remote EID
   ↓
MCTP Socket
   ↓
Higher-Level Protocol
```

Do not begin with PLDM if the MCTP route is broken.

------------------------------------------------------------------------

## 36. Debugging Checklist

### 1. Physical binding

``` text
PCIe?
I²C?
UART?
LPC?
```

### 2. Interface

``` bash
ip link
mctp link
```

### 3. Network

``` bash
mctp network
```

### 4. Local EID

``` bash
mctp address
```

### 5. Routes

``` bash
mctp route show
```

### 6. Socket status

``` bash
mctp stat
```

### 7. Higher-level service

For PLDM:

``` text
pldmd
```

------------------------------------------------------------------------

## 37. MCTP Endpoint Discovery → PLDM

This is the bridge to Day 27.

The OpenBMC PLDM design describes `pldmd` using MCTP D-Bus information
to enumerate MCTP endpoints.

Conceptually:

``` text
MCTP
  ↓
Endpoint discovered
  ↓
EID
  ↓
Supported Message Types
  ↓
PLDM supported?
  ↓
pldmd
  ↓
PLDM Terminus
```

`pldmd` watches MCTP endpoint changes and maintains PLDM terminus state.
citeturn0search3

------------------------------------------------------------------------

## 38. Why EID Matters for PLDM

Suppose:

``` text
BMC   = EID 9
GPU   = EID 10
NIC   = EID 11
Retimer = EID 12
```

A PLDM request can target:

``` text
PLDM → EID 10
```

without requiring the PLDM application to understand whether the
underlying binding is PCIe, SMBus or another supported transport.

------------------------------------------------------------------------

## 39. Complete MCTP + PLDM Flow

``` text
BMC Application
       ↓
PLDM Request
       ↓
MCTP Socket
       ↓
Destination EID = 10
       ↓
MCTP Routing
       ↓
Binding
       ↓
Remote MCTP Endpoint
       ↓
PLDM Request
       ↓
PLDM Response
       ↓
MCTP
       ↓
BMC Application
```

This is the architecture Day 27 will explore in detail.

------------------------------------------------------------------------

## 40. MCTP vs IPMI

  -----------------------------------------------------------------------
  Concept                 IPMI                    MCTP
  ----------------------- ----------------------- -----------------------
  Primary role            Management              Transport protocol
                          protocol/interface      

  Addressing              IPMI-specific           EID

  Transport abstraction   More tightly coupled    Explicitly separated

  Higher-level protocol   IPMI commands           PLDM and others

  Extensibility           OEM extensions often    Designed to carry
                          used                    multiple protocols

  Physical bindings       IPMI-defined mechanisms Separate MCTP bindings

  Linux model             IPMI stack              `AF_MCTP` sockets
  -----------------------------------------------------------------------

OpenBMC's MCTP design specifically discusses limitations of extending
IPMI and motivates a transport/messaging separation. citeturn0search1

------------------------------------------------------------------------

## 41. MCTP vs Redfish

They solve different problems.

### Redfish

``` text
Management API
REST / JSON
Remote management
```

### MCTP

``` text
Transport
Endpoint communication
Device-to-device management messaging
```

Therefore:

``` text
Redfish ≠ MCTP
```

A system can use Redfish, MCTP and PLDM at different layers.

------------------------------------------------------------------------

## 42. MCTP vs D-Bus

### D-Bus

Primarily:

``` text
Process ↔ Process inside BMC
```

### MCTP

Primarily:

``` text
MCTP Endpoint ↔ MCTP Endpoint
```

A PLDM daemon may use D-Bus internally while communicating with a remote
device over MCTP externally.

------------------------------------------------------------------------

## 43. MCTP vs Ethernet

MCTP is not simply another TCP/IP protocol.

``` text
Ethernet
  ↓
IP
  ↓
TCP
  ↓
Application
```

versus:

``` text
Physical Binding
  ↓
MCTP
  ↓
PLDM
```

MCTP can operate over management channels without requiring a
conventional IP stack.

------------------------------------------------------------------------

## 44. MCTP Control vs PLDM

Do not confuse:

``` text
MCTP Control Protocol
```

with:

``` text
PLDM commands
```

### MCTP Control

Transport-level functions:

``` text
EID
Endpoint information
Transport management
```

### PLDM

Platform-management functions:

``` text
Sensors
PDRs
FRU
Firmware
BIOS
Platform events
```

------------------------------------------------------------------------

## 45. Source / Repository Map

Useful OpenBMC repositories and documents:

``` text
openbmc/docs
```

MCTP architecture and design.

``` text
openbmc/libmctp
```

Portable MCTP implementation.

``` text
openbmc/pldm
```

PLDM implementation.

``` text
openbmc/libpldm
```

PLDM protocol library.

OpenBMC's package-group configuration also includes MCTP and PLDM as
DMTF PMCI implementations. citeturn0search6

------------------------------------------------------------------------

## 46. Important Source Files

Start source reading with:

``` text
docs/designs/mctp/mctp.md
docs/designs/mctp/mctp-kernel.md
docs/designs/mctp/mctp-userspace.md
docs/designs/pldm-stack.md
```

These documents establish the architecture before diving into
implementation code.
citeturn0search0turn0search1turn0search2turn0search3

------------------------------------------------------------------------

## 47. Example Topology

A simple MCTP network:

``` text
                Network 1
                    │
          ┌─────────┴─────────┐
          │                   │
      BMC EID 9          Device EID 10
          │                   │
        mctp0              Binding
          │                   │
          └────── MCTP ───────┘
```

A more complex platform:

``` text
                       BMC
                  ┌─────┴─────┐
                  ▼           ▼
                I²C          PCIe
                  │           │
                EID 10      EID 20
                  │           │
                Device      Device
```

------------------------------------------------------------------------

## 48. Example End-to-End Message

Suppose the BMC needs information from a device:

``` text
1. Application creates request
2. Higher-level protocol encodes it
3. MCTP socket targets EID 10
4. Kernel MCTP finds the route
5. MCTP packetizes the message
6. Binding transmits packets
7. Remote endpoint receives them
8. Remote MCTP reassembles the message
9. Higher-level protocol processes it
10. Response returns using MCTP
```

This is the practical meaning of:

> **MCTP is the transport layer.**

------------------------------------------------------------------------

## 49. Common Mistake: "MCTP Replaces Redfish"

Incorrect.

``` text
Redfish → management API
MCTP    → transport
PLDM    → platform-management messaging
```

They can coexist.

------------------------------------------------------------------------

## 50. Common Mistake: "MCTP Is Only Host ↔ BMC"

Not necessarily.

MCTP can connect:

``` text
Host
BMC
Management Controllers
PCIe Devices
Power Controllers
Security Devices
Other Platform Components
```

The OpenBMC design describes MCTP endpoints broadly as platform entities
that communicate over MCTP. citeturn0search1

------------------------------------------------------------------------

## 51. Common Mistake: "EID Is a MAC Address"

No.

``` text
EID
≠
MAC Address
```

EID identifies an MCTP endpoint within an MCTP network.

The physical address used by a particular binding is separate.

------------------------------------------------------------------------

## 52. Common Mistake: "EID Is Globally Unique"

Not necessarily.

EIDs are scoped to an MCTP network. Different networks can contain
duplicate EID values. citeturn0search0

------------------------------------------------------------------------

## 53. Common Mistake: "MCTP Always Uses Kernel Support"

Not historically.

OpenBMC documents:

``` text
Userspace MCTP
```

and:

``` text
Kernel MCTP
```

The kernel approach is recommended for new designs. citeturn0search1

------------------------------------------------------------------------

## 54. Common Mistake: "libmctp Is Required on Linux"

Not when kernel MCTP support is available.

The current `libmctp` documentation recommends kernel MCTP sockets for
Linux applications when supported. citeturn0search5

------------------------------------------------------------------------

## 55. Interview Questions

### Q: What is MCTP?

> MCTP is a transport protocol for communication between management
> components. It separates transport from higher-level management
> protocols and supports multiple physical bindings. In OpenBMC, the
> kernel implementation provides a socket API for userspace
> applications.

### Q: Why introduce MCTP when IPMI exists?

> OpenBMC identified limitations in using IPMI as both transport and
> messaging model. MCTP separates transport from higher-level messaging,
> allowing multiple bindings and protocols such as PLDM.

### Q: What is an EID?

> EID is an Endpoint ID that identifies an MCTP endpoint within an MCTP
> network.

### Q: MCTP vs PLDM?

> MCTP provides transport; PLDM provides higher-level
> platform-management messaging.

### Q: What is AF_MCTP?

> The Linux address family used by the kernel MCTP implementation to
> provide socket-based communication.

### Q: Packet vs message?

> A message is the higher-level data exchanged between endpoints;
> packets are the transport units used to carry that message over a
> binding.

### Q: Interface vs network?

> An interface represents a physical MCTP binding; a network defines the
> EID address space and routing context.

------------------------------------------------------------------------

## 56. Day 25 → Day 26

``` text
DAY 25
Fault
 ↓
SEL
 ↓
phosphor-sel-logger
 ↓
IPMI
 ↓
ipmitool
```

Now:

``` text
DAY 26
Endpoint
 ↓
MCTP
 ↓
EID
 ↓
Transport
 ↓
PLDM
```

The series moves from traditional management toward modern
platform-management communication.

------------------------------------------------------------------------

## 57. Day 26 → Day 27

Day 26 establishes:

``` text
MCTP
 ↓
Transport
 ↓
Endpoint
 ↓
EID
 ↓
Message Delivery
```

Day 27 asks:

``` text
What does the message actually do?
```

Answer:

``` text
PLDM
 ↓
PLDM Types
 ↓
PLDM Commands
 ↓
PDRs
 ↓
Sensors
 ↓
FRU
 ↓
Firmware
 ↓
Platform Events
```

------------------------------------------------------------------------

## 58. Complete Architecture

``` text
                         PLATFORM
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
      HOST                  BMC                DEVICE
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                       MCTP Network
                            │
                 ┌──────────┴──────────┐
                 │                     │
             Endpoint                Endpoint
              EID 8                  EID 10
                 │                     │
                 └──────────┬──────────┘
                            │
                           MCTP
                            │
                         Transport
                            │
                           PLDM
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
            Sensors        FRU       Firmware
```

------------------------------------------------------------------------

## 59. Big Picture

The OpenBMC architecture now includes several distinct layers:

``` text
Hardware
   ↓
Sensors / FRU / Devices
   ↓
D-Bus
   ↓
OpenBMC Services
   ↓
Management Interfaces
   ├── Redfish
   └── IPMI / SEL

Device-to-device management
   ↓
MCTP
   ↓
PLDM
```

Keep the layers separate:

``` text
D-Bus
 → internal BMC IPC

Redfish / IPMI
 → management interfaces

MCTP
 → endpoint transport

PLDM
 → platform-management messaging
```

------------------------------------------------------------------------

## 60. What We Learned Today

-   MCTP = Management Component Transport Protocol.
-   MCTP separates transport from higher-level management messaging.
-   OpenBMC's MCTP architecture was motivated by limitations of
    traditional IPMI communication.
-   MCTP can operate over multiple physical bindings.
-   An MCTP endpoint is identified using an EID.
-   EIDs are scoped to an MCTP network.
-   MCTP interfaces and networks are separate concepts.
-   Messages can be split into packets and reassembled.
-   MCTP supports routing and forwarding.
-   MCTP Control Protocol handles transport-level management.
-   MCTP tags help correlate request/response traffic.
-   OpenBMC documents both userspace and kernel MCTP implementations.
-   The kernel implementation is recommended for new designs.
-   Linux exposes MCTP through `AF_MCTP` sockets.
-   The `mctp` utility manages links, networks, addresses and routes.
-   `libmctp` remains useful when kernel MCTP is unavailable or a
    standalone stack is required.
-   PLDM runs above MCTP.
-   MCTP is not a replacement for Redfish or D-Bus.
-   MCTP provides a reusable transport foundation for modern platform
    management.

------------------------------------------------------------------------

## 61. One-Line Summary

> **MCTP gives OpenBMC a reusable transport layer for communication
> between management endpoints, separating physical transport from
> higher-level protocols such as PLDM and enabling flexible
> platform-management architectures.**

------------------------------------------------------------------------

## 62. Final Mental Model

``` text
        HOST / BMC / DEVICE
                 │
                 ▼
          Physical Binding
       ┌──────┬──────┬──────┐
       ▼      ▼      ▼      ▼
      I²C    PCIe   UART    LPC
       └──────┬──────┬──────┘
              ▼
             MCTP
              │
       ┌──────┴──────┐
       ▼             ▼
      EID          Routing
       │             │
       └──────┬──────┘
              ▼
        Higher Protocol
              │
             PLDM
              │
       Platform Management
```

### Key sentence

> **MCTP tells the platform how to transport the message; PLDM tells the
> platform what the message means.**


------------------------------------------------------------------------

# References

1.  OpenBMC MCTP & PLDM overview\
    https://github.com/openbmc/docs/blob/master/designs/mctp/mctp.md

2.  OpenBMC in-kernel MCTP design\
    https://github.com/openbmc/docs/blob/master/designs/mctp/mctp-kernel.md

3.  OpenBMC userspace MCTP design\
    https://github.com/openbmc/docs/blob/master/designs/mctp/mctp-userspace.md

4.  OpenBMC PLDM stack design\
    https://github.com/openbmc/docs/blob/master/designs/pldm-stack.md

5.  OpenBMC libmctp\
    https://github.com/openbmc/libmctp

6.  OpenBMC PLDM\
    https://github.com/openbmc/pldm

7.  OpenBMC libpldm\
    https://github.com/openbmc/libpldm

8.  OpenBMC package configuration for DMTF PMCI implementations\
    https://github.com/openbmc/openbmc/blob/master/meta-phosphor/recipes-phosphor/packagegroups/packagegroup-obmc-apps.bb

------------------------------------------------------------------------

# End of Day 26

### OpenBMC Learning Series

**MCTP --- From Endpoint → EID → Transport → PLDM**

> Separate the transport.\
> Identify the endpoint.\
> Deliver the message.\
> Let the higher-level protocol define its meaning.

**Learn. Build. Go Deeper.**
