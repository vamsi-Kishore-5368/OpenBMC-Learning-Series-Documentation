# OpenBMC Learning Series --- Day 28

# OpenBMC Yocto & BitBake Build System

## From Recipe → Package → RootFS → BMC Image

## 1. Day 28 in the Series

Day 25 covered IPMI/SEL, Day 26 covered MCTP, and Day 27 covered PLDM.

Day 28 moves from **runtime protocols** to the **build system that turns
OpenBMC source and metadata into BMC firmware**.

``` text
Day 25 → IPMI / SEL
Day 26 → MCTP
Day 27 → PLDM
Day 28 → Yocto / BitBake
```

The central pipeline is:

``` text
OpenBMC Source
      ↓
Yocto Layers
      ↓
Recipes / Configuration
      ↓
BitBake
      ↓
Tasks + Dependencies
      ↓
Packages
      ↓
Root Filesystem
      ↓
BMC Image
      ↓
Flash / QEMU / Boot
```

The official OpenBMC README uses `setup <machine>` to configure a target
and `bitbake obmc-phosphor-image` to build the image. \[1\]

------------------------------------------------------------------------

## 2. What Is Yocto?

The Yocto Project is a build ecosystem for creating customized
Linux-based distributions for target hardware.

Yocto is **not simply a Linux distribution**. It provides metadata,
tools and build infrastructure used to construct a tailored Linux
system.

For OpenBMC, this is valuable because different BMC platforms can
require different:

-   CPU architectures
-   bootloaders
-   kernels
-   device trees
-   hardware interfaces
-   storage layouts
-   management services
-   sensors
-   networking
-   security configuration

Instead of manually compiling every component, the build metadata
describes how each component should be fetched, configured, compiled,
installed, packaged and included in an image.

------------------------------------------------------------------------

## 3. What Is BitBake?

BitBake is the task execution engine used by OpenEmbedded/Yocto.

A useful distinction is:

``` text
Yocto / OpenEmbedded
        ↓
Build framework + metadata
        ↓
BitBake
        ↓
Parse metadata
Resolve dependencies
Execute tasks
        ↓
Packages / Images
```

BitBake reads metadata such as:

``` text
.bb
.bbappend
.conf
.inc
.bbclass
```

and constructs a dependency graph of tasks.

BitBake tasks are execution units and conventionally use the `do_`
prefix. \[2\]

### Interview answer

> Yocto provides the build framework and metadata ecosystem for creating
> custom Linux distributions, while BitBake is the engine that parses
> that metadata, resolves dependencies and executes build tasks.

------------------------------------------------------------------------

## 4. Why OpenBMC Uses Yocto

OpenBMC supports many hardware platforms. The current OpenBMC repository
contains common and vendor/platform layers such as:

``` text
meta-phosphor
meta-openembedded
meta-security
meta-arm
meta-ibm
meta-hpe
meta-google
meta-intel-openbmc
meta-nvidia
meta-quanta
meta-supermicro
meta-yadro
...
```

The exact set depends on the current repository and target machine.
\[1\]

This lets OpenBMC separate:

``` text
Common functionality
        +
Platform-specific functionality
        ↓
Target BMC image
```

------------------------------------------------------------------------

## 5. What Is a Yocto Layer?

A layer is a collection of build metadata.

A layer can contain:

-   recipes
-   `.bbappend` files
-   configuration
-   classes
-   machine definitions
-   distribution configuration
-   patches
-   files
-   image metadata

Typical structure:

``` text
meta-example/
├── conf/
│   └── layer.conf
├── recipes-example/
│   └── my-service/
│       ├── my-service.bb
│       └── files/
└── classes/
```

OpenBMC documentation describes its layers as the `meta-*` directories
in the repository. \[3\]

------------------------------------------------------------------------

## 6. Why Layers Matter

Suppose a common service is used by many BMC platforms.

A common layer can provide:

``` text
service.bb
```

A platform layer can customize it using:

``` text
service.bbappend
```

without copying the entire recipe.

``` text
Common Layer
    service.bb
         +
Platform Layer
    service.bbappend
         ↓
Final BitBake metadata
```

Benefits:

-   reuse
-   separation of concerns
-   platform customization
-   easier maintenance
-   easier upstream contribution

------------------------------------------------------------------------

## 7. What Is a Recipe?

A recipe is a `.bb` file containing metadata that tells BitBake how to
build software.

A recipe may define:

``` text
SUMMARY
DESCRIPTION
LICENSE
SRC_URI
PV
DEPENDS
RDEPENDS
S
do_configure
do_compile
do_install
FILES
```

Simplified example:

``` bitbake
SUMMARY = "Example BMC application"
LICENSE = "Apache-2.0"

SRC_URI = "git://example.org/project.git;branch=main"
S = "${WORKDIR}/git"

DEPENDS = "systemd"

do_compile() {
    oe_runmake
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 myapp ${D}${bindir}/
}
```

A recipe describes **how to build and package software**; it is not
itself the final package.

------------------------------------------------------------------------

## 8. Recipe vs Package

``` text
Recipe
  ↓
Instructions / metadata
  ↓
Build
  ↓
Package
  ↓
Installable output
```

One recipe can produce multiple packages:

``` text
foo.bb
  ├── foo
  ├── foo-dev
  └── foo-dbg
```

The actual split depends on the recipe and packaging metadata.

------------------------------------------------------------------------

## 9. What Is .bbappend?

A `.bbappend` extends an existing recipe from another layer.

Example:

``` text
meta-common/
    recipes-app/foo/foo.bb

meta-platform/
    recipes-app/foo/foo.bbappend
```

A `.bbappend` can add:

-   patches
-   files
-   dependencies
-   configuration
-   install steps
-   platform-specific behavior

OpenBMC's development documentation explicitly describes `.bbappend` as
a mechanism for appending to `.bb` recipes. \[3\]

------------------------------------------------------------------------

## 10. Configuration Files

Important configuration areas include:

``` text
conf/local.conf
conf/bblayers.conf
machine configuration
distribution configuration
layer.conf
```

### local.conf

Controls build-specific settings such as machine selection, parallelism,
cache/download locations and local development configuration.

### bblayers.conf

Defines the layers available to the build.

Inspect layers with:

``` bash
bitbake-layers show-layers
```

Do not put every reusable project/platform change into `local.conf`.
Reusable changes normally belong in appropriate layer metadata.

------------------------------------------------------------------------

## 11. Machine Configuration

A machine describes the target hardware.

It can affect:

-   CPU architecture
-   kernel
-   bootloader
-   device tree
-   machine features
-   image format
-   hardware-specific packages
-   storage/flash configuration

OpenBMC selects the target through:

``` bash
. setup <machine>
```

For example, where supported:

``` bash
. setup romulus
```

The current supported machine list should always be taken from the
OpenBMC tree. \[1\]

------------------------------------------------------------------------

## 12. Distribution Configuration

Distribution configuration defines higher-level distribution behavior.

OpenBMC metadata uses concepts such as:

``` text
DISTRO_FEATURES
DISTROOVERRIDES
PREFERRED_PROVIDER
VIRTUAL-RUNTIME
```

The selected machine, distro and layer metadata together determine the
final build configuration.

------------------------------------------------------------------------

## 13. Packagegroups

A packagegroup collects related packages.

Instead of making an image directly list many packages:

``` text
service-a
service-b
service-c
service-d
```

a packagegroup can represent a functional collection:

``` text
packagegroup-<feature>
       ↓
service-a
service-b
service-c
service-d
```

This is useful for composing OpenBMC functionality cleanly.

------------------------------------------------------------------------

## 14. Dependencies

Two important variables are:

### DEPENDS

Build-time dependency:

``` bitbake
DEPENDS += "systemd"
```

### RDEPENDS

Runtime dependency:

``` bitbake
RDEPENDS:${PN} += "bash"
```

Think:

``` text
DEPENDS
Recipe A → Recipe B
"Need B to build A"

RDEPENDS
Package A → Package B
"Need B at runtime"
```

------------------------------------------------------------------------

# 15. BitBake Task Model

Recipes are executed through tasks.

A simplified normal recipe flow is:

``` text
do_fetch
   ↓
do_unpack
   ↓
do_patch
   ↓
do_configure
   ↓
do_compile
   ↓
do_install
   ↓
do_package
   ↓
do_package_write_*
```

Image construction adds:

``` text
do_rootfs
   ↓
do_image
   ↓
do_image_complete
```

The exact task graph depends on the recipe, inherited classes and
dependencies. \[4\]

------------------------------------------------------------------------

## 16. do_fetch

Fetches source described by `SRC_URI`.

Example:

``` bitbake
SRC_URI = "git://example.org/project.git;branch=main"
```

The Yocto task reference defines `do_fetch` as fetching source using
`SRC_URI`. \[4\]

------------------------------------------------------------------------

## 17. do_unpack

Unpacks downloaded source into the recipe work area.

``` text
Downloaded source
      ↓
do_unpack
      ↓
WORKDIR / source tree
```

------------------------------------------------------------------------

## 18. do_patch

Applies patches specified by recipe metadata.

``` bitbake
SRC_URI += "file://0001-platform-fix.patch"
```

Flow:

``` text
Source
  +
Patches
  ↓
Patched Source
```

------------------------------------------------------------------------

## 19. do_configure

Prepares the source for compilation.

Depending on the project, this can involve:

-   CMake
-   Meson
-   Autotools
-   Make
-   custom scripts

Yocto defines `do_configure` as configuring build-time options. \[4\]

------------------------------------------------------------------------

## 20. do_compile

Builds the software.

``` text
Source
  ↓
Compiler / Build System
  ↓
Objects
  ↓
Executable / Library
```

The default behavior depends on the recipe and inherited classes;
Makefile-based recipes can use `oe_runmake`. \[4\]

------------------------------------------------------------------------

## 21. do_install

Installs built files into the recipe staging directory:

``` text
${D}
```

Example:

``` bitbake
do_install() {
    install -d ${D}${bindir}
    install -m 0755 myapp ${D}${bindir}/
}
```

This may create:

``` text
${D}/usr/bin/myapp
```

`do_install` does not mean the file is already in the final BMC image.

------------------------------------------------------------------------

## 22. do_package

Turns staged files into packages.

``` text
${D}
  ↓
Package splitting
  ↓
foo
foo-dev
foo-dbg
```

The resulting packages become inputs to image construction.

------------------------------------------------------------------------

## 23. do_rootfs

For an image recipe, `do_rootfs` creates the target root filesystem from
the selected packages.

``` text
Package metadata
      ↓
Selected packages
      ↓
do_rootfs
      ↓
Target RootFS
```

The Yocto task reference states that `do_rootfs` identifies packages for
installation and creates the root filesystem before image generation.
\[4\]

------------------------------------------------------------------------

## 24. do_image

`do_image` starts image generation after rootfs creation.

Conceptually:

``` text
RootFS
 +
Kernel
 +
Boot artifacts
 +
Image configuration
       ↓
    do_image
       ↓
   BMC image
```

Yocto documents `do_image` as the task that starts image generation
after `do_rootfs`. \[4\]

------------------------------------------------------------------------

## 25. do_image_complete

Final image-generation/post-processing work happens in
`do_image_complete`.

Simplified:

``` text
do_rootfs
    ↓
do_image
    ↓
do_image_complete
```

The exact image formats depend on the machine and configuration.

------------------------------------------------------------------------

# 26. The Complete Recipe → Image Flow

``` text
                 Recipe (.bb)
                       |
                       v
                    BitBake
                       |
        +--------------+--------------+
        |              |              |
      Fetch          Patch         Configure
        |              |              |
        +--------------+--------------+
                       |
                    Compile
                       |
                       v
                    Install
                       |
                       v
                    Package
                       |
                       v
                 Package output
                       |
                       v
                   do_rootfs
                       |
                       v
                    RootFS
                       |
                       v
                   do_image
                       |
                       v
                   BMC Image
```

This is the core mental model for Day 28.

------------------------------------------------------------------------

# 27. What Happens When You Run bitbake?

Consider:

``` bash
bitbake obmc-phosphor-image
```

BitBake does not simply "compile OpenBMC".

At a high level it:

``` text
1. Parses metadata
2. Loads configured layers
3. Reads MACHINE / DISTRO configuration
4. Finds recipes and providers
5. Resolves dependencies
6. Builds the task graph
7. Reuses cache where possible
8. Fetches sources
9. Builds dependencies
10. Packages software
11. Creates rootfs
12. Generates the image
```

This is the standard OpenBMC build target documented by the official
repository. \[1\]

------------------------------------------------------------------------

# 28. Dependency Graph

A simplified OpenBMC dependency graph:

``` text
obmc-phosphor-image
        |
        +-- bmcweb
        |     |
        |     +-- libraries
        |
        +-- phosphor services
        |
        +-- systemd
        |
        +-- kernel
        |
        +-- bootloader
        |
        +-- platform packages
```

BitBake determines task ordering from dependency metadata.

It does not simply execute recipes in the order they appear in a
directory.

------------------------------------------------------------------------

# 29. Shared State (sstate)

Yocto uses a shared-state cache.

First build:

``` text
Source → Build → Result
                  ↓
                sstate
```

Later build:

``` text
Source / metadata
       ↓
Can result be reused?
       ↓
Yes → restore cached result
No  → execute task
```

This is a major reason incremental builds can be much faster than the
initial build.

OpenBMC's development documentation notes that later builds can use
cached data from the first build. \[3\]

------------------------------------------------------------------------

# 30. Build Directory

After environment setup, a build directory contains generated state and
configuration.

Conceptually:

``` text
build/
├── conf/
│   ├── local.conf
│   └── bblayers.conf
└── tmp/
    ├── work/
    ├── deploy/
    ├── sysroots/
    └── ...
```

Exact layout can vary by OpenEmbedded/Yocto version and configuration.

------------------------------------------------------------------------

# 31. Important Generated Areas

### tmp/work/

Recipe-specific work areas.

``` text
tmp/work/<arch>/<recipe>/<version>/
```

### tmp/deploy/images/`<machine>`{=html}/

Machine-specific deployment artifacts.

### WORKDIR/temp/

Task execution scripts and logs.

Yocto documents task `run` files, logs and task ordering information
under the recipe's `WORKDIR/temp/`. \[5\]

------------------------------------------------------------------------

# 32. Inspect Tasks

List tasks:

``` bash
bitbake -c listtasks <recipe>
```

Example:

``` bash
bitbake -c listtasks bmcweb
```

The Yocto documentation explicitly provides
`bitbake -c listtasks recipename`. \[5\]

------------------------------------------------------------------------

# 33. Run an Individual Task

Examples:

``` bash
bitbake -c fetch bmcweb
bitbake -c compile bmcweb
bitbake -c install bmcweb
bitbake -c package bmcweb
```

This is useful for targeted debugging instead of rebuilding the whole
image.

------------------------------------------------------------------------

# 34. bitbake -c devshell

Open a development shell:

``` bash
bitbake -c devshell <recipe>
```

Useful for:

-   compiler environment inspection
-   manual build commands
-   testing configuration
-   examining variables
-   reproducing failures

------------------------------------------------------------------------

# 35. bitbake -e

Inspect the final expanded environment:

``` bash
bitbake -e <recipe>
```

Examples:

``` bash
bitbake -e bmcweb | grep '^SRC_URI='
bitbake -e bmcweb | grep '^DEPENDS='
```

This is especially useful when multiple layers, includes, classes and
overrides modify the same variable.

------------------------------------------------------------------------

# 36. bitbake-layers

Useful commands:

``` bash
bitbake-layers show-layers
```

``` bash
bitbake-layers show-recipes
```

``` bash
bitbake-layers show-recipes bmcweb
```

These help identify:

-   configured layers
-   available recipes
-   providers
-   versions

------------------------------------------------------------------------

# 37. Recipe Selection

Recipe selection can be affected by:

-   layer metadata
-   recipe versions
-   machine overrides
-   distro configuration
-   providers
-   `PREFERRED_PROVIDER`
-   `PREFERRED_VERSION`

For virtual providers:

``` bitbake
PREFERRED_PROVIDER_virtual/foo = "foo-provider"
```

Conceptually:

``` text
virtual/foo
   |
   +-- provider A
   +-- provider B
   +-- provider C
           ↓
     selected provider
```

OpenBMC metadata uses provider-selection mechanisms in platform
configuration. \[6\]

------------------------------------------------------------------------

# 38. Overrides

Modern BitBake syntax uses colon overrides.

Examples:

``` bitbake
PACKAGECONFIG:append = " feature"
```

Machine-specific:

``` bitbake
PACKAGECONFIG:append:my-machine = " feature"
```

Append a file:

``` bitbake
SRC_URI:append:my-machine = " file://machine.cfg"
```

When reading older OpenBMC metadata, you may also encounter historical
underscore syntax.

------------------------------------------------------------------------

# 39. Adding a Custom Layer

Example:

``` text
meta-vamsi/
├── conf/
│   └── layer.conf
└── recipes-vamsi/
    └── my-bmc-tool/
        ├── my-bmc-tool.bb
        └── files/
```

The layer must be included in the build configuration.

Then BitBake can discover the recipe.

------------------------------------------------------------------------

# 40. Example Custom Recipe

``` bitbake
SUMMARY = "Custom BMC tool"
LICENSE = "CLOSED"

SRC_URI = "file://my-bmc-tool.service"

inherit systemd

SYSTEMD_SERVICE:${PN} = "my-bmc-tool.service"

do_install() {
    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/my-bmc-tool.service \
        ${D}${systemd_system_unitdir}/
}
```

This is a simplified example. Real projects should use appropriate
licensing, source, packaging and service configuration.

------------------------------------------------------------------------

# 41. Adding a .bbappend

Example:

``` text
foo.bb
foo.bbappend
```

The append might contain:

``` bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI += "file://platform.conf"

do_install:append() {
    install -d ${D}${sysconfdir}/foo
    install -m 0644 ${WORKDIR}/platform.conf \
        ${D}${sysconfdir}/foo/
}
```

This lets a platform layer customize the common recipe.

------------------------------------------------------------------------

# 42. Adding a Package to the Image

Important distinction:

``` text
Recipe builds successfully
        ≠
Package is included in image
```

The package must be selected by:

-   image metadata
-   packagegroup
-   dependency
-   platform-specific image append
-   another appropriate image composition mechanism

Flow:

``` text
Recipe
  ↓
Package
  ↓
Image selection
  ↓
do_rootfs
  ↓
BMC image
```

------------------------------------------------------------------------

# 43. Kernel / Device Tree / Bootloader

OpenBMC image construction is not only userspace.

A BMC firmware image can involve:

``` text
Bootloader
Kernel
Device Tree
RootFS
Image layout / flash metadata
```

The machine configuration determines much of this platform-specific
integration.

------------------------------------------------------------------------

# 44. OpenBMC Image Target

The official build command is:

``` bash
bitbake obmc-phosphor-image
```

After a successful build, artifacts are normally deployed under:

``` text
tmp/deploy/images/<machine>/
```

The exact artifact names and formats vary by machine and current
metadata. \[1\]\[3\]

------------------------------------------------------------------------

# 45. QEMU Flow

OpenBMC supports QEMU-based development/testing for supported platforms.

Conceptually:

``` text
OpenBMC Source
      ↓
BitBake
      ↓
BMC Image
      ↓
QEMU
      ↓
OpenBMC boots
      ↓
Tests
```

The official OpenBMC repository documents QEMU-based system-level CI for
firmware images. \[1\]

------------------------------------------------------------------------

# 46. Build Failure Debugging

If:

``` bash
bitbake obmc-phosphor-image
```

fails, first identify:

``` text
Recipe
Task
Error
Dependency
```

For example:

``` text
ERROR: Task (.../foo.bb:do_compile) failed
```

Focus first on:

``` text
foo
do_compile
```

Then inspect the task log.

Do not immediately delete the whole build directory.

------------------------------------------------------------------------

# 47. Task Logs

Recipe task logs are under the recipe's work directory:

``` text
tmp/work/
    ...
    <recipe>/
       <version>/
          temp/
             log.do_compile.*
             run.do_compile.*
             log.task_order
```

The Yocto task documentation describes these generated task logs and run
scripts. \[5\]

------------------------------------------------------------------------

# 48. Common Build Errors

### Missing provider

``` text
Nothing PROVIDES 'foo'
```

Investigate:

-   layer inclusion
-   recipe name
-   provider
-   machine compatibility

### Missing runtime provider

``` text
Nothing RPROVIDES 'foo'
```

Investigate:

-   package name
-   runtime dependency
-   package provider

### Fetch failure

Investigate:

-   `SRC_URI`
-   revision
-   network
-   checksum
-   authentication

### Patch failure

Investigate:

-   patch context
-   source revision
-   patch ordering

### Compile failure

Investigate:

-   compiler error
-   dependency
-   source compatibility
-   build flags

------------------------------------------------------------------------

# 49. Clean Tasks

Common commands:

``` bash
bitbake -c clean <recipe>
```

``` bash
bitbake -c cleansstate <recipe>
```

``` bash
bitbake -c cleanall <recipe>
```

They are not equivalent.

Use the least destructive command that solves the problem.

Yocto documents the different effects of `do_clean`, `do_cleansstate`
and `do_cleanall`. \[5\]

------------------------------------------------------------------------

# 50. Why Not Always Use cleansstate?

Because removing shared-state results can force expensive work to run
again.

A better workflow is:

``` text
Build failure
    ↓
Find recipe
    ↓
Find task
    ↓
Read log
    ↓
Inspect metadata
    ↓
Fix root cause
    ↓
Re-run required task
```

Only clean/rebuild when there is evidence that stale build state is
actually involved.

------------------------------------------------------------------------

# 51. Debugging Recipe Metadata

A practical sequence:

``` bash
bitbake-layers show-layers
```

``` bash
bitbake-layers show-recipes <recipe>
```

``` bash
bitbake -e <recipe>
```

Then inspect:

``` text
.bb
.bbappend
.conf
.bbclass
```

This helps trace how the final metadata was constructed.

------------------------------------------------------------------------

# 52. Example: Custom BMC Service End-to-End

Suppose we create:

``` text
my-bmc-service
```

Desired runtime files:

``` text
/usr/bin/my-bmc-service
/etc/systemd/system/my-bmc-service.service
```

Build flow:

``` text
my-bmc-service.bb
       ↓
do_compile
       ↓
do_install
       ↓
do_package
       ↓
my-bmc-service package
       ↓
Image/packagegroup selection
       ↓
do_rootfs
       ↓
BMC RootFS
       ↓
do_image
       ↓
BMC Image
```

At runtime:

``` text
BMC boots
   ↓
systemd
   ↓
my-bmc-service
```

------------------------------------------------------------------------

# 53. Example: Modifying bmcweb

Suppose a platform requires a platform-specific configuration or patch.

Use:

``` text
bmcweb.bbappend
```

instead of copying the complete common recipe.

``` text
Common bmcweb recipe
        +
Platform bbappend
        ↓
Platform-specific build
        ↓
Package
        ↓
Image
```

This is a practical example of why Yocto layers are powerful.

------------------------------------------------------------------------

# 54. Build-Time vs Runtime

This distinction is critical for OpenBMC engineers.

### Build time

``` text
Yocto
BitBake
Layers
Recipes
Classes
Packages
Image generation
```

### Runtime

``` text
Linux
systemd
D-Bus
OpenBMC services
bmcweb
IPMI
MCTP
PLDM
Sensors
Logging
```

The same component can therefore be studied from two angles:

``` text
How is it built?
→ Yocto / BitBake

How does it run?
→ Linux / systemd / D-Bus / OpenBMC
```

------------------------------------------------------------------------

# 55. Connecting Day 28 to Earlier Days

A PLDM change from Day 27 might require:

``` text
PLDM source change
      ↓
Recipe metadata
      ↓
BitBake
      ↓
Package
      ↓
BMC image
      ↓
Boot
      ↓
pldmd
      ↓
MCTP
      ↓
Remote endpoint
```

Similarly, a change to:

-   IPMI
-   Redfish
-   D-Bus services
-   sensors
-   logging
-   Entity Manager
-   MCTP
-   PLDM

eventually has to become part of the firmware image.

That is why understanding Yocto/BitBake is fundamental to OpenBMC
development.

------------------------------------------------------------------------

# 56. Interview Questions

### Q1. What is Yocto?

A build ecosystem for creating customized Linux distributions for target
hardware.

### Q2. What is BitBake?

The build/task execution engine that parses metadata, resolves
dependencies and runs tasks.

### Q3. What is a recipe?

A `.bb` file describing how a software component is fetched, built,
installed and packaged.

### Q4. What is a layer?

A collection of Yocto metadata such as recipes, configuration, classes
and machine information.

### Q5. What is `.bbappend`?

A mechanism for extending an existing recipe from another layer.

### Q6. What is `local.conf`?

Build-specific configuration.

### Q7. What is `bblayers.conf`?

Configuration describing which layers participate in the build.

### Q8. What does `bitbake obmc-phosphor-image` do?

Builds the OpenBMC image target and its required dependencies for the
configured machine.

### Q9. What does `do_fetch` do?

Fetches source described by `SRC_URI`.

### Q10. What does `do_compile` do?

Compiles the recipe's source.

### Q11. What does `do_install` do?

Installs built files into the recipe staging destination `${D}`.

### Q12. What does `do_package` do?

Turns staged files into packages.

### Q13. What does `do_rootfs` do?

Creates the target root filesystem from selected packages.

### Q14. What does `do_image` do?

Starts image generation.

### Q15. What is sstate?

Shared-state cache used to reuse task results when inputs allow it.

### Q16. How do you inspect the final recipe environment?

``` bash
bitbake -e <recipe>
```

### Q17. How do you list tasks?

``` bash
bitbake -c listtasks <recipe>
```

### Q18. How do you see layers?

``` bash
bitbake-layers show-layers
```

### Q19. How do you find recipe providers?

``` bash
bitbake-layers show-recipes <recipe>
```

### Q20. How do you debug a failed task?

Find the recipe/task and inspect its `log.do_<task>.*` under the recipe
`WORKDIR/temp` directory.

------------------------------------------------------------------------

# 57. Common Interview Mistakes

### Mistake 1

"Yocto is an operating system."

Better:

> Yocto is a project/build ecosystem for creating customized Linux
> distributions.

### Mistake 2

"BitBake builds everything sequentially."

Better:

> BitBake resolves a dependency/task graph and executes tasks according
> to dependencies and available parallelism.

### Mistake 3

"If a recipe builds, it is automatically in the image."

False.

The resulting package must be selected by the image, packagegroup or
dependencies.

### Mistake 4

"`do_compile` creates the BMC image."

False.

It compiles a recipe. Image generation happens later.

### Mistake 5

"Always run cleansstate when something fails."

Better:

> First inspect the failing task and log; clean only when appropriate.

------------------------------------------------------------------------

# 58. Most Important Commands

``` bash
# Configure target
. setup <machine>

# Build OpenBMC
bitbake obmc-phosphor-image

# Show layers
bitbake-layers show-layers

# Show recipe providers
bitbake-layers show-recipes <recipe>

# Inspect environment
bitbake -e <recipe>

# List tasks
bitbake -c listtasks <recipe>

# Fetch only
bitbake -c fetch <recipe>

# Compile
bitbake -c compile <recipe>

# Install
bitbake -c install <recipe>

# Development shell
bitbake -c devshell <recipe>

# Clean recipe output
bitbake -c clean <recipe>
```

------------------------------------------------------------------------

# 59. Five Levels to Remember

``` text
1. LAYER
   Where metadata lives

2. RECIPE
   How software is built

3. PACKAGE
   Installable software output

4. ROOTFS
   Target filesystem made from packages

5. IMAGE
   Bootable/deployable BMC firmware artifact
```

Therefore:

``` text
Layer
  ↓
Recipe
  ↓
Tasks
  ↓
Package
  ↓
RootFS
  ↓
BMC Image
  ↓
Hardware / QEMU
```

------------------------------------------------------------------------

# 60. Final Day 28 Architecture

``` text
                    OpenBMC Source
                           |
                    +------+------+
                    |             |
                  Layers      Configuration
                    |             |
                    +------+------+
                           |
                        BitBake
                           |
                    Dependency Graph
                           |
          +----------------+----------------+
          |                |                |
        Fetch            Compile          Package
          |                |                |
          +----------------+----------------+
                           |
                        do_rootfs
                           |
                           v
                         RootFS
                           |
                        do_image
                           |
                           v
                       BMC Image
                           |
                           v
                     Flash / QEMU
                           |
                           v
                     Running BMC
```

The key chain is:

``` text
Recipe
  ↓
BitBake Tasks
  ↓
Package
  ↓
RootFS
  ↓
BMC Image
```

------------------------------------------------------------------------

# 61. Official Source Map

### OpenBMC Repository

https://github.com/openbmc/openbmc

### OpenBMC Development Documentation

https://github.com/openbmc/docs/tree/master/development

### OpenBMC --- Add New System

https://github.com/openbmc/docs/blob/master/development/add-new-system.md

### Yocto Project Documentation

https://docs.yoctoproject.org/

### Yocto Tasks Reference

https://docs.yoctoproject.org/ref-manual/tasks.html

### BitBake User Manual

https://docs.yoctoproject.org/bitbake/

### OpenBMC Phosphor Image Class

https://github.com/openbmc/meta-phosphor/blob/master/classes/obmc-phosphor-image.bbclass

------------------------------------------------------------------------

# References

\[1\] OpenBMC --- Official Repository and Build Instructions\
https://github.com/openbmc/openbmc

\[2\] BitBake User Manual --- Metadata and Tasks\
https://docs.yoctoproject.org/bitbake/bitbake-user-manual-metadata.html

\[3\] OpenBMC Development --- Add a New System\
https://github.com/openbmc/docs/blob/master/development/add-new-system.md

\[4\] Yocto Project --- Tasks Reference\
https://docs.yoctoproject.org/ref-manual/tasks.html

\[5\] Yocto Project --- Task Inspection and Work Directory\
https://docs.yoctoproject.org/dev/ref-manual/tasks.html

\[6\] OpenBMC --- Distribution/provider configuration examples\
https://github.com/openbmc/openbmc/tree/master/meta-phosphor/conf

------------------------------------------------------------------------

# End of Day 28

## From Recipe → Package → RootFS → BMC Image

``` text
             OPENBMC SOURCE
                   |
                   v
                 LAYERS
                   |
                   v
                RECIPES
                   |
                   v
                BITBAKE
                   |
                   v
                  TASKS
                   |
                   v
               PACKAGES
                   |
                   v
                ROOTFS
                   |
                   v
              BMC IMAGE
                   |
                   v
             HARDWARE / QEMU
```

**Day 28 establishes the build-system foundation required to understand
how OpenBMC services, protocols, machine configuration and
customizations become a real BMC firmware image.**
