---
title: Debug probes
---

The debug probe is the interface between pyOCD and the target, and it drives the SWD or JTAG signals that control
the target. By way of the connection between the debug probe and target, selecting the debug probe implicitly controls
which target pyOCD debugs.

There are two major flavours of debug probe:

- **On-board probes**. Many evaluation boards include an on-board debug probe, so you can plug it in and start using
    it without needing any other devices.
- **Standalone probes**. For debugging custom hardware you typically need a standalone probe that connects via an
    SWD/JTAG cable. Most commercially available debug probes, such as the SEGGER J-Link or Arm ULINKplus, are standalone.


PyOCD uses debug probe driver plug-ins to enable support for different kinds of debug probes. It comes with plug-ins for
these types of debug probes:

 Plug-in Name        | Debug Probe Type
---------------------|--------------------
`cmsisdap`           | [CMSIS-DAP](http://www.keil.com/pack/doc/CMSIS/DAP/html/index.html)
`pemicro`            | [PE Micro](https://pemicro.com/) Cyclone and Multilink
`picoprobe`          | Raspberry Pi [Picoprobe](https://github.com/raspberrypi/picoprobe)
`jlink`              | [SEGGER](https://segger.com/) [J-Link](https://www.segger.com/products/debug-trace-probes/)
`stlink`             | [STMicro](https://st.com/) [STLinkV2](https://www.st.com/en/development-tools/st-link-v2.html) and [STLinkV3](https://www.st.com/en/development-tools/stlink-v3set.html)
`remote`             | pyOCD [remote debug probe client]({% link _docs/remote_probe_access.md %})


## Unique IDs

Every debug probe has a **unique ID**. For debug probes that connect with USB, this is nominally the same as
its USB serial number. However, every debug probe plugin determines for itself what the unique ID means. Some
debug probes types are not connected with USB but are accessed across the network. In this case, the unique ID
is the probe's network address.

The unique ID parameter is actually a simple form of URL. It can be prefixed with the name of a debug probe plugin
followed by a colon, e.g. `cmsisdap:`, to restrict the type of debug probe that will match. This form is also a
requirement for certain probe types, such as the remote probe client, where the unique ID is a host address rather than
serial number.


## Auto target type identification

Certain types of on-board debug probes can report the type of the target to which they are connected.

Debug probes that support automatic target type reporting:

- CMSIS-DAP probes based on the DAPLink firmware
- STLinkV2-1 and STLinkV3


## Listing available debug probes

To view the connected probes and their unique IDs, run `pyocd list`. This command will produce output looking like this:

      #   Probe                             Unique ID
    ------------------------------------------------------------------------------------------
      0   Arm LPC55xx DAPLink CMSIS-DAP     000000803f7099a85fdf51158d5dfcaa6102ef474c504355
      1   Arm Musca-B1 [musca_b1]           500700001c16fcd400000000000000000000000097969902

For those debug probes that support automatic target type reporting, the default target type is visible in brackets
next to the probe's name. In addition, the name of the board is printed instead of the type of debug probe. This can be
seen in the example output above for the "Arm Musca-B1" board, which has a default target type of `musca_b1`.

If no target type appears in brackets, as can be seen above for the "Arm LPC55xx DAPLink CMSIS-DAP" probe (because it
is a standalone probe), it means the debug probe does not report the type of its connected target. In this
case, the target type must be manually specified either on the command line with the `-t` / `--target`
argument, or by setting the `target_override` session option (possibly in a config file).

Note that the printed list includes only those probes that pyOCD can actively query for, which currently means only USB
based probes.


## Selecting the debug probe

All of the pyOCD subcommands that communicate with a target require the user to either implicitly or explicitly
specify a debug probe.

There are three ways the debug probe is selected:

1. Implicitly, if only one probe is connected to the host, pyOCD can use it automatically without further configuration.

2. If there is more than one probe connected and pyOCD is not told which to use, it will ask on the console. It presents
    the same list of probes reported by `pyocd list`, plus this question:

        Enter the number of the debug probe or 'q' to quit>

    and waits until a probe index is entered.

3. Explicitly, with the use of `-u UID` / `--uid=UID` / `--probe=UID` command line arguments. These arguments accept
    either a whole or partial unique ID.

If no probes are currently connected and pyOCD is executed without explicitly specifying the probe to use, it will
by default print a message asking for a probe to be connected and wait. If the `-W` / `--no-wait` argument is passed,
pyOCD will exit with an error instead.



## Probe driver plug-in notes

This section contains notes on the use of different types of debug probes and the corresponding driver plug-ins.

### CMSIS-DAP

There are two major versions of CMSIS-DAP, which use different USB classes:

- v1: USB HID. This version is slower than v2. Still the most common version.
- v2: USB vendor-specific using bulk pipes. Higher performance than v1. WinUSB-enabled to allow driverless usage on Windows 8 and above. Can be used with Windows 7 only if a driver is installed with a tool such as Zadig.

These are several commercial probes using the CMSIS-DAP protocol:

- Microchip EDBG/nEDBG
- Microchip Atmel-ICE
- Cypress KitProg3
- Cypress MiniProg4
- Keil ULINKplus
- NXP LPC-LinkII
- NXP MCU-Link
- NXP MCU-Link Pro
- NXP OpenSDA

In addition, there are numerous other commercial and open source debug probes based on CMSIS-DAP.

PyOCD supports automatic target type identification for debug probes built with the
[DAPLink](https://github.com/ARMmbed/DAPLink) firmware.

#### Session options

- `cmsis_dap.deferred_transfers` (bool, default True) Whether to use deferred transfers in the CMSIS-DAP probe backend.
    By disabling deferred transfers, all writes take effect immediately. However, performance is negatively affected.
- `cmsis_dap.limit_packets` (bool, default False) Restrict CMSIS-DAP backend to using a single in-flight command at a
    time. This is useful on some systems where USB is problematic, in particular virtual machines.


### STLink

<div class="alert alert-warning">
Recent STLink firmware versions will only allow access to STM32 targets. If you are using a target
from a silicon vendor other than ST Micro, please use a different debug probe.
</div>

No host resident drivers need to be installed to use STLink probes; only libusb is required. (This may not be true for Windows 7, but has not been verified.)

The minimum supported STLink firmware version is V2J24, or any V3 version. However, upgrading to the latest version
is strongly recommended. Numerous bugs have been fixed, and new commands added for feature and performance improvements.

- V2J26: Adds 16-bit transfer support. If not supported, pyOCD will fall back to 8-bit transfers—it is possible this
    will produce unexpected behaviour if used to access Device memory (e.g. memory mapped registers).
- V2J28: Minimum version for multicore target support.
- V2J32/V3J6: Allows access to banked DP registers. Usually not needed.

[Firmware updates](https://www.st.com/en/development-tools/stsw-link007.html)

PyOCD supports automatic target type identification for on-board STLink probes that report a board ID.


### J-Link

To use a Segger J-Link probe, the driver package must be installed. Segger makes drivers available for Linux, macOS,
and Windows.

[Firmware and driver installer and updates](https://www.segger.com/downloads/jlink/)

On macOS, you can install the `segger-jlink` cask with Homebrew to get managed driver updates.

Please note that flash programming performance using a J-Link through pyOCD is currently slower than using the J-Link
software directly (or compared to CMSIS-DAP). This is because pyOCD uses the low-level DAP commands provided by J-Link,
which are inherently slower than higher level commands (which are less flexible and more difficult and complex to
integrate).

#### Session options

- `jlink.device` (str, no default)
    If this option is set to a supported J-Link device name, then the J-Link will be asked connect
    using this name. Otherwise, the J-Link is configured for only the low-level CoreSight operations
    required by pyOCD. Ordinarily, it does not need to be set.
- `jlink.power` (bool, default True)
    Enable target power when connecting via a J-Link probe, and disable power when
    disconnecting.
- `jlink.non_interactive` (bool, default True)
    Controls whether the J-Link DLL is allowed to present UI dialog boxes and its control
    panel. Note that dialog boxes will actually still be visible, but the default option
    will be chosen automatically after 5 seconds.


