# renode-nrf52

Goal is to have a Nordic nrf52 binary running in Renode - see https://github.com/renode/renode and https://renode.io/. I used the latest official Renode v1.11 on OSX and a self compiled daily version (1st December).

In this example I use:
- the S132 softdevice, source is https://www.nordicsemi.com/Software-and-tools/Software/S132
- gcc version is gcc-arm-none-eabi-9-2019-q4-major-x86_64-linux
- Nordic SDK is nRF5_SDK_15.3.0_59ac345, source is https://www.nordicsemi.com/eng/Products/Bluetooth-low-energy/nRF5-SDK
- The ble_app_blinky.elf is the output of `SDK/examples/ble_peripheral/ble_app_blinky/pca10040/s132/armgcc`
- I load the elf file and the softdevice seperate, I used the command `arm-none-eabi-objcopy -v -O binary -I ihex ./s132_nrf52_6.1.1_softdevice.hex ./s132.bin` to create the binary version of the softdevice
- Same error is visible when the pca10056 and s140 combo is used

When run in renode, it freezes when entering the SVC_Handler (see `renode-verbose-log.txt` for more details):

```
<snip>
15:55:21.4908 [INFO] cpu: Entering function app_util_critical_region_enter at 0x27A64
15:55:21.4908 [INFO] cpu: Entering function app_util_critical_region_enter at 0x27A66
15:55:21.4908 [NOISY] cpu: Allocated 64B pointer at 0x140257097617280.
15:55:21.4909 [NOISY] cpu: Allocated is now 98.62MiB.
15:55:21.4909 [NOISY] cpu: Deallocated a 64B pointer at 0x140257097617280.
15:55:21.4909 [INFO] cpu: Entering function nrf_sdh_enable_request at 0x2B5B4
15:55:21.4909 [NOISY] cpu: Allocated 64B pointer at 0x140257097617280.
15:55:21.4909 [NOISY] cpu: Allocated is now 98.62MiB.
15:55:21.4909 [NOISY] cpu: Deallocated a 64B pointer at 0x140257097617280.
15:55:21.4910 [INFO] cpu: Entering function sd_softdevice_enable (entry) at 0x2B540
15:55:21.4910 [NOISY] nvic: Internal IRQ 11.
15:55:21.4910 [NOISY] cpu: IRQ 0, value True
15:55:21.4910 [NOISY] cpu: Setting CPU IRQ #0 to True
15:55:21.4910 [NOISY] nvic: Acknowledged IRQ 11.
15:55:21.4911 [NOISY] cpu: IRQ 0, value False
15:55:21.4911 [NOISY] cpu: Setting CPU IRQ #0 to False
15:55:21.4911 [NOISY] cpu: Allocated 64B pointer at 0x140257097617280.
15:55:21.4911 [NOISY] cpu: Allocated is now 98.62MiB.
15:55:21.4911 [NOISY] cpu: Loop to itself detected
15:55:21.4912 [NOISY] cpu: Deallocated a 64B pointer at 0x140257097617280.
15:55:21.4912 [INFO] cpu: Entering function SVC_Handler (entry) at 0x262E6
```

Using gdb the backtrace looks like:

```
(gdb) target remote :3333
Remote debugging using :3333
SVC_Handler () at ../../../../../../modules/nrfx/mdk/gcc_startup_nrf52.S:328
328	../../../../../../modules/nrfx/mdk/gcc_startup_nrf52.S: No such file or directory.
(gdb) bt
#0  SVC_Handler () at ../../../../../../modules/nrfx/mdk/gcc_startup_nrf52.S:328
#1  <signal handler called>
#2  0x0002b596 in sd_softdevice_enable (p_clock_lf_cfg=p_clock_lf_cfg@entry=0x2000ff44, fault_handler=0x27351 <app_error_fault_handler>) at ../../../../../../components/softdevice/s132/headers/nrf_sdm.h:322
#3  0x0002b610 in nrf_sdh_enable_request () at ../../../../../../components/softdevice/common/nrf_sdh.c:214
#4  0x0002a2c8 in ble_stack_init () at ../../../main.c:573
#5  main () at ../../../main.c:573
```

Changes to the default Nordic example config is that the project use the internal RC instead the XTAL, however there is no difference when I run the application in Renode.

The renode configuration file looks like this:

```
using sysbus

mach create
machine LoadPlatformDescription @platforms/cpus/nrf52840.repl

$elf?=@ble_app_blinky.elf
$softdevice?=@s132.bin

logLevel -1
showAnalyzer uart0
showAnalyzer uart1
sysbus LogAllPeripheralsAccess true
cpu LogFunctionNames true
verboseMode true

sysbus Tag <0xF0000FE0 1> "Nodic-is_manual_peripheral_setup_needed" 0x0

macro reset
"""
    sysbus LoadBinary $softdevice 0x000
    sysbus LoadELF $elf
"""
runMacro $reset

#machine StartGdbServer 3333
start

```

