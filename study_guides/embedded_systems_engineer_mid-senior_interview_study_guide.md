
A deep-dive study guide oriented around real-world scenarios. Questions are pitched at the level of someone who has shipped embedded products and is being evaluated for a role spanning firmware, embedded Linux, PCB design, and applied ML/CV.

---

# Table of Contents

1. [C/C++ for Embedded](#1-cc-for-embedded)
2. [Embedded Linux, Device Tree, Drivers, U-Boot](#2-embedded-linux-device-tree-drivers-u-boot)
3. [Yocto and Buildroot](#3-yocto-and-buildroot)
4. [Build Systems: Make, CMake, Cross-Compilation](#4-build-systems-make-cmake-cross-compilation)
5. [Qt and QML](#5-qt-and-qml)
6. [RTOS, Interrupts, State Machines, Embedded Architecture](#6-rtos-interrupts-state-machines-embedded-architecture)
7. [Communication Protocols](#7-communication-protocols)
8. [Schematic Capture and PCB Design](#8-schematic-capture-and-pcb-design)
9. [Lab, Troubleshooting, Validation](#9-lab-troubleshooting-validation)
10. [Computer Vision, ML, and Embedded Deployment](#10-computer-vision-ml-and-embedded-deployment)
11. [MCU Platforms and Toolchains](#11-mcu-platforms-and-toolchains)
12. [IoT-Adjacent Tech (Go, Python, JS/TS, SQL, Node.js)](#12-iot-adjacent-tech)

---

## 1. C/C++ for Embedded

### Q1.1 — Explain `volatile`, what it actually guarantees, and why it is *not* sufficient for sharing data between an ISR and main code.

`volatile` tells the compiler that a variable's value can change outside the current thread of execution, so the compiler must not optimize away reads/writes (e.g., it can't cache the value in a register or coalesce multiple reads). It guarantees that each access in the source corresponds to a real memory access in the generated code, in source order *with respect to other volatile accesses*.

What `volatile` does **not** do:
- It does not provide atomicity. A 32-bit write on an 8-bit MCU is multiple instructions.
- It does not provide a memory barrier. The CPU and compiler can still reorder non-volatile accesses around it. On a Cortex-M with a write buffer, the store may not be visible to a DMA engine without a `DMB`/`DSB`.
- It does not synchronize with other cores or DMA masters.

For ISR-to-main sharing on a single-core MCU, you typically need: (1) `volatile` on the shared variable so reads aren't elided, (2) atomic access (either a single-instruction-sized type or critical sections via `__disable_irq()`/`__enable_irq()` or `cli`/`sei`), and (3) ordering — usually a memory barrier if you're touching peripheral memory or have separate buffers. In C11/C++11, `_Atomic`/`std::atomic` is the right answer; it gives you atomicity and ordering, and on most embedded compilers correctly emits the right barrier instructions for the architecture.

### Q1.2 — A junior engineer hands you a ring buffer for UART RX where the ISR writes and a task reads. They used `head` and `tail` as `volatile uint16_t`. The buffer occasionally drops bytes under load. Walk through what's wrong and how to fix it.

The classic SPSC ring buffer must satisfy two invariants: the producer (ISR) only writes `head`, the consumer only writes `tail`, and updates to the buffer storage must be visible *before* the corresponding index update is visible. `volatile` doesn't enforce that ordering on architectures with store buffers.

Failure modes I'd suspect:
1. **Compiler reordering of non-volatile vs. volatile.** `buffer[head] = byte;` is a non-volatile write; `head = next;` is volatile. The compiler can't reorder *across* the volatile, but on Cortex-M7 with its store buffer, the consumer can still observe the index update before the data write retires. Fix: `__DMB()` between data write and index update on the producer side.
2. **Read tearing on the index.** If `head`/`tail` were `uint32_t` on an AVR, the consumer might read a half-updated value. `uint16_t` on a 32-bit Cortex-M is fine but worth confirming.
3. **Wraparound logic.** Using `head == tail` for both empty and full requires sacrificing one slot. A bug I've seen: using `count` instead, modified by both sides without atomics — race.
4. **Buffer size not power of two** combined with modulo on a slow MCU — not a correctness issue but worth flagging.

The clean fix: switch to `std::atomic<uint16_t>` (or `_Atomic` in C11) with `memory_order_release` on the producer's index store and `memory_order_acquire` on the consumer's index load. Drop `volatile` on the buffer storage. This is portable and the compiler emits the correct barriers.

### Q1.3 — What's the difference between `const`, `constexpr`, and `consteval` in modern C++? When does each matter on an MCU with limited flash?

- `const` means "I won't modify this through this binding." A `const int x = read_adc();` is perfectly legal — `x` is initialized at runtime and can live in RAM.
- `constexpr` means "this can be evaluated at compile time *if used in a constant context*." A `constexpr` variable must be initialized with a constant expression; a `constexpr` function can be called at runtime or compile time. On an MCU, `constexpr` lookup tables go to flash (`.rodata`), saving RAM. Critical for things like CRC tables, lookup-based math (sin tables), or bitfield masks.
- `consteval` (C++20) means "must be evaluated at compile time." Useful for forcing compile-time computation of something expensive — e.g., a `consteval` function that builds a CAN ID filter mask must run at compile time, so you'll get a compile error if you accidentally invoke it with runtime data.

Practical impact: on an STM32F0 with 32KB flash and 8KB RAM, moving a 1KB CRC32 table from `const` (which on some compilers/setups still lands in RAM) to `constexpr` (which is reliably in `.rodata`) can be the difference between fitting and not. Always check the map file.

### Q1.4 — Explain the C++ "Rule of Zero / Three / Five" and why it matters for embedded code that wraps a hardware resource.

If your class manages a resource (a DMA channel, a file descriptor, a peripheral lock), you have to think about copy semantics. The Rule of Three (C++98): if you define any of destructor, copy constructor, or copy assignment, you probably need all three. The Rule of Five (C++11): add move constructor and move assignment. The Rule of Zero: design so that you don't manage resources directly — use RAII types like `std::unique_ptr` — and the compiler-generated specials are correct.

For embedded RAII wrappers around peripherals, the typical pattern is non-copyable, optionally movable:

```cpp
class SpiTransaction {
public:
    explicit SpiTransaction(Spi& bus) : bus_(bus) { bus_.lock(); }
    ~SpiTransaction() { bus_.unlock(); }
    SpiTransaction(const SpiTransaction&) = delete;
    SpiTransaction& operator=(const SpiTransaction&) = delete;
    SpiTransaction(SpiTransaction&&) = default;
    SpiTransaction& operator=(SpiTransaction&&) = default;
private:
    Spi& bus_;
};
```

Copying would mean two objects believing they own the lock — double unlock on destruction. Default-everything would silently introduce that bug. The Rule of Zero / Five forces you to think about it.

### Q1.5 — Why might `new` and `delete` be problematic in firmware, and what alternatives do you reach for?

The objections to dynamic allocation in firmware:
1. **Heap fragmentation.** Long-running embedded systems with mixed allocation sizes will fragment until allocation fails despite enough total free memory. There's no compaction.
2. **Non-deterministic timing.** `malloc` walks a free list; worst-case time depends on heap state. Hard real-time systems can't accept that.
3. **Failure handling.** Most firmware has nowhere to go if allocation fails — there's no "swap." Bugs become hard reboots.
4. **Code size.** Pulling in `new`/`delete`/exceptions can add tens of KB.

Alternatives:
- **Static allocation.** All objects live in `.bss` or `.data`, sized at compile time.
- **Object pools.** Pre-allocated arrays of fixed-size objects with a free list. O(1) allocation/free, no fragmentation.
- **Stack allocation with placement new.** Allocate a buffer, construct objects in place. Useful for bounded-lifetime objects.
- **Arena / bump allocators.** Allocate from a region, free the whole region at once. Good for per-frame or per-request lifetimes.
- **`etl::` (Embedded Template Library)** — STL-like containers with fixed capacity, no heap.

If you must use the heap, restrict it to startup-only allocation, then disable further allocation (some RTOSes support this).

### Q1.6 — Describe the C startup sequence from reset vector to `main()` on a Cortex-M. What does the linker script need to provide?

On reset, the Cortex-M loads the initial SP from address 0x0 and the reset handler address from 0x4. The reset handler (often `Reset_Handler` in `startup_*.s`) does:

1. Optionally configure the FPU (CP10/CP11 in CPACR).
2. Copy `.data` from flash LMA to RAM VMA. The linker script defines `_sidata`, `_sdata`, `_edata`.
3. Zero `.bss`. Linker provides `_sbss`, `_ebss`.
4. Call `__libc_init_array` (C++ static constructors, registered via `.init_array`).
5. Call `SystemInit()` (vendor clock setup) — sometimes done before .data copy.
6. Call `main()`.
7. If `main()` returns, loop forever (or call `_exit`).

The linker script provides:
- Memory regions (FLASH, RAM, sometimes CCMRAM, BKPSRAM).
- Section assignments: `.isr_vector` at flash origin, `.text`/`.rodata` in flash, `.data` with `AT>FLASH` so it has a load address in flash and a virtual address in RAM, `.bss` in RAM, `.heap` and `.stack` typically at the end of RAM growing toward each other.
- The symbols startup code uses (`_sidata`, `_sdata`, etc.).

Common interview gotcha: explaining why `static int x = 5;` ends up in flash *and* RAM (LMA in flash, VMA in RAM, copied at startup), while `static int x;` only consumes RAM (zeroed in `.bss`).

### Q1.7 — What is undefined behavior, and why is it especially dangerous in embedded code? Give three concrete examples.

UB means the standard imposes no requirements. The compiler can assume UB doesn't happen and optimize accordingly — leading to "the code worked at -O0 and broke at -O2."

Three embedded-flavored examples:
1. **Signed integer overflow.** `for (int i = 0; i <= INT_MAX; i++)` is a UB-driven infinite loop after the compiler concludes overflow can't happen and removes the bound check. Use unsigned for counters where wraparound is intended.
2. **Strict aliasing violation.** Casting a `uint32_t*` to a `float*` to reinterpret bits. The compiler may assume they don't alias, reorder loads/stores, and produce wrong results. Use `memcpy` for type punning, or `std::bit_cast` in C++20.
3. **Reading uninitialized variables.** On a Cortex-M, an uninitialized stack variable is whatever was on the stack — possibly secrets, possibly a valid-looking pointer that crashes much later. The compiler may assume any value, including ones that make conditional branches "unreachable."

Other firmware-specific traps: dereferencing peripheral pointers without `volatile` (compiler hoists or removes accesses), passing non-NUL-terminated strings to `strlen`, signed shift of a negative number.

### Q1.8 — Explain function pointers vs. virtual functions. When would you use each in firmware, and what's the cost?

A function pointer is a single machine word holding an address; calling it is one indirect branch. A virtual function call is a vtable lookup: load object → load vptr → load slot → indirect branch. Same number of cycles in practice (both are indirect calls), but virtual adds: one pointer of overhead per object (vptr) and one vtable per class in flash.

Use function pointers for:
- C interfaces (driver dispatch tables — Linux's `file_operations` is the canonical example).
- Lightweight callbacks where you don't need polymorphism.
- ISR handlers in jump tables.

Use virtual functions when:
- You have a real "is-a" relationship and multiple implementations.
- You want RAII and type safety.
- You're building plugin-like architectures (sensor drivers behind a common interface).

The hidden cost of virtual: it inhibits inlining across the call. For a hot loop, devirtualizing via templates (CRTP) can give 2-5x speedup. For a 1Hz state machine, it doesn't matter.

### Q1.9 — What does `-fno-exceptions -fno-rtti` buy you, and what do you give up?

You give up: `try`/`catch`, `dynamic_cast`, `typeid`. The runtime support for exceptions adds 30-100KB of code (unwinding tables, personality routines). RTTI adds per-class type info in flash.

You buy: smaller binary, deterministic control flow (exceptions can throw from anywhere, which complicates real-time analysis), and no surprise heap allocation from exception objects.

What you give up that matters: STL throws on errors. `vector::at()` throws on out-of-range. With `-fno-exceptions`, throwing aborts. So you either avoid throwing STL operations or use `etl::` / your own containers. Some libraries (Boost components, modern C++ patterns like `std::optional` returns instead of throws) work fine without exceptions.

In automotive/industrial firmware, `-fno-exceptions -fno-rtti` is standard. Most MISRA-aligned codebases require it.

### Q1.10 — Walk through a stack overflow on an MCU. How do you detect it, prevent it, and diagnose it after the fact?

Detection at runtime:
- **Stack painting.** Fill stack with a known pattern (`0xA5`) at startup. Periodically scan for the highest address still painted to compute high-water mark.
- **MPU guard region.** Place an unmapped or read-only region just past the stack bottom. Overflow triggers a MemManage fault — clean and immediate.
- **Cortex-M MSPLIM/PSPLIM** (Armv8-M). Hardware stack limit registers; CPU faults on overflow.
- **RTOS hooks.** FreeRTOS `configCHECK_FOR_STACK_OVERFLOW` does pattern checks and high-water marks per task.

Prevention:
- Avoid recursion or bound it.
- Avoid large stack-allocated arrays (`uint8_t buf[4096]` on a task with 1KB stack = obvious crash).
- Use `-fstack-usage` to get per-function stack usage at compile time; combine into a call-graph analysis tool to compute worst-case stack depth.
- Avoid VLAs (variable-length arrays).

Postmortem: on fault, dump the SP, the fault registers (CFSR, HFSR, MMFAR), and ideally a stack trace by walking back through saved frames. Tools like Segger Ozone or a CrashCatcher-style fault handler help.

### Q1.11 — Compare RAII for resource management vs. C-style cleanup. Give a concrete embedded scenario.

C-style cleanup uses `goto cleanup:` or nested if-checks, manually releasing resources on each path. It's correct but error-prone — every new error path needs every cleanup added.

RAII ties resource lifetime to object scope: destructors run automatically on scope exit, including via exceptions or early returns.

Embedded scenario: a function that takes a mutex, configures DMA, transfers data, releases DMA, releases mutex. C version:

```c
int transfer(...) {
    if (mutex_lock(&m) != 0) return -1;
    if (dma_acquire(&ch) != 0) { mutex_unlock(&m); return -2; }
    if (dma_transfer(&ch, ...) != 0) { dma_release(&ch); mutex_unlock(&m); return -3; }
    dma_release(&ch);
    mutex_unlock(&m);
    return 0;
}
```

Add a fourth resource and you've got eight cleanup paths. C++ version: `MutexLock lk(m); DmaChannel ch; ch.transfer(...);` — destructors handle the rest, in reverse order of construction. The C++ compiler enforces correctness; the C version relies on programmer discipline.

### Q1.12 — What's the difference between `static` storage duration, `thread_local`, and stack/automatic? Where does each live in memory on a typical MCU?

- **Automatic (stack).** Function-local variables. Live in the active task's stack region in RAM. Allocated on function entry, deallocated on return.
- **Static storage.** `static` locals, file-scope variables, globals. Initialized variables go to `.data` (RAM, copied from flash at startup). Zero-initialized go to `.bss` (RAM, zeroed at startup). Lifetime is the entire program.
- **`thread_local`.** Per-thread storage. On bare-metal MCUs without thread support, often unsupported or maps to `static`. On RTOS, requires runtime support — FreeRTOS doesn't natively provide it; some toolchains hook through TLS sections per task. Avoid in firmware unless you've verified the support.
- **Heap.** `malloc`/`new`. Lives in a heap region defined by the linker script.

On Cortex-M, the typical layout is: flash holds `.text`, `.rodata`, and the LMA of `.data`. RAM holds `.data`, `.bss`, heap (growing up), and stack (growing down). CCMRAM if present often holds hot variables for speed.

### Q1.13 — Describe a real production C++ codebase pitfall around static initialization order.

The "static initialization order fiasco": objects with static storage duration in different translation units have an unspecified relative initialization order. If a static object's constructor uses another static object from a different TU, it may run before the dependency is initialized.

Embedded scenario: a `Logger` static instance and a `ConfigManager` static instance, where `Logger`'s constructor calls `ConfigManager::getLogLevel()`. If `Logger` initializes first, you read uninitialized memory.

Fixes:
1. **Construct on first use idiom.** Wrap the static in a function returning a reference to a function-local static — guaranteed initialized on first call (and thread-safe in C++11+).
2. **Avoid non-trivial static constructors.** In firmware, this is often a hard rule: no global objects with constructors that do real work. Use `init()` methods called explicitly from `main()`.
3. **Use `constinit`** (C++20) to require compile-time initialization, eliminating the order question.

In MISRA C++ and most safety-critical guidelines, dynamic initialization of statics is forbidden for exactly this reason.

---

## 2. Embedded Linux, Device Tree, Drivers, U-Boot

### Q2.1 — Walk through the boot sequence from power-on to userspace `init` on a typical ARM SoC (e.g., i.MX or AM335x).

1. **BootROM.** Mask-ROM in the SoC runs first. It reads boot mode pins or eFuses to decide where to load the next stage from (eMMC, SD, SPI flash, NAND, USB, network).
2. **First-stage bootloader (SPL/MLO).** A small loader (U-Boot SPL, on TI parts called MLO). It runs from SoC SRAM because DDR isn't initialized yet. Its job: bring up DDR (using vendor-specific calibration), set up basic clocks, then load the second stage into DDR.
3. **Second-stage bootloader (U-Boot proper).** Full U-Boot. Initializes more peripherals (Ethernet, USB, more storage), reads its environment, runs the boot script. Loads kernel, device tree blob, and (optionally) initramfs.
4. **Kernel handoff.** U-Boot jumps to the kernel entry point with registers set per the ARM boot protocol: r0=0, r1=machine ID (legacy) or unused, r2=pointer to ATAGs or DTB.
5. **Kernel boot.** Decompresses (if zImage), sets up the MMU, parses the DTB, brings up drivers via the device tree, mounts the rootfs.
6. **Init.** Kernel execs `/sbin/init` (or whatever's specified by `init=` on the cmdline). Could be systemd, BusyBox init, or a custom binary. Userspace takes over.

On secure boot systems, add: BootROM verifies SPL signature, SPL verifies U-Boot, U-Boot verifies kernel/dtb (FIT image with signed hash). Each stage authenticates the next.

### Q2.2 — What is the device tree, why does it exist, and what's the difference between `.dts`, `.dtsi`, and `.dtb`?

The device tree is a hierarchical data structure describing hardware that isn't discoverable (i.e., not PCI/USB). Pre-DT, ARM kernels had thousands of board files — C code with hardcoded register addresses, IRQ numbers, and platform_device registrations. Every new board needed a kernel patch. The DT moves this hardware description out of the kernel into a separate file the bootloader passes in.

- **`.dts`** — Device Tree Source. Human-readable, board-specific. One per board.
- **`.dtsi`** — Device Tree Source Include. Reusable fragments — typically the SoC's common peripherals — included by `.dts` files. `imx6q.dtsi` describes everything on an i.MX6 Quad; `imx6q-sabresd.dts` includes it and adds board-specific bits.
- **`.dtb`** — Device Tree Blob. Compiled binary form (`dtc` is the compiler). Loaded by the bootloader and passed to the kernel.
- **`.dtbo`** — Overlay. A patch applied at runtime to enable or modify nodes (e.g., enabling an SPI peripheral and binding a driver to a sensor).

The kernel has driver-side bindings: each driver declares which `compatible` strings it supports; the kernel walks the DT and instantiates a `platform_device` for each matched node, calling the driver's `probe()` with the device's resources from the DT.

### Q2.3 — You're bringing up a new I2C sensor on an existing i.MX6 board. Walk through every step from "I have the datasheet" to "userspace can read sensor data."

1. **Schematic check.** Confirm which I2C bus the sensor is on (e.g., I2C2), pull-up values, address pins, interrupt pin if any. Confirm the I2C bus is enabled in the existing DT or needs to be enabled.
2. **DT entry.** Add a child node under the I2C controller node:
   ```
   &i2c2 {
       status = "okay";
       clock-frequency = <400000>;
       my_sensor: temp_sensor@48 {
           compatible = "ti,tmp102";
           reg = <0x48>;
           interrupt-parent = <&gpio4>;
           interrupts = <12 IRQ_TYPE_EDGE_FALLING>;
       };
   };
   ```
3. **Pinmux.** Make sure the I2C pins are muxed correctly via `pinctrl` properties. Often defined in the SoC dtsi but sometimes the board has a conflict.
4. **Kernel driver.** Check whether a driver exists upstream. For TMP102, `drivers/hwmon/tmp102.c` exists. Enable it in `make menuconfig` (`CONFIG_SENSORS_TMP102=m` or `=y`). Rebuild kernel/modules.
5. **Boot and verify.** Check `dmesg` for probe messages. `i2cdetect -y 1` to confirm the device responds (note bus numbering can shift). `ls /sys/class/hwmon/` for the new entry. `cat /sys/class/hwmon/hwmon0/temp1_input` for the reading.
6. **Troubleshoot if no probe.** Check `compatible` matches the driver's `of_match_table`. Check `dmesg` for I2C errors (NACK = wrong address or no power). Scope the bus to verify pull-ups and signal integrity.
7. **Userspace integration.** Either read sysfs directly or use a library (libsensors, libiio if it's an IIO driver).

If no upstream driver exists, you write one — typically based on a similar driver, registering a `i2c_driver` with a probe function, doing the register reads in your sysfs/IIO callbacks.

### Q2.4 — Explain the Linux driver model: how do `platform_driver`, `i2c_driver`, and `spi_driver` differ, and what does `probe()` typically do?

The Linux device model has three core concepts: bus, device, driver. A bus type defines how devices and drivers on it are matched. Common buses for embedded:

- **Platform bus.** A pseudo-bus for non-discoverable, memory-mapped peripherals on the SoC (UARTs, GPIOs, timers). Devices come from the device tree or board files. Match by `compatible` string.
- **I2C bus.** Real I2C devices. Match by `compatible` (DT) or device ID table (legacy).
- **SPI bus.** Same model as I2C.
- **USB, PCI.** Discoverable buses with their own enumeration.

For each, you register a driver struct (`platform_driver`, `i2c_driver`, `spi_driver`) with at least:
- A `name` and `of_match_table` (or `id_table`).
- `probe()` — called when a matching device appears.
- `remove()` — called on driver/device unbind.
- Optional `suspend`/`resume`.

`probe()` typically:
1. `devm_kzalloc()` a private context struct.
2. Read DT properties via `of_property_read_*` or `device_property_*`.
3. Acquire resources: `devm_ioremap_resource()` for MMIO, `devm_request_irq()` for interrupts, `devm_clk_get()` for clocks, `devm_regulator_get()` for power, `devm_gpiod_get()` for GPIOs.
4. Initialize hardware (reset, configure registers).
5. Register with subsystems: `input_register_device()`, `iio_device_register()`, `cdev_add()`, etc.
6. Return 0 on success or `-EPROBE_DEFER` if a dependency (e.g., regulator) isn't ready yet — kernel will retry later.

The `devm_*` family is critical: resources are auto-freed on probe failure or remove, eliminating most cleanup-path bugs.

### Q2.5 — What is the difference between a character device, a block device, and a misc device? When would you use each?

- **Character device.** Stream-oriented. Read/write byte-by-byte (or in arbitrary chunks). Most embedded peripherals: UART, GPIO, sensors that don't fit IIO, custom hardware. Identified by major/minor number; created via `cdev_init`/`cdev_add` and `device_create`.
- **Block device.** Block-oriented (typically 512-byte or 4KB blocks), with a request queue, supports random access, hosts filesystems. eMMC, SSD, SD card, ramdisk. Heavier infrastructure (gendisk, request queue, IO scheduler).
- **Misc device.** A simplified char device sharing major number 10. Use when you want a single device node and don't need a custom major. Register with `misc_register()` — much shorter than full cdev setup.

For most simple custom firmware-facing drivers, misc is the right answer. Use full cdev when you need multiple minor numbers (e.g., one per channel of a multi-channel device). Block is for storage.

In modern kernel idioms, prefer existing subsystems first: IIO for sensors, input for HID-like, hwmon for temps/voltages, GPIO/pwm/leds for those, v4l2 for cameras. Only fall back to a raw char device when nothing fits.

### Q2.6 — A user reports that their Linux board hangs on boot intermittently, after the kernel banner but before login. How do you debug this?

Systematic narrowing:

1. **Get more output.** Boot with `earlycon` and `console=ttymxc0,115200`. Add `loglevel=8` and `ignore_loglevel`. If nothing extra appears, the hang is hardware or very early.
2. **Identify last printk.** What's the last message? `Freeing unused kernel memory` means kernel is done; hang is in userspace init. `Mounted root` means rootfs is up; problem in init. If we hang in driver init, the message just before tells us which driver.
3. **Userspace init hangs.** Boot with `init=/bin/sh` to drop to a shell before init runs. Then start services manually to find the offender.
4. **Driver hang.** `initcall_debug` in cmdline prints each driver's init/probe with timing. The one without a "returned" message is the culprit.
5. **Hardware issues.** Intermittent suggests power, clock, or signal integrity. Check Vcc rails on a scope at boot. Check brownout. Check thermal — does it correlate with cold/warm boot?
6. **Reproducibility.** Run a boot loop with serial logging. Capture 50 boots; analyze timings and last messages.
7. **JTAG.** If you have it, halt the CPU when it hangs and read the PC. You'll see exactly where it's stuck — usually a spinning wait on a register that never changes.

In real life, common causes of "hangs after kernel banner": eMMC enumeration failure (timing/voltage), watchdog that's not being kicked because a service hung, a `udev` rule blocking on a missing device, or a misbehaving driver in a probe loop.

### Q2.7 — Explain `of_property_read_u32`, `gpiod_get`, and `regmap`. Why do these abstractions matter?

These are kernel APIs that decouple drivers from how resources are described.

- **`of_property_read_u32(np, name, &val)`** — reads a u32 from a DT node. The driver doesn't care that the value came from DT; could be ACPI on x86 (`device_property_read_u32` is the bus-agnostic wrapper). Drivers using `device_property_*` work on both.
- **`gpiod_get(dev, "reset", flags)`** — gets a GPIO descriptor by name. The DT says `reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;`. The driver calls `gpiod_set_value(reset, 1)` and the framework handles polarity and which controller. Replaced the older numeric `gpio_request` API which required global GPIO numbers — error-prone across SoCs.
- **`regmap`** — abstracts register access over different buses. A sensor might be available in I2C, SPI, or memory-mapped variants. With regmap, the driver writes `regmap_write(map, REG_CTRL, val)` and the framework dispatches. Also handles caching (avoids re-reading static config registers), volatile/non-volatile distinction, and atomic read-modify-write.

These abstractions matter because they let a single driver support multiple hardware variants and platforms, dramatically reducing code duplication. They also enforce correctness — `gpiod` knows which lines are output-only; regmap can validate register ranges.

### Q2.8 — What's the difference between sysfs, procfs, debugfs, and configfs? When would you expose a driver feature via each?

- **sysfs (`/sys`).** Stable ABI. One value per file. For attributes that are fundamental properties of the device — `temp1_input`, `fan1_speed`. Backwards-compat is enforced; once a sysfs attribute ships, you can't remove or change its semantics without breakage. Use for production-facing knobs.
- **procfs (`/proc`).** Originally for process info; now also has kernel info (`/proc/cpuinfo`, `/proc/interrupts`). New driver-specific stuff should *not* go here — sysfs is preferred. Procfs files often have multi-value formatted output, which is harder to parse stably.
- **debugfs (`/sys/kernel/debug`).** Unstable ABI, debug-only. Mount-controlled. Use for "I want to dump driver state during development" or "let me poke this register from userspace to test." Not for production interfaces.
- **configfs (`/sys/kernel/config`).** Userspace creates kernel objects by `mkdir`. Used by USB gadget, target/iSCSI, NVMe target — places where userspace defines new instances of something rather than configuring existing ones.

For a typical sensor driver: production attributes via IIO/hwmon (which use sysfs). Debug knobs via debugfs. Don't use procfs for new drivers.

### Q2.9 — Explain U-Boot's role beyond just "load the kernel." What features matter on a production embedded device?

Beyond the basic boot:

1. **Boot environment.** Stored in flash (separate region or in the boot partition). Holds boot commands, kernel arguments, network config. Persistent across reboots, modifiable from U-Boot prompt or Linux (`fw_setenv`).
2. **Boot scripts.** `boot.scr` is a U-Boot script that picks the right kernel/dtb based on env vars or hardware presence. Lets you ship a single image that boots on multiple board variants.
3. **Network boot.** TFTP and NFS boot are essential for development — flash-and-reboot cycles are slow, network is fast. PXE boot for production deployment.
4. **A/B updates / fallback.** U-Boot can implement a boot-counter scheme: try slot A, if it fails N times (counter not reset by userspace), boot slot B. Critical for over-the-air update reliability — bricking a fielded device means a truck roll.
5. **Secure boot / verified boot.** U-Boot verifies the kernel's signature against a key burned in eFuse / OTP before jumping. Essential for any device that handles sensitive data.
6. **FIT images.** Flattened Image Tree: kernel + dtb + initramfs + signatures in one file with metadata. Lets you ship multiple kernel variants and pick at boot, sign atomically.
7. **DFU / fastboot.** Recovery modes for re-flashing without external tools.
8. **SPL.** First-stage loader for chips where the BootROM can't load full U-Boot directly (DDR not init).

For production, the U-Boot decisions you'll actually fight over: image layout (slots, env), update strategy, recovery path, secure boot key management.

### Q2.10 — What's the difference between an initramfs and a real rootfs? Why have an initramfs?

An initramfs is a CPIO archive embedded in the kernel image (or loaded separately) that's unpacked into a tmpfs and mounted as `/`. It contains a minimal userspace whose job is to mount the *real* rootfs and `switch_root` into it.

Why bother:
1. **Modules for the real rootfs's storage.** Your eMMC or NVMe driver might be a module. The kernel can't mount the rootfs from a device whose driver isn't loaded yet. Initramfs holds the module and an init script that loads it.
2. **Encryption / unlock.** Decrypt the rootfs (LUKS) before mounting. Need userspace to prompt for keys or query a TPM.
3. **Network root.** NFS root requires bringing up the network first — that's userspace work.
4. **Recovery / dual-boot logic.** Decide which slot to boot, run integrity checks.

For a simple embedded device with eMMC and a built-in driver, you can skip initramfs entirely and have the kernel mount eMMC directly. This is faster to boot and simpler. Initramfs is overhead unless you need its features.

A subtle distinction: **initrd** is a legacy block-device-based ramdisk (the kernel mounts it as a block device); **initramfs** is the modern tmpfs-based approach. Always use initramfs.

### Q2.11 — How does the Linux kernel handle interrupts on ARM? Walk through from IRQ line to handler.

1. **Hardware.** Peripheral asserts an interrupt line. Goes to the interrupt controller (GIC on ARMv7-A/v8-A; NVIC on Cortex-M, but Cortex-M usually doesn't run Linux).
2. **GIC distributes.** GIC routes the interrupt to a CPU based on affinity. CPU receives an IRQ exception.
3. **Vector entry.** ARM jumps to the IRQ vector. Kernel saves context, switches to IRQ stack.
4. **Generic IRQ layer.** Reads the GIC's IAR register to identify which IRQ fired, looks up the IRQ descriptor (`struct irq_desc`).
5. **Flow handler.** Calls the flow handler — typically `handle_fasteoi_irq` for level-triggered or `handle_edge_irq` for edge.
6. **Driver handler.** Flow handler invokes the registered handler (`request_irq` callback). Handler runs with IRQs disabled (or not, depending on flags).
7. **Top half ack.** Handler acks the device, possibly schedules a bottom half (tasklet, workqueue, threaded IRQ), returns `IRQ_HANDLED`.
8. **EOI.** Generic IRQ layer sends End-of-Interrupt to GIC.
9. **Bottom half.** Tasklet runs in softirq context (still atomic, can't sleep). Workqueue runs in process context (can sleep). Threaded IRQ (`request_threaded_irq`) runs the bulk of handler code in a kernel thread — can sleep, preferred for most modern drivers because it plays nicely with PREEMPT_RT.

On preempt-rt kernels, even non-threaded IRQs run as threads by default (forced threading), trading a bit of latency for preemptibility.

### Q2.12 — Compare `printk()`, `dev_dbg()`, `pr_info()`, and `trace_*`. When do you use each in driver development?

- **`printk(KERN_INFO ...)`** — the original. Direct kernel log. Always emits.
- **`pr_info()`, `pr_err()`, `pr_debug()`** — wrappers around printk with module-name prefix (set via `pr_fmt`). Cleaner.
- **`dev_info(dev, ...)`, `dev_dbg(dev, ...)`** — same but with the device name prefix. Always prefer these in drivers — output tells you which instance, which is invaluable when you have two of the same chip.
- **`pr_debug` / `dev_dbg`** — compiled out unless `DEBUG` is defined for that file or `dynamic_debug` is enabled. With dynamic debug, you can turn them on per-file or per-line at runtime via `/sys/kernel/debug/dynamic_debug/control`. Production-safe.
- **`trace_*` (tracepoints)** — static probe points emitted into a ring buffer, read via ftrace/perf. Designed for high-volume, low-overhead instrumentation. Used for performance analysis: every IRQ, every scheduling event, every block I/O. Don't replace logs but complement them.

For a driver: `dev_info` for one-shot lifecycle events (probe success, link up). `dev_err` for errors. `dev_dbg` for verbose debug. Tracepoints for hot paths you want visibility into.

### Q2.13 — What's PREEMPT_RT, what does it change, and when do you need it?

PREEMPT_RT is the real-time patchset (now mostly merged) that turns Linux into a hard-real-time-capable kernel. Key changes:

1. **Spinlocks become sleeping mutexes** (rtmutex). This is the big one — most spinlocks in the kernel become preemptible. Reduces worst-case latency dramatically.
2. **IRQ handlers become threads.** Every IRQ runs in a kernel thread by default, schedulable with priorities.
3. **Priority inheritance.** RT mutexes implement PI to prevent unbounded priority inversion.
4. **High-resolution timers replace tick-based.** Already mainstream now.

Result: worst-case scheduling latencies in low microseconds (on capable hardware) instead of milliseconds.

When you need it: motion control, audio with tight buffer constraints, industrial protocols with hard deadlines (EtherCAT distributed clocks, some PROFINET classes), motor control loops in software.

When you don't: typical IoT, HMI, network gateway, video streaming. Soft real-time is fine and the overhead of PREEMPT_RT (some throughput loss, more complex driver requirements) isn't worth it.

Caveat: PREEMPT_RT requires drivers to be RT-clean. Many vendor BSPs aren't. You'll spend time fixing IRQ handler `mdelay`s and untracked spinlocks held across `msleep`.

### Q2.14 — Real-world scenario: your Yocto-built Linux image boots, but the touchscreen driver doesn't probe. Walk through diagnosis.

1. **`dmesg | grep -i touch`** — does the driver even try to probe? If no message at all, the driver isn't built or isn't matched.
2. **Check kernel config.** `zcat /proc/config.gz | grep TOUCH` — is the driver enabled? If not, recipe issue.
3. **Check DT.** `dtc -I fs /sys/firmware/devicetree/base | grep -A 10 touch` — is the touch node present, status "okay", correct compatible string?
4. **Check the driver's match table.** `modinfo` or grep the source for `of_match_table`. Mismatch between DT compatible and driver's expected string is the most common bug.
5. **Driver loaded but probe fails.** `dmesg` will have the error. Common ones:
   - `EPROBE_DEFER` repeating — a dependency (regulator, GPIO controller, I2C bus) isn't ready.
   - I2C/SPI errors — wrong address, bus number, or chip not powered.
   - GPIO request failure — pin claimed by another driver (pinmux conflict).
6. **Hardware checks.** Power rail on the touch IC, reset line not asserted, INT line connected and configured. Scope the I2C bus during probe — do you see traffic to the right address? Do you see ACKs?
7. **DT overlay or status.** Sometimes vendor BSP ships with the touch disabled in DT (`status = "disabled"`). Override in your machine-specific dts.

Real story pattern: I've seen this caused by the I2C bus running at 400kHz when the touch chip needed 100kHz, by a pull-up missing on INT, and once by a Yocto recipe that removed the kernel module from the rootfs but not the kconfig. Always reach for the schematic and a scope after the software checks.

---

## 3. Yocto and Buildroot

### Q3.1 — Compare Yocto and Buildroot. When would you pick each?

Buildroot:
- A make-based system that builds a single rootfs from configuration. Output: kernel, bootloader, rootfs.
- Configuration-centric: `make menuconfig`, single `.config`-style file.
- Fast initial learning curve. Builds are reproducible-ish (less effort than Yocto to get bit-identical).
- Limited multi-image support — designed for one image per build.
- Smaller ecosystem of recipes than Yocto, but covers most common packages.
- Hard to support multiple machines without scripting around it.

Yocto / OpenEmbedded:
- A meta-build system with layers, recipes (BitBake), and an SDK output.
- Layer-based: BSP layer per board, distro layer for policy, application layer for your stuff.
- Steep learning curve. BitBake has its own quirks. But infinitely flexible.
- Multi-machine, multi-image, multi-distro from one source tree.
- Industry standard for commercial embedded Linux. AGL, NXP, TI, Xilinx all ship Yocto BSPs.
- Better at: SDK generation, package feeds, license compliance reports, reproducibility (eSDK, sstate-cache).

Pick Buildroot for: hobby projects, single-product startups where you want to ship fast, anything where the rootfs fits on a few MB and you don't care about long-term maintenance.

Pick Yocto for: products with multiple SKUs, multi-year maintenance horizons, anywhere you need an SDK to ship to application teams, regulated industries needing reproducible builds and audit trails.

In practice, vendor BSPs strongly bias the choice. NXP ships Yocto; choosing Buildroot means porting their stuff yourself.

### Q3.2 — Explain Yocto's layer model and the relationship between BitBake, recipes, OE-Core, and Poky.

- **BitBake.** The build engine. Parses recipes, resolves dependencies, schedules tasks (fetch, configure, compile, install, package), runs them in parallel.
- **OE-Core (`meta`).** The core OpenEmbedded layer. Provides the bbclasses (build patterns), core recipes (toolchain, libc, busybox, basic system), and the build infrastructure.
- **Poky.** A reference distribution that combines OE-Core, BitBake, and `meta-poky` (distro policy) and `meta-yocto-bsp` (a few reference BSPs). It's a starting point, not the only way.
- **Layers.** Directories with their own recipes that extend or override the build. Conventional layers:
  - **BSP layers** (`meta-freescale`, `meta-ti`, `meta-raspberrypi`) — machine definitions, kernel, bootloader, vendor drivers.
  - **Distro layers** (`meta-poky`, custom company distro) — policy: which init system, which package manager, default features.
  - **Software layers** (`meta-openembedded`, `meta-qt6`) — extra packages.
  - **Application layers** (your project's `meta-yourproject`) — your application recipes, your image, your machine if custom.
- **Recipes (`.bb`).** Per-package build instructions. `SRC_URI`, `LICENSE`, `do_compile`, etc.
- **Append files (`.bbappend`).** Modify a recipe from another layer without forking it. Your BSP layer might `.bbappend` `linux-yocto.bb` to add patches.
- **Classes (`.bbclass`).** Reusable build patterns. `inherit cmake` pulls in CMake-specific tasks.
- **`local.conf`** — your build's configuration. `MACHINE`, `DISTRO`, `IMAGE_INSTALL`.
- **`bblayers.conf`** — which layers are active.

Mental model: layers are a search path; recipes are build scripts; BitBake stitches them together based on `MACHINE` and image contents.

### Q3.3 — Walk through writing a BitBake recipe for a custom application that uses CMake, links against Qt, and needs a systemd service file.

```bitbake
SUMMARY = "My HMI application"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://LICENSE;md5=..."

SRC_URI = "git://git.example.com/myhmi.git;protocol=https;branch=main \
           file://myhmi.service"
SRCREV = "abc123..."

S = "${WORKDIR}/git"

DEPENDS = "qtbase qtdeclarative"
RDEPENDS:${PN} = "qtbase qtdeclarative-qmlplugins"

inherit cmake systemd

EXTRA_OECMAKE = "-DCMAKE_BUILD_TYPE=Release"

SYSTEMD_SERVICE:${PN} = "myhmi.service"
SYSTEMD_AUTO_ENABLE:${PN} = "enable"

do_install:append() {
    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/myhmi.service ${D}${systemd_system_unitdir}/
}

FILES:${PN} += "${systemd_system_unitdir}/myhmi.service"
```

Key things:
- `DEPENDS` is build-time. `RDEPENDS:${PN}` is runtime — without it, your image might miss the QML plugins your app loads at runtime.
- `inherit cmake` brings in `do_configure`, `do_compile`, `do_install` defaults that work for CMake projects.
- `inherit systemd` adds the service-enabling logic.
- `SRCREV` pinning is non-negotiable for reproducibility. Use `AUTOREV` only during dev.
- The `${PN}` is the package name; `${D}` is the install destination; `${WORKDIR}` is where sources unpack.

A subtle one: if your app uses QML, you also need `qtquickcontrols2` and various plugins in `RDEPENDS` — they're loaded by name at runtime, BitBake can't infer the dependency.

### Q3.4 — What is sstate-cache and how does it affect build times and reproducibility?

sstate (shared state) is Yocto's task-output cache. Each task's signature is hashed from its inputs (recipe content, dependencies, environment). After a task completes, its output is packaged into the sstate-cache. Next time the same hash comes up, the task is restored from cache instead of re-run.

Effects:
- **Build time.** First clean build of a typical image: 1-4 hours. Subsequent builds with sstate populated: minutes. CI fleets share an sstate-cache via NFS or HTTP mirror — `SSTATE_MIRRORS` URL.
- **Reproducibility.** sstate is keyed by deterministic hashes of inputs. If your inputs change unexpectedly (timestamps, hostnames, paths), hashes change, cache misses. Yocto has `BB_HASHBASE_WHITELIST` and reproducibility flags (`BUILD_REPRODUCIBLE_BINARIES`) to scrub these.
- **Pitfalls.** A bad sstate (stale, corrupt) can mask bugs — your build picks up an old object file. `bitbake -c cleansstate <recipe>` to force rebuild. CI should periodically validate from-scratch builds against sstate-restored ones.

For a team: stand up an internal sstate mirror behind HTTP. Configure `SSTATE_MIRRORS = "file://.* http://sstate.example.com/PATH"`. Developers pull from it; CI populates it. Build times across the team converge.

### Q3.5 — A junior engineer added a recipe and the image grew by 80MB. How do you find what's bloating it?

1. **`buildhistory`.** Enable it (`INHERIT += "buildhistory"`). It records per-image and per-package size changes between builds. `git diff` of `buildhistory/images/<machine>/<image>/files-in-image.txt` shows what was added.
2. **`bitbake -g <image>`** generates a dependency graph. `pn-buildlist` lists everything pulled in.
3. **Image size analysis.** `oe-pkgdata-util list-pkg-files <pkg>` shows files per package. `du -sh ${IMAGE_ROOTFS}/*` shows directory sizes.
4. **Check transitive dependencies.** Did the new recipe pull in something heavy? Common offenders: pulling in `python3` for a build script, pulling in `gstreamer` plugins for a small audio feature, pulling in `boost`.
5. **Image manifest diff.** Yocto produces an `<image>.manifest` listing all installed packages. Diff before/after.

Common fixes: `RDEPENDS` over-specified, debug symbols not stripped (`INHIBIT_PACKAGE_STRIP` accidentally set), full `qt5` instead of just modules needed, `-dev` packages slipping into the image, locale data (`IMAGE_LINGUAS = "en-us"` instead of all).

### Q3.6 — Explain `IMAGE_FEATURES` vs. `DISTRO_FEATURES`. Why do we have both?

- **`DISTRO_FEATURES`** — capabilities baked into the build globally. They affect *how packages are compiled*. `wifi` in DISTRO_FEATURES makes the kernel and wpa_supplicant build with WiFi support. Removing it later means rebuilding affected recipes.
- **`MACHINE_FEATURES`** — what the hardware supports. `bluetooth`, `usbhost`, `pci`. Constrains what DISTRO_FEATURES can effectively enable.
- **`IMAGE_FEATURES`** — what's *included in this specific image*. `debug-tweaks` enables empty root password and other dev conveniences. `splash` adds psplash. These don't change how packages are built, just what gets installed.

The intersection that ships in your image is roughly: package was built with the relevant DISTRO_FEATURE, machine supports it (MACHINE_FEATURES), and the image includes the package or feature group (IMAGE_FEATURES / IMAGE_INSTALL).

Critical real-world example: a feature in DISTRO_FEATURES that you don't need bloats every package that conditionally compiles it. Audit the list; remove `x11`, `wayland`, `pulseaudio`, etc. if your headless gateway doesn't need them. Saves hundreds of MB.

### Q3.7 — How do you patch the kernel from a Yocto recipe without forking the kernel tree?

A `.bbappend` to `linux-yocto.bb` (or whichever kernel recipe your BSP uses):

```
# linux-yocto_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI += "file://0001-my-fix.patch \
            file://my-defconfig.cfg"
```

Place patches in `meta-yourlayer/recipes-kernel/linux/linux-yocto/`. They apply during `do_patch`. Configuration fragments (`.cfg` files) merge into the kernel `.config`.

Key practices:
- Use proper patch headers with `From:` and `Subject:` so they're upstreamable.
- Prefer config fragments over modifying defconfig — fragments are versioned cleanly.
- Use `KMACHINE` and `COMPATIBLE_MACHINE` correctly so patches only apply where intended.
- For substantial changes, consider `kernel-meta` (SRC_URI metadata files) which lets you organize patches as features.

When changes get large enough — many patches, custom drivers — it's often cleaner to maintain a fork at a Git URL and pin SRCREV. Patch-stack-in-recipe works up to maybe 20-30 patches before becoming unwieldy.

### Q3.8 — What does `do_rootfs` do, and how is it different from `do_image`?

- **`do_rootfs`** — populates the root filesystem directory by installing all the packages listed in `IMAGE_INSTALL` (and recursively their `RDEPENDS`). Runs post-install scripts, generates `/etc/passwd`, locale data, etc. The output is a directory tree.
- **`do_image_*`** — packages the rootfs into deliverable formats (ext4, squashfs, tar.gz, wic). One task per format in `IMAGE_FSTYPES`.

Key insight: image type is decoupled from rootfs construction. Same rootfs can produce ext4 and tar.gz simultaneously by adding both to `IMAGE_FSTYPES`. `wic` is special — it composes a partitioned disk image from `.wks` recipes that reference other image artifacts (kernel, dtb, rootfs partition). That's how you get a complete bootable SD card image.

### Q3.9 — How do SDKs generated by Yocto differ from the build environment, and when does each matter?

Yocto produces two flavors:

- **Standard SDK** (`bitbake -c populate_sdk <image>`). A self-contained, relocatable cross-toolchain plus headers and libraries matching the image. Application developers use it without needing the Yocto build infrastructure. Source it (`source environment-setup-...`) and you get `$CC`, `$CXX`, `$CFLAGS` pointing at the cross-compiler.
- **Extensible SDK (eSDK)** (`bitbake -c populate_sdk_ext`). Includes a slimmed BitBake and lets developers add new recipes, modify existing ones, build custom images, and contribute changes back upstream — without setting up the full build env.

Standard SDK for: app developers building one app against a fixed BSP. Fast.
eSDK for: integrators who need to add packages, modify the kernel, etc., without becoming Yocto experts. Heavier.

The build environment (your `bitbake` checkout with all layers) is what *produces* SDKs and images. Day-to-day app developers in a healthy team should use the SDK, not the build env.

### Q3.10 — Real-world scenario: a Yocto build that worked yesterday now fails with checksum mismatches on a fetch. What are the likely causes and fixes?

Common causes:
1. **SRC_URI with no SRCREV pin and a moving branch.** Code changed upstream; checksum doesn't match `SRC_URI[sha256sum]`. Fix: pin SRCREV to a commit; remove the broken checksum or update it.
2. **Tarball URL changed contents.** Some upstreams overwrite tags or republish tarballs. Mitigated by Yocto's "premirrors" — set up an internal mirror of source tarballs to avoid this exact problem.
3. **Network proxy / corruption.** Partial download. Clear `DL_DIR/<file>` and retry.
4. **A vendor moved their server.** The URL still resolves to a 404 or login page that's HTML, not the file. Checksum is computed on the HTML.
5. **`AUTOREV` bit you.** You used `${AUTOREV}` and someone pushed.

Mitigation strategies:
- Internal source mirror (`PREMIRRORS`). All fetches try the mirror first.
- Pin everything. SRCREV for git, sha256sum for tarballs.
- `BB_NO_NETWORK=1` after a successful fetch to ensure builds are fully offline.
- For commercial products: archive `DL_DIR` per release. License compliance often requires it anyway.

### Q3.11 — Compare `IMAGE_INSTALL`, `CORE_IMAGE_EXTRA_INSTALL`, and `RDEPENDS:${PN}`. When do you use each?

- **`IMAGE_INSTALL`** in your image recipe: the canonical "what goes in this image." Set in `myimage.bb`. Use `+=` to extend.
- **`CORE_IMAGE_EXTRA_INSTALL`** in `local.conf`: a hook to add packages to any image inheriting `core-image.bbclass` without modifying the image recipe. Useful for developer-only additions (gdb, strace) without polluting the production image recipe.
- **`RDEPENDS:${PN}`** in a *package's* recipe: this package needs that package installed. Pulls in transitively when the parent is installed. Use this when a runtime dependency is mandatory for your package to function.

Anti-pattern: dumping everything into IMAGE_INSTALL and not declaring RDEPENDS in package recipes. Means your package only works because the image happens to install its dependencies. Move it to another image and it breaks.

### Q3.12 — What's the difference between Buildroot's BR2_PACKAGE_FOO=y and the postbuild script approach to customization?

In Buildroot:
- **`BR2_PACKAGE_FOO=y`** in `.config` (or a defconfig) installs the foo package into the rootfs. This is the supported, version-controlled way to add functionality.
- **Post-build / post-image scripts** (`BR2_ROOTFS_POST_BUILD_SCRIPT`, `BR2_ROOTFS_POST_IMAGE_SCRIPT`). Run after rootfs construction or after image generation. Used for: copying custom files, fixing permissions, generating dynamic config, building combined images.
- **Rootfs overlay** (`BR2_ROOTFS_OVERLAY`). Directory whose contents are copied verbatim into the rootfs. Static files: hostname, network config, custom scripts.

Best practice: prefer packages and overlays. Reserve postbuild scripts for things that genuinely need shell logic (signing, encrypting, packaging). Postbuild scripts are easy to abuse and end up being the source of "it builds for me" issues.

---

## 4. Build Systems: Make, CMake, Cross-Compilation

### Q4.1 — What is a CMake toolchain file, and what should it contain for cross-compilation to ARM?

A toolchain file is a CMake script loaded *before* the project, that tells CMake how to compile for a non-host target. CMake reads it via `-DCMAKE_TOOLCHAIN_FILE=path`.

Minimum contents:

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)

set(CMAKE_FIND_ROOT_PATH /path/to/sysroot)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

set(CMAKE_C_FLAGS_INIT "-mcpu=cortex-a53 -mfpu=neon")
```

Key things:
- `CMAKE_SYSTEM_NAME` triggers cross-compile mode in CMake. Without it, CMake assumes host build.
- `CMAKE_FIND_ROOT_PATH` plus the four `_MODE_*` settings keep `find_package`/`find_library` from accidentally finding host libraries.
- Compiler flags via `*_INIT` so user `CFLAGS` can append.

In Yocto, the SDK's `environment-setup-*` script generates a toolchain file (`OEToolchainConfig.cmake`) and sets `$CMAKE_TOOLCHAIN_FILE` for you. Don't reinvent — source the SDK and use it.

### Q4.2 — Explain the difference between `add_library`'s STATIC, SHARED, INTERFACE, OBJECT, and MODULE.

- **STATIC** — `.a` archive. Compiled and bundled into linker invocation. No runtime dependency.
- **SHARED** — `.so`/`.dll`/`.dylib`. Loaded at runtime by dynamic linker. Reduces image size if multiple binaries share it; adds runtime dependency.
- **MODULE** — like SHARED but not linked against directly; loaded explicitly via `dlopen()`. Plugins.
- **INTERFACE** — header-only library. No compiled artifact. You attach properties (include dirs, compile defs, dependencies) and consumers inherit them via `target_link_libraries`. The modern way to expose header-only deps.
- **OBJECT** — compiles sources to `.o` objects without making a library. Useful when you want to share compilation but link the objects directly into multiple targets, avoiding duplicate compilation.

In firmware:
- STATIC for everything except the final executable. Avoids dynamic linker complexity. (Though dynamic linking on full embedded Linux is fine and standard.)
- INTERFACE for things like `compile_options` aggregators or constants modules.
- OBJECT when you want one set of compilation but link into both an executable and tests.

### Q4.3 — What does `target_include_directories(... PUBLIC ... PRIVATE ... INTERFACE ...)` actually mean? Why is the distinction critical?

These are *visibility* keywords for usage requirements:
- **PRIVATE** — the include path is used when building *this* target. Consumers don't see it.
- **INTERFACE** — the include path is used by consumers (anyone linking this target), not by this target itself.
- **PUBLIC** — both. The target uses it, and consumers inherit it.

Why critical: it's the foundation of "modern CMake." Get this right and dependencies cascade correctly through `target_link_libraries`. Get it wrong and you get either build failures or invisible coupling.

Rule of thumb: if your *header* uses something (e.g., your public header includes `<boost/optional.hpp>`), the include dir must be PUBLIC or INTERFACE. If only your `.cpp` uses it, PRIVATE.

Same rules apply to `target_compile_definitions`, `target_compile_options`, `target_link_libraries`. The whole modern-CMake idiom is: each target declares its own usage requirements, and consumers inherit them via `target_link_libraries`. Stop using global `include_directories()` and `add_compile_options()`.

### Q4.4 — Compare Make and CMake. Why has CMake become dominant for cross-platform/embedded?

Make is a build engine: it understands "if A depends on B and B is newer, rebuild A." It doesn't know anything about compilers, dependencies, or platforms. You write the rules.

CMake is a *generator*: it produces Makefiles (or Ninja files, Visual Studio projects, Xcode projects). It encodes platform knowledge: how to find a library on Linux vs. Windows, how to build a shared library, how to handle Apple frameworks. You describe *what* you want, CMake figures out *how*.

Why CMake won:
1. **Multi-platform.** One CMakeLists.txt, build on Windows, Linux, macOS.
2. **IDE integration.** Generates VS, Xcode, Qt Creator, CLion projects.
3. **find_package.** Finds installed libraries portably.
4. **Modern CMake (3.x+).** Target-based design encapsulates dependencies cleanly.
5. **Vendor adoption.** Qt, Boost, OpenCV, gRPC — everything ships CMake.

What CMake doesn't replace: Yocto/Buildroot (those *use* CMake under the hood), Ninja for raw speed (CMake generates Ninja files), or Make for tiny standalone projects.

Trade-off: CMake's syntax is famously bad. Compared to Meson (much cleaner, also a generator) and Bazel (hermetic builds, complex), it's the messy incumbent — but the network effect is overwhelming.

### Q4.5 — Walk through what happens when you run `cmake -S . -B build` and then `cmake --build build`. Where does each step's information come from?

`cmake -S . -B build` — the **configure** step:
1. CMake reads `CMakeLists.txt` from the source dir.
2. Detects the host system (or reads toolchain file for cross).
3. Detects compilers (compiles a test program to verify).
4. Resolves all `find_package`, `find_library` calls — needs sysroot or system libs available.
5. Evaluates conditionals (`if(WIN32)`, `if(CMAKE_BUILD_TYPE STREQUAL Debug)`).
6. Generates build system files (Makefiles or Ninja files) in `build/`.
7. Caches discovered values in `build/CMakeCache.txt`. Re-running uses the cache unless explicitly cleared.

`cmake --build build` — the **build** step:
1. Invokes the underlying build tool (make, ninja, msbuild) on the generated files.
2. Build tool computes what's stale (timestamps), invokes compiler/linker.
3. CMake generates dependency tracking so header changes trigger rebuilds correctly.

If you change `CMakeLists.txt`, the generated files are regenerated automatically (Ninja/Make have a rule to re-run cmake). If you change a toolchain file, you have to delete the cache (`build/CMakeCache.txt`) — toolchain isn't re-evaluated otherwise.

For embedded teams: standardize the configure command in a script (`configure-target.sh`) that sources the SDK and invokes cmake. New devs run one script.

### Q4.6 — What is `pkg-config`, and how does it work with cross-compilation and CMake?

`pkg-config` is a small tool that reports compiler/linker flags for a library. Each library installs a `.pc` file (`gtk+-3.0.pc`) describing its include dirs, libs, deps, and version. `pkg-config --cflags --libs gtk+-3.0` returns the right flags. CMake's `find_package(PkgConfig)` plus `pkg_check_modules` integrates it.

Cross-compilation gotcha: by default `pkg-config` looks at host paths (`/usr/lib/pkgconfig`). For cross-compiling, you need:

```
PKG_CONFIG_LIBDIR=$SDK_SYSROOT/usr/lib/pkgconfig:$SDK_SYSROOT/usr/share/pkgconfig
PKG_CONFIG_SYSROOT_DIR=$SDK_SYSROOT
PKG_CONFIG_PATH=  # don't bleed in host paths
```

This makes `pkg-config` look in the sysroot and prepend the sysroot path to absolute paths in `.pc` files. The Yocto SDK setup script does this for you.

Fun gotcha: `.pc` files in vendor SDKs sometimes have host-baked paths (`/usr/lib/...`) that need fixup. Yocto handles this; manually-built sysroots may not.

### Q4.7 — Your CMake project links a static library, but symbols are missing at link time. The symbols are definitely in the .a. What's wrong?

Static library link order matters: when ld processes `-la`, it pulls in only objects needed to resolve currently-undefined symbols. If `lib1.a` needs symbols from `lib2.a`, you must list `lib1` *before* `lib2` on the link line.

```
gcc main.o -l1 -l2  # works if l1 needs l2
gcc main.o -l2 -l1  # may fail with undefined references
```

In modern CMake, `target_link_libraries(myapp PRIVATE lib1 lib2)` orders them as written. If you have circular-ish dependencies (lib1 needs lib2, lib2 needs lib1), use `--start-group`/`--end-group`:

```cmake
target_link_libraries(myapp PRIVATE -Wl,--start-group lib1 lib2 -Wl,--end-group)
```

Or, more idiomatically, restructure to avoid the cycle.

Other causes of missing symbols:
- C++ name mangling — a header declared `extern "C"` but the lib was built C++.
- Wrong ABI (libstdc++ vs libc++).
- Symbol stripped (`-s`) accidentally.
- Symbol in a `.a` archive but not pulled in because no object referenced it. With `--whole-archive`, force everything in.

### Q4.8 — Explain ccache and how it integrates with CMake. What problem does it solve?

ccache caches compiler invocations. Same source + same flags + same compiler = cached `.o` from previous build. Misses go to the real compiler.

For a clean rebuild after `git checkout`: without ccache, full recompile (minutes to hours). With ccache, almost instantaneous if the cache has the objects.

CMake integration:

```cmake
find_program(CCACHE ccache)
if(CCACHE)
    set(CMAKE_C_COMPILER_LAUNCHER ${CCACHE})
    set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE})
endif()
```

Or set `CC="ccache gcc"` in the env.

Caveats:
- Cache hash includes preprocessor output; macros affecting compilation are correctly differentiated.
- `-Werror` plus warnings depending on build context can cause inconsistent hits — usually fine.
- Remote ccache (via `CCACHE_REMOTE_STORAGE`) lets a team share. Big win for CI fleets.

ccache is roughly distinct from Yocto sstate-cache: ccache is per-compilation-unit and works for any build; sstate is per-recipe-task and works inside Yocto. Both can be enabled together.

### Q4.9 — What are CMake "presets" and why use them?

`CMakePresets.json` is a JSON file (since CMake 3.19) standardizing configure/build/test invocations. Instead of `cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=... -B build/release`, you do `cmake --preset arm-release`. The preset records the full configuration.

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "arm-release",
      "binaryDir": "build/${presetName}",
      "toolchainFile": "cmake/arm-toolchain.cmake",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release"
      }
    }
  ]
}
```

Why use them:
- Standard interface across IDE and CLI. Qt Creator, VS Code, CLion all read presets.
- New devs run `cmake --preset arm-release` and don't need a wiki page of flags.
- CI uses the same presets as developers — no drift.
- `CMakeUserPresets.json` (gitignored) lets devs add personal presets.

For cross-compile-heavy projects, presets are a clear win.

### Q4.10 — Cross-compiling, you get "ELF file class invalid" on the target. What does this mean?

It means the binary's architecture or bit-ness doesn't match the target. The dynamic linker can read the ELF header but the class (32 vs 64 bit) or architecture is wrong.

Common causes:
- Built for x86_64, copied to ARM target. Use `file <binary>` to check what you actually built.
- Built for armv7 (`-march=armv7-a`), running on armv6.
- Built against a sysroot but using the wrong libc (glibc binary on a musl target).
- Hard-float vs soft-float mismatch. ARM EABI distinguishes them; mixing crashes.

Fixes:
- Verify your toolchain prefix (`arm-linux-gnueabihf-` is hard-float, `arm-linux-gnueabi-` is soft).
- Source the right SDK setup script.
- Check `readelf -A binary` for ABI tags.
- For Yocto, verify `MACHINE` setting matches the deployment target.

### Q4.11 — How does CMake's `find_package` work, and what's the difference between Module mode and Config mode?

`find_package(Foo)` searches for either:

- **Module mode**: looks for `FindFoo.cmake` in `CMAKE_MODULE_PATH` or built-in. CMake-team-maintained or self-written. Used for libraries that don't ship CMake config (zlib, OpenSSL historically).
- **Config mode**: looks for `FooConfig.cmake` (or `foo-config.cmake`) installed by the library itself. Modern libraries ship this. CMake searches `CMAKE_PREFIX_PATH`, system paths, and `<prefix>/lib/cmake/Foo/`.

Config mode is preferred — the library author knows their structure best. `FindFoo.cmake` modules are a workaround for libraries that don't.

In cross-compile, `CMAKE_PREFIX_PATH` should point to the sysroot. The toolchain file's `CMAKE_FIND_ROOT_PATH` plus `CMAKE_FIND_ROOT_PATH_MODE_PACKAGE=ONLY` keeps `find_package` searching only the sysroot.

When you write a library, ship a `FooConfig.cmake` via `install(EXPORT ...)` and `configure_package_config_file()`. Consumers do `find_package(Foo CONFIG REQUIRED)` and `target_link_libraries(theirapp PRIVATE Foo::Foo)`. Clean.

### Q4.12 — Explain `add_custom_command` vs. `add_custom_target`. Real-world use case for each.

- **`add_custom_command(OUTPUT foo.h DEPENDS foo.idl COMMAND idlgen ...)`** — declares a *file* generation rule. The rule runs only when the output is needed by something. Used for: code generation (protobuf, IDL, lex/yacc), resource compilation (Qt's `moc`/`uic`/`rcc` — though `qt_add_executable` handles this), embedded-data conversion (xxd-style binary-to-header).
- **`add_custom_target(name COMMAND ...)`** — declares a *named* target that always runs when invoked. Used for: deploy scripts, format-checks (`clang-format`), packaging (zip up artifacts), running tests in unusual ways.

Real-world embedded:
- `add_custom_command` to run a Python script that converts a JSON config to a C++ header at build time. The header is `DEPENDS` on the JSON, so editing the JSON triggers a rebuild.
- `add_custom_target(flash COMMAND st-flash write ${CMAKE_BINARY_DIR}/firmware.bin 0x08000000)`. Run with `cmake --build build --target flash`. Devs love it.

Common pitfall: `add_custom_target` always runs (it has no outputs to compare timestamps against). Don't use it for file generation; use `add_custom_command` and have a target depend on the output.

### Q4.13 — Real-world scenario: your CMake build works locally but fails on CI with "could not find Qt." Walk through diagnosis.

1. **Is Qt installed on CI?** Often the first answer. CI containers may not have Qt unless explicitly installed.
2. **Is the right Qt installed?** Qt5 vs Qt6. `find_package(Qt6 REQUIRED)` won't find Qt5.
3. **Is `CMAKE_PREFIX_PATH` set?** Qt installs typically need a hint: `-DCMAKE_PREFIX_PATH=/opt/Qt/6.6.0/gcc_64`.
4. **For cross-compile**, are you sourcing the SDK? Yocto SDK provides Qt; without sourcing, CMake searches host.
5. **Architecture mismatch.** SDK provides ARM Qt; CI runs CMake configure for x86 by accident.
6. **`Qt6_DIR` cached.** A previous build cached a wrong path. Delete `build/` and re-configure.
7. **Container-specific.** The CI image might lack `qt6-base-dev` even though the host had it.

Fix patterns:
- CI script: explicit `CMAKE_PREFIX_PATH` or `Qt6_DIR`. Don't rely on system find paths.
- Pin Qt version in container build.
- Use the same SDK on CI as devs use locally — eSDK or sourced standard SDK.
- Add a CMake check that errors with a useful message: `if(NOT TARGET Qt6::Core) message(FATAL_ERROR "Qt6 not found; set CMAKE_PREFIX_PATH or source SDK") endif()`.

---

## 5. Qt and QML

### Q5.1 — Explain Qt's signal/slot mechanism. What's actually happening under the hood, and what are the connection types?

Signal/slot is Qt's type-safe observer pattern. Under the hood:

- `Q_OBJECT` macro and `moc` (the meta-object compiler) generate a `QMetaObject` for each class — a runtime description with method signatures, signal/slot info, properties.
- `connect(sender, &Sender::sig, receiver, &Receiver::slot)` records the connection in the sender's `QMetaObject`'s connection list.
- Emitting a signal (`emit sig(args)`) calls the moc-generated `sig(args)` function which iterates connections and dispatches to slots.

Connection types (set via 5th arg to `connect`):
- **`Qt::AutoConnection`** (default). Resolves at emission: same-thread → DirectConnection, cross-thread → QueuedConnection.
- **`Qt::DirectConnection`**. Slot runs immediately in the emitter's thread. Like a function call.
- **`Qt::QueuedConnection`**. Slot is posted as an event to the receiver's thread's event loop. Args are copied (must be registered with `qRegisterMetaType` for non-built-in types). Async.
- **`Qt::BlockingQueuedConnection`**. Like Queued, but emitter blocks until slot returns. Cross-thread synchronous call. Will deadlock if both threads are the same.
- **`Qt::UniqueConnection`** (flag). Don't add the connection if it already exists.

Critical for embedded: cross-thread signal/slot uses the event loop. If your worker thread doesn't run an event loop (`exec()`), queued signals to it never fire. This is the #1 newbie threading bug in Qt.

### Q5.2 — What's the difference between QQuickItem, QQuickPaintedItem, and QQuickFramebufferObject? When would you use each in a custom QML component?

All three are bases for custom QML visual items, varying in how they render:

- **`QQuickItem`** — declares a scene-graph node. You build the geometry in `updatePaintNode()` returning `QSGNode*` (typically `QSGGeometryNode` with a custom material). Renders directly via the scene graph's GL/Vulkan/Metal backend. Highest performance, lowest abstraction. Used for: custom shapes, charts, particle effects.
- **`QQuickPaintedItem`** — gives you a `QPainter`-based `paint()` method. Convenient (QPainter API), but Qt has to render to a texture and upload. Slower than QQuickItem. Used for: simple custom drawings where you don't want to learn the scene graph, or porting from QGraphicsView.
- **`QQuickFramebufferObject`** — you render to an FBO with raw GL calls. Qt composites the FBO into the scene. Used for: 3D content, custom OpenGL renderers, video pipelines, GPU compute output displayed on screen.

For embedded GPUs (Mali, Adreno, PowerVR), `QQuickItem` with the scene graph is overwhelmingly the right choice — the SG is optimized for batching. Falling back to `QQuickPaintedItem` for everything is a common performance footgun.

### Q5.3 — Compare Qt Quick (QML) and Qt Widgets. When do you use each on an HMI?

Qt Widgets:
- Mature, desktop-centric. `QPushButton`, `QListView`, `QFormLayout`.
- Renders via QPainter (CPU) or with the optional OpenGL backend.
- Limited animation, no built-in GPU compositing.
- Excellent for traditional desktop apps.

Qt Quick / QML:
- Scene-graph, GPU-accelerated. Designed for animations and touch.
- Declarative QML language for UI; C++ backend for logic.
- Better designer/developer separation (QML is editable, Widgets is C++).
- Animations, transitions, gradients, shaders are first-class.

For an embedded HMI on a touchscreen (the work you're doing now): QML is overwhelmingly the right answer. Widgets shows its desktop heritage — no native gestures, awkward animations, doesn't leverage GPU on iMX/Jetson/etc.

For an internal tool, a developer dashboard, an industrial config app on a desktop PC: Widgets is faster to write, looks "right" on Windows/Mac/Linux.

Mixed: QQuickWidget lets you embed QML in Widgets, useful during migration.

### Q5.4 — In QML, explain the difference between Item, Loader, Repeater, ListView, and Component. When does each matter?

- **Item** — base type. A non-visual rectangle (no fill). Container for layout/grouping.
- **Component** — a definition (template) that can be instantiated. `Component { Rectangle { ... } }` defines but doesn't show. Created via `Loader`, `Repeater`, dynamic `Qt.createComponent`, or `Component.onCompleted` patterns.
- **Loader** — instantiates a component on demand, typically based on a `source` URL or a `sourceComponent`. Lazy loading. Used for screens/pages that aren't shown until needed.
- **Repeater** — instantiates N copies of a component for a model. Eager — all instances live, all visible. Use for small N (icons in a toolbar).
- **ListView** / **GridView** — view types over a model. Lazy: only items visible (plus a buffer) are instantiated. Recycles items. Use for any potentially-long list.

When this matters: I've seen HMIs with hundreds of items in a `Repeater` cause 2-second startup hangs because every item materialized. Switching to `ListView` with a delegate makes it instant. Conversely, using `ListView` for 5 toolbar icons is over-engineered.

### Q5.5 — What is Qt's property system, and how does it integrate with QML?

Qt properties are declared via `Q_PROPERTY`:

```cpp
class Sensor : public QObject {
    Q_OBJECT
    Q_PROPERTY(double temperature READ temperature NOTIFY temperatureChanged)
public:
    double temperature() const;
signals:
    void temperatureChanged();
};
```

The property system gives you:
- Reflection: `obj->property("temperature")` works at runtime.
- QML binding: in QML, `text: sensor.temperature` re-evaluates whenever `temperatureChanged()` fires.
- Animation: `PropertyAnimation` interpolates property values.
- Serialization: `QSettings` and similar can save properties.

For QML integration, the recipe:
1. Declare properties with `Q_PROPERTY` and `NOTIFY` signals.
2. Register the type: `qmlRegisterType<Sensor>("MyApp", 1, 0, "Sensor")` or, in modern Qt 6, use `QML_ELEMENT` + `qt_add_qml_module`.
3. Use it: `Sensor { id: s }` then `Text { text: s.temperature }`.

The `NOTIFY` signal is critical — without it, QML bindings can't refresh. Forgetting it means the value shows the initial state and never updates. Static analysis tools (qmllint) catch this.

### Q5.6 — Explain Qt Quick Controls 2's "styles" system. How does it differ from Qt Quick Controls 1, and why does it matter for embedded?

Qt Quick Controls 1 (deprecated) was a heavyweight stack of styled controls implemented in QML on top of native styles. Each control had layers of bindings; performance was mediocre on embedded GPUs.

Qt Quick Controls 2 was rewritten with embedded performance in mind:
- C++-backed implementation for hot paths.
- Style is set per-app, not per-control. Built-in: Default, Material, Universal, Fusion, Imagine.
- Custom styles via QML files in a directory matching the style name.
- Much fewer scene-graph nodes per control.

For embedded, this matters because:
- Lower per-control overhead → smoother UI on modest GPUs.
- "Imagine" style lets designers skin everything from PNG/SVG without touching C++.
- Style is selected via `qtquickcontrols2.conf` or env var, swappable without recompile.

For your John Deere HMI work: QQC2 with a custom style is the contemporary baseline. QQC1 is dead.

### Q5.7 — What is the QML rendering thread model? What's the dedicated render thread, and what runs on the GUI thread?

Qt Quick uses a multi-threaded render loop on most platforms:

- **GUI thread** (main thread): runs QML JavaScript, handles input events, processes signals/slots, calls `updatePaintNode()` on items, and prepares the scene graph delta.
- **Render thread**: takes the scene graph snapshot, issues GL/Vulkan calls, swaps buffers. Runs while GUI thread is doing the next frame's work — pipelining.

Implications:
- **Don't block the GUI thread.** Heavy JS or expensive C++ slots stutter the UI. Use threaded workers.
- **Don't touch GUI objects from the render thread.** `updatePaintNode()` runs while GUI thread is blocked, so it's safe to read item state there. But `QSGNode` lifetimes need care.
- **Threaded vs. basic render loop.** Some platforms (older drivers, broken VSync) fall back to "basic" render loop which renders on the GUI thread. Slower but simpler.

For embedded, the threaded loop is critical to hit 60fps. Verify it's enabled: `QSG_INFO=1` env var prints what's chosen.

### Q5.8 — How does Qt for Device Creation / Boot to Qt differ from a generic Qt build, and why might you use it on a product?

Boot to Qt is a commercial offering (now part of Qt for Device Creation):
- A pre-built Yocto-based Linux image with Qt installed and tuned.
- A reference HMI that boots to a Qt app within seconds.
- Tooling integration: deploy from Qt Creator to the target with one click.
- Optional commercial licensing covering GPL exceptions.

What you get vs. a hand-rolled image:
- Faster time-to-prototype.
- Tested platform configurations across NXP, TI, Renesas, etc.
- Commercial support and bug fixes.

What it costs:
- Money (per-device royalties or per-developer seats).
- Less control. The reference image is opinionated.

When to use: companies shipping HMI products that prioritize time-to-market over BSP customization, or that need commercial Qt licenses anyway (proprietary apps statically linked to LGPL Qt require dynamic linking workarounds; a commercial license avoids them). DIY (open-source Qt + your own Yocto BSP) is fine for teams with the expertise — your team probably falls here.

### Q5.9 — A QML application's UI lags during heavy backend work. Walk through how you'd diagnose and fix.

1. **Identify what's blocking.** Run with `QSG_RENDER_TIMING=1` and `QML_PROFILER` on. Check whether the GUI thread is busy in JS, in a C++ slot, or starved.
2. **`QML Profiler` in Qt Creator** shows binding evaluation times, JS execution, and frame timing. Heavy bindings (bindings that re-evaluate often and do expensive work) light up.
3. **Common causes:**
   - **Synchronous I/O on the GUI thread.** File reads, network requests, DB queries. Move to a worker thread or async API.
   - **Heavy property bindings.** A binding that runs `JSON.parse` on every change. Cache, throttle, or compute in C++.
   - **Excessive QML object creation.** A `Repeater` with 1000 complex delegates. Switch to `ListView` with reuseItems.
   - **Big images decoded on GUI thread.** Use `Image.asynchronous: true` or pre-process to texture-friendly formats.
   - **Animations triggering layout recomputation.** Transform-only animations are cheap; width/height animations re-layout.
4. **Fix patterns:**
   - **`QtConcurrent::run` / `QThread` worker.** Heavy C++ work off-thread, results back via queued signals.
   - **`WorkerScript` in QML.** Run JS off the GUI thread.
   - **C++ models via `QAbstractItemModel`** instead of JS arrays of QML objects. Massive speedup for large lists.
   - **Throttle bindings** with `Timer` debounce.
5. **GPU side**: if the render thread is the bottleneck (not GUI), simplify shaders, reduce overdraw, batch geometry.

For your day job: the J1939 message stream into QML is a classic case — never decode messages on the GUI thread, never bind QML directly to per-message updates. Buffer in C++, throttle to 30Hz, expose via a model.

### Q5.10 — Explain Qt's parent/child memory model. How does it interact with smart pointers and QML ownership?

Every `QObject` can have a parent. When the parent destructs, it deletes its children. This gives you tree-shaped memory management without smart pointers.

Pattern:
```cpp
auto* parent = new QWidget();
auto* child = new QPushButton(parent);  // Child cleaned up when parent is.
```

Interactions:
- **Don't mix `std::unique_ptr<QObject>` with QObject parents.** Both will try to delete, double-free. Use `QScopedPointer` or `std::unique_ptr<T, QObjectDeleteLater>` if needed, or just use parents.
- **`std::shared_ptr<QObject>`** works but Qt's parent system doesn't know about it. Risky.
- **QML ownership.** Objects created in QML are owned by QML's GC. C++ objects exposed to QML default to C++ ownership unless you set `QQmlEngine::setObjectOwnership(obj, JavaScriptOwnership)`. Returning a raw pointer from a Q_INVOKABLE method is a trap if QML decides to GC it.
- **`deleteLater()`.** Posts a delete to the event loop. Safe to call from slots or anywhere; the actual delete happens after the current event finishes. Useful when you can't delete immediately (e.g., from inside the object's own slot).

For C++ models exposed to QML: own them in C++ (a member of a singleton or main app), expose by raw pointer with `CppOwnership`. Don't `new` them and hand off.

### Q5.11 — Compare QStringLiteral, QLatin1String, and QString::fromUtf8(). When does each matter?

- **`QStringLiteral("hello")`** — compile-time UTF-16 literal. The QString header is laid out at compile time; constructing the QString is a no-op at runtime (just sets a pointer). Free.
- **`QLatin1String("hello")`** — wraps a `const char*` Latin-1 string. Avoids the UTF-8-to-UTF-16 conversion when comparing or assigning. Cheap, but only for ASCII/Latin-1.
- **`QString::fromUtf8("hello")`** — runtime conversion from UTF-8. Allocates and converts. Not free.
- **`QString("hello")`** — implicitly UTF-8 (Qt 6) or local 8-bit (Qt 5 with `QT_NO_CAST_FROM_ASCII` off). Same cost as `fromUtf8`.

Rules of thumb:
- For literals in code: `QStringLiteral`. Faster, no warnings under `QT_NO_CAST_FROM_ASCII`.
- For comparisons against literals: `QLatin1String("...")` or `u"..."` (Qt 6 with `QString::operator==(QStringView)`).
- For runtime data from C strings: `fromUtf8`.

In a hot path on embedded (e.g., parsing, key lookups), this matters: replacing `QString("CONFIG")` with `QStringLiteral("CONFIG")` removed a measurable allocation in a tight loop on a Cortex-A7-class system I worked on. Profile before micro-optimizing.

### Q5.12 — Real-world scenario: your QML HMI updates from a background CAN bus reader at 1000 Hz. Naive binding makes the UI unresponsive. Architecture?

1. **CAN reader thread** — C++ `QObject` moved to a `QThread`. Reads SocketCAN, parses J1939, builds parameter snapshots.
2. **Throttling layer** — a `QTimer` (in the reader thread or a dedicated decimator) emits a snapshot at 20-30 Hz to the GUI thread. Coalesces 1000 Hz of changes into refresh-rate-aligned updates. The HMI doesn't need 1000 Hz — your eye doesn't see it, your pixels don't display it.
3. **Snapshot model** — a `QObject` with `Q_PROPERTY` for each displayed parameter, in the GUI thread. Receives the snapshots via queued connection. Emits `*Changed` signals.
4. **QML bindings** — bind to the snapshot model's properties. Bindings re-evaluate at the throttled rate.
5. **For lists or histories** — use a `QAbstractListModel` with `dataChanged()` for affected ranges, not per-row signals.
6. **Memory** — pre-allocate ring buffers in C++; don't let the reader thread push QObjects.

Anti-patterns to avoid:
- Direct binding from the reader (DirectConnection across threads to QML). Crashes or undefined.
- Emitting a signal per CAN frame across threads. Floods the event queue, GUI can't drain.
- JS arrays in QML for histories. Slow GC, bad perf.

This is fundamentally what the J1939-to-HMI bridge looks like in real industrial UIs.

### Q5.13 — What's QtConcurrent, and when do you use it vs. QThread vs. QThreadPool?

- **`QtConcurrent`** — high-level. `QtConcurrent::run(func, args)` runs `func` on the global thread pool, returns a `QFuture`. Map/filter/reduce on containers in parallel. Easy.
- **`QThreadPool`** — manage a pool of `QRunnable` jobs. Lower-level than QtConcurrent but higher than QThread.
- **`QThread`** — a single thread. Either subclass and override `run()` (quick scripts), or instantiate and `moveToThread` an `QObject` (proper Qt event-loop thread).

Decision tree:
- Fire-and-forget background work, get a result later → `QtConcurrent::run`.
- Long-lived worker with an event loop and signals/slots → `QObject` + `moveToThread` + `QThread`. CAN reader, file watcher, etc.
- Many short jobs in parallel (image processing tiles) → `QThreadPool`.

Common mistake: subclassing `QThread` and putting slots on the subclass. The slots run on the thread that *created* the QThread (the GUI thread), not on the thread itself. The right pattern is: a worker `QObject`, instantiate a `QThread`, `worker.moveToThread(&thread)`, `thread.start()`.

---

## 6. RTOS, Interrupts, State Machines, Embedded Architecture

### Q6.1 — Compare a bare-metal super-loop, an RTOS (FreeRTOS/Zephyr), and a full Linux. What are the trade-offs and when do you pick each?

Super-loop:
- `while(1) { do_a(); do_b(); do_c(); }` — cooperative, single-threaded.
- Tiny code footprint, deterministic if tasks are short.
- Hard to maintain as complexity grows. Adding a new task means surgery on the loop.
- No preemption — a misbehaving function blocks everything.
- Best for: simple sensors, motor controllers with one main loop, anything <10KB code, tight power budgets.

RTOS (FreeRTOS, Zephyr, ThreadX, RTEMS, Mbed OS):
- Multiple tasks with priorities, preemption, blocking primitives (semaphores, queues, mutexes).
- 5-30KB overhead.
- Predictable latencies; you can analyze worst-case schedulability.
- No memory protection by default (MPU optional, less mature than Linux).
- No filesystems, networking, etc., out of the box (though Zephyr is closing the gap).
- Best for: most modern MCU products. Anything with > 1 concurrent activity, anything with deadlines, anything with a network stack.

Embedded Linux:
- Full POSIX, drivers ecosystem, packages, networking, security frameworks.
- 50-200MB minimum, MMU required, slow boot (seconds).
- Not real-time by default; PREEMPT_RT bridges some of the gap.
- Best for: HMIs, gateways, anything needing complex software, anything with displays/networking/storage.

Decision: cost and complexity. Linux costs you DDR, eMMC, MMU, power, boot time. RTOS costs you sophistication. Super-loop costs you scalability. Pick the cheapest tool that does the job.

Hybrid systems are common: Linux on AP for HMI + Cortex-M coprocessor with FreeRTOS for hard real-time motor control, communicating via shared memory. ARM SoCs increasingly ship with this exact topology (i.MX 8M Plus, RT11xx).

### Q6.2 — Explain priority inversion. Give a concrete example and the techniques to prevent it.

Priority inversion: a high-priority task is blocked by a low-priority task holding a resource, while a medium-priority task runs (preempting the low-priority task). Net effect: high-priority task waits for medium-priority task — inversion.

Mars Pathfinder is the canonical example: an information bus task (high priority) shared a mutex with a meteorological task (low priority); a communications task (medium priority) preempted the meteorological one while it held the mutex; watchdog reset the rover.

Techniques to prevent:
- **Priority inheritance.** When a high-priority task blocks on a mutex held by a lower one, temporarily boost the lower task's priority to that of the highest waiter. Released when mutex is freed. FreeRTOS supports this via `xSemaphoreCreateMutex` (vs. `xSemaphoreCreateBinary`).
- **Priority ceiling protocol.** Assign each mutex a ceiling = highest priority of any task that uses it. Acquiring it boosts the holder to the ceiling. Bounds inversion.
- **Disable preemption** while holding the resource. Crude but effective for short critical sections.
- **Lock-free designs.** Avoid the shared resource; use atomics and lock-free queues.

In practice on FreeRTOS: use mutexes (PI), not binary semaphores, for shared resources between tasks of different priorities. Document priority assignments and the mutexes each task takes.

### Q6.3 — What's the difference between a hardware interrupt, a software interrupt, and an exception on Cortex-M?

On ARM Cortex-M, all are handled through the same exception mechanism (NVIC):

- **Hardware interrupts** (IRQs) — peripherals signal NVIC; vectored to handler. UART RX, DMA done, GPIO edge.
- **Software interrupts** — generated by software. Two flavors:
  - `SVCall` (supervisor call) — a `SVC` instruction triggers it. Used for syscalls in RTOS implementations to switch to privileged mode safely.
  - `PendSV` — a special "pendable" interrupt. RTOS schedulers use it for context switches; lowest priority so it runs after all real ISRs are done.
- **Exceptions** — synchronous events from CPU itself: HardFault, MemManage, BusFault, UsageFault. Triggered by bad memory access, undefined instruction, divide by zero, etc.

All share the same vector table at flash start, the same exception entry/exit sequence (push xPSR, PC, LR, R12, R0-R3 onto the active stack), and the same NVIC priority configuration.

The PendSV mechanism is elegant: the scheduler doesn't need to do context switching synchronously in every ISR; it just sets PendSV pending. After the highest-priority work is done, PendSV fires (lowest priority) and does the switch. This decouples ISR latency from scheduler complexity.

### Q6.4 — Walk through the FreeRTOS context switch on Cortex-M. What's saved, where, and by whom?

ARM hardware does the first half automatically on exception entry:
- Pushes `xPSR, PC, LR, R12, R0-R3` (the "auto-saved" context) onto the active stack (PSP for tasks, MSP for handlers).
- Loads exception number, switches to handler mode.

For a context switch (PendSV):
1. PendSV handler entry. CPU has already saved auto-context onto the outgoing task's PSP.
2. Handler reads `psp` register, then **manually saves** R4-R11 (callee-saved) onto the same stack. (FPU context is saved if `EXC_RETURN` indicates FP was used.)
3. Saves the new PSP value into the outgoing task's TCB (`pxCurrentTCB->pxTopOfStack`).
4. Calls scheduler (`vTaskSwitchContext`) to pick the next task.
5. Loads new task's `pxTopOfStack` into PSP.
6. Pops R4-R11 from the new stack.
7. Returns from exception via `BX LR` with the right `EXC_RETURN`.
8. Hardware automatically pops `xPSR, PC, LR, R12, R0-R3` from new task's stack and resumes execution at its PC.

Net effect: callee-saved regs are saved/restored manually; caller-saved are saved by hardware. The scheduler picks who runs next. The whole switch is 30-100 cycles depending on FPU.

For embedded interview depth: be ready to discuss `EXC_RETURN` magic values (which stack to use, which mode to return to), MSP vs PSP (handlers use MSP, tasks use PSP), and why FreeRTOS picks PendSV over SVC for the context switch (PendSV's pendability and tail-chaining).

### Q6.5 — What is the deferred interrupt pattern (top half / bottom half), and why is it important?

Top half: the ISR. Runs with interrupts disabled (or at high priority), must be fast. Acks the device, captures critical data, signals work to be done later.

Bottom half: a task or work item that does the heavy processing — protocol parsing, application callbacks, logging.

Why:
- **ISR latency.** Long ISRs delay other interrupts. A 50µs ISR blocks a 10µs RT requirement.
- **Locking.** ISRs typically can't take regular mutexes (would deadlock if the holder is the preempted task). Bottom-half tasks can.
- **Re-entrance.** Bottom halves run in task context; you can call `printf`, take mutexes, allocate.

FreeRTOS implementation: ISR signals via `xSemaphoreGiveFromISR` or `xQueueSendFromISR`. A worker task blocks on that semaphore/queue and runs the heavy lifting.

Linux equivalent: tasklets (deprecated), softirqs, workqueues, threaded IRQs.

Bare-metal equivalent: ISR sets a flag; main loop polls and processes. (Super-loop's bottom half is "the rest of the loop.")

Real-world: a UART RX ISR that captured a byte and dropped it into a ring buffer (top half, ~20 cycles); a parser task that decoded protocol frames from the ring (bottom half, ms-class). Decoupling let the UART run at 1 Mbaud without missing bytes.

### Q6.6 — Compare semaphores, mutexes, and queues in an RTOS. When do you use each?

- **Mutex.** Mutual exclusion for a shared resource. Has an owner; only owner can release. Supports priority inheritance. Used for: protecting a peripheral, a data structure, a logging channel.
- **Binary semaphore.** Signaling between contexts. ISR signals task. No owner. No PI. Used for: "interrupt happened, wake the worker."
- **Counting semaphore.** Tracks a count of available resources. ISR or producer increments; consumer waits and decrements. Used for: bounded resource pools, counting occurrences.
- **Queue / message queue.** Pass data between contexts. Producer enqueues, consumer dequeues, blocks if empty/full. Used for: command channels, event streams, work distribution.

Subtleties:
- Don't use a mutex for ISR-to-task signaling. The mutex would have to be "owned" — and ISRs can't own things.
- Don't use a binary semaphore where you really mean a mutex (no PI; priority inversion bug waiting).
- Queues with copy-by-value semantics avoid most ownership issues but have copy cost. Pass pointers for big payloads (and manage lifetime carefully).

### Q6.7 — Explain a hierarchical state machine (HSM). When do you reach for one over a flat FSM?

A flat FSM has states and transitions; an HSM has nested states (a state can contain sub-states). Behavior at the parent state is inherited unless overridden by a child.

Example: a UI with states `Idle`, `Active` (containing `MenuOpen`, `Editing`, `Confirming`), `Error`. A "back" button behavior is defined at `Active` level — applies to all sub-states unless one overrides it. Without HSM, you'd duplicate the back-handler in every sub-state.

When to use HSM:
- Many states with shared behavior. Flat FSM would have N×M edges; HSM compresses via inheritance.
- Long-running modes with sub-modes (e.g., a machine controller's `Running` state with `Auto`, `Manual`, `Calibrate` sub-states).
- UI navigation flows.
- Anywhere you find yourself copy-pasting transitions across states.

Implementations:
- Hand-coded (Miro Samek's QP framework is the reference).
- C++ Boost.SML or `std::variant` + visitor pattern for typed transitions.
- Tools: Yakindu, Stateflow (Simulink), QM (Samek's GUI tool).

For embedded: Samek's QP-nano / QP-C is widely used for safety-critical HSMs. His book "Practical UML Statecharts in C/C++" is the canonical reference.

### Q6.8 — What is a watchdog timer, and how do you architect software to use one correctly?

A watchdog is a hardware timer that resets the MCU if not periodically "kicked." If the system hangs or runs amok, the watchdog brings it back.

Naïve usage: kick the watchdog from a single high-priority task. Bad — if a low-priority task hangs but the kicker keeps running, you don't catch it.

Better architecture (windowed watchdog or "task supervision"):
1. Each critical task has a flag/counter it sets/increments on each iteration.
2. A supervisor task (or the kick logic) checks all task flags before kicking. Only kicks if all are alive.
3. Optional: each task reports liveness within a deadline; missing reports fail the kick.
4. Multi-stage: a software watchdog first (logs the offender, attempts graceful recovery), hardware watchdog as backstop.

Windowed watchdog (e.g., STM32 WWDG): must be kicked within a window. Both too-late *and* too-early trigger reset. Catches tight loops that kick repeatedly.

Crash diagnostics on watchdog reset:
- Save reason (RCC_CSR / RCC_RSR on STM32, similar elsewhere) early in init.
- Persist last task names, last events to a non-volatile RAM region.
- Report to telemetry on next boot.

Real industrial story: a watchdog reset that occurred once a week was a CAN parser hanging on a malformed frame. Without per-task supervision, we couldn't have isolated which task. The supervisor saved minutes-to-days of debugging.

### Q6.9 — Explain how an MPU (memory protection unit) is used on an MCU and what problems it solves.

An MPU is a simplified MMU: it defines memory regions with attributes (RWX, cacheability) and faults on violations. No translation, no virtual memory.

Use cases:
1. **Stack overflow detection.** Region just past the stack marked no-access; overflow → MemManage fault. Better than pattern-painting because it's immediate.
2. **NULL pointer detection.** Region around 0x0 marked no-access. Catches `*p` where `p` is uninitialized to 0.
3. **Code/data separation.** Mark `.text` as RX, `.data`/`.bss` as RW non-executable. Prevents stack-smash exploits from running injected code on the stack.
4. **Task isolation in RTOS.** Each task has its own MPU configuration; can't access other tasks' RAM. FreeRTOS-MPU and Zephyr support this. Limits damage from buggy tasks.
5. **Peripheral access control.** Restrict which tasks can talk to which peripherals.

Limitations vs. MMU:
- Fixed number of regions (usually 8 or 16). Configurations are coarse.
- No virtual addresses; regions are physical.
- Switching configurations is fast but not free; per-task switching has overhead.

Many MCU codebases ship without MPU enabled because it's "complicated." Worth the effort for safety-critical or security-sensitive products.

### Q6.10 — Explain the difference between cooperative and preemptive scheduling. What are the implications for shared data?

Cooperative: tasks yield voluntarily. No preemption. A task runs until it explicitly suspends/yields/blocks. Predictable: data is safe between yield points without locks (a single CPU; no preemption means no race within a task between yield points).

Preemptive: scheduler can interrupt a task at any time on a tick or higher-priority wakeup. Tasks can be suspended mid-instruction. Need locks for any cross-task shared data.

Implications:
- **Coop simpler for shared data.** Just don't yield between read and write of a shared structure.
- **Coop riskier for latency.** A task that doesn't yield (or yields slowly) starves everyone.
- **Preemptive needs careful locking.** Mutexes, atomics, careful design.

Linux and FreeRTOS are preemptive. Some bare-metal RTOSes are coop (Protothreads, some lightweight schedulers). Zephyr supports both modes per-thread.

A common pattern: preemptive between threads (real ISR latency requirements), coop within a thread's set of tasks (a thread's main loop dispatching to handlers without preemption between them).

### Q6.11 — What are atomic operations, and what does C11/C++11 give you on a Cortex-M?

Atomic operations are read-modify-writes that complete without interruption — no other observer sees a partial state. Built on hardware primitives:
- Cortex-M3+ has LDREX/STREX (load/store exclusive). Read with LDREX, modify, attempt STREX; if anyone else wrote in the interim, STREX fails and you retry.
- Cortex-M0/M0+ lacks these. Atomics fall back to disabling interrupts globally — coarse but works.

C11 `<stdatomic.h>` and C++11 `<atomic>` provide:
- `atomic_load`, `atomic_store`, `atomic_compare_exchange_weak`, `atomic_fetch_add`, etc.
- Memory orderings: `relaxed`, `acquire`, `release`, `acq_rel`, `seq_cst`. Map to DMB/DSB/ISB barriers.

For embedded:
- Ring buffer indices: `atomic<uint16_t>` with release/acquire pairs.
- Reference counting: `atomic_fetch_sub` with `acq_rel`.
- Lock-free flags: `atomic_flag` test_and_set.

Caveats:
- On M0/M0+, atomics are heavy (interrupt mask). Avoid in hot ISRs.
- Multi-core ARM (e.g., dual-core RTxx, A+M heterogeneous) needs cache-coherent regions or explicit cache management.
- Atomic on a struct that doesn't fit in a machine word may use a hidden mutex (`atomic<BigStruct>`) — slow.

### Q6.12 — What is interrupt latency, and what factors determine the worst case on Cortex-M?

Interrupt latency: time from peripheral asserting IRQ line to first instruction of the handler executing.

Factors:
1. **NVIC overhead.** Cortex-M defines 12 cycles for exception entry (auto-save, vector fetch, pipeline refill). Roughly fixed.
2. **Higher-priority ISRs in progress.** Worst case = sum of higher-priority ISRs that could be running.
3. **Interrupts disabled regions.** Any code that does `__disable_irq()` extends the worst case by its longest critical section.
4. **Bus contention.** If the CPU is doing flash reads, DMA might steal the bus.
5. **Tail-chaining.** If another interrupt is pending when this one finishes, Cortex-M skips the un-stack/re-stack and goes straight in. *Reduces* latency. Good.
6. **Late-arrival.** A higher-priority IRQ arriving during entry of a lower one redirects to the higher. Reduces latency for the higher.
7. **FPU lazy stacking.** First FP-using ISR pays the FPU save cost.
8. **Cache and memory wait states.** On Cortex-M7 with caches, cold instruction fetches add cycles.

To minimize:
- Keep critical sections (`__disable_irq` blocks) tiny and short.
- Set NVIC priorities correctly so latency-critical IRQs preempt others.
- Place ISR handlers in TCM or SRAM (faster than flash on M7).
- Use `vector_table_in_ram` for fastest fetch.

### Q6.13 — Real-world scenario: an STM32-based product is hitting a HardFault during heavy CAN traffic. Walk through diagnosis.

1. **Capture fault state.** Implement a HardFault handler that dumps:
   - SCB->HFSR, CFSR, MMFAR, BFAR, AFSR (fault status registers).
   - The stacked PC, LR, R0-R3 (auto-saved context — points at the offending instruction).
   - Active task name and stack pointer (RTOS).
   - The full register set if you can.
2. **Decode CFSR.** Forced fault? Bus fault on a load? Imprecise (write buffer)?
3. **PC analysis.** With the addr2line tool against the .elf, find the function and line.
4. **Common patterns under load:**
   - **Stack overflow** — heavy ISR nesting blowing past stack. Check stack high-water marks.
   - **CAN ISR re-entered before completion** — usually NVIC priority misconfig.
   - **Bus fault on an unmapped region** — DMA misconfigured, writing past buffer.
   - **Imprecise fault** — write to a register that's clock-gated. Common with sleep modes; clock turned off but write enqueued.
   - **Null/dangling pointer** — message queue overrun, freed buffer accessed.
5. **Reproduce in lab.** Inject fault: increase CAN load, run for long durations, watch for the trigger.
6. **Tools.** Segger Ozone for live debug. SWO trace for non-intrusive logging. Hardfault inspection scripts.
7. **Hardening.** MPU regions to catch overflows precisely. Run-time stack monitor. Bus fault enabled (some firmware leaves only HardFault enabled — enable BusFault, MemManage, UsageFault separately for better diagnosis).

Real story: a system hardfaulted under high CAN load because a queue was overflowing and the producer ISR was trying to log via printf to a UART that was ring-buffer-locked. Fix: drop on overflow with a counter, never block in ISR.

### Q6.14 — Compare static and dynamic memory allocation in an RTOS context. How does FreeRTOS support both?

Static: all kernel objects (tasks, queues, semaphores) are pre-allocated, typically in `.bss`. Sizes known at compile time. No fragmentation possible. FreeRTOS supports via `xTaskCreateStatic`, `xQueueCreateStatic`, etc., with `configSUPPORT_STATIC_ALLOCATION=1`. You provide the storage buffer.

Dynamic: kernel objects allocated from heap. FreeRTOS provides 5 heap implementations:
- **heap_1**: simple, no free. Allocate-only.
- **heap_2**: best-fit free, can fragment.
- **heap_3**: wraps libc malloc/free.
- **heap_4**: best-fit with coalescing. Reduces fragmentation. Most common choice.
- **heap_5**: heap_4 across multiple non-contiguous regions.

When to choose:
- Safety-critical: static. Predictable, analyzable, no fragmentation. MISRA-friendly.
- Resource-constrained or one-shot: dynamic with heap_4 if you allocate at boot only.
- Generally avoid dynamic during normal operation in long-running embedded systems.

A pragmatic mix: static for kernel objects, dynamic only for app-level scratch buffers in well-defined arenas, and a hard rule that no dynamic allocation happens after `init` completes.

### Q6.15 — Explain DMA, and design considerations for using it correctly with an RTOS.

DMA is a controller that moves data between memory and peripherals (or memory-to-memory) without CPU involvement. CPU configures source, destination, length, mode; starts transfer; goes back to other work. DMA controller signals completion via interrupt.

Wins:
- Free the CPU during long transfers (UART RX of a 1KB buffer).
- Higher throughput (DMA can be back-to-back, CPU can't always).
- Lower power (CPU sleeps during transfer).

Considerations:
1. **Cache coherency.** On Cortex-M7 with caches, DMA bypasses the cache. Before a DMA write to memory, the CPU must clean the cache (push dirty lines to memory). After a DMA write *into* memory, CPU must invalidate its cache (otherwise reads get stale lines). Or place DMA buffers in non-cacheable memory regions.
2. **Buffer alignment.** Many DMA controllers require alignment matching transfer size. 32-bit transfer → 4-byte alignment.
3. **Buffer lifetime.** Buffer must remain valid throughout the transfer. Stack-allocated DMA buffers are a classic bug — function returns, stack reused, DMA writes to corrupted memory.
4. **MPU regions.** DMA-accessible regions need correct attributes.
5. **Endianness/swap.** Some DMAs do byte swaps. Configure deliberately.
6. **Multiple masters.** SoC has CPU, DMA, sometimes other DMAs. Bus arbitration; throughput is shared.
7. **Errors.** DMA errors (FIFO under/overrun) need handling.

RTOS pattern: DMA hands off via callback or semaphore to a worker task. Don't do heavy work in the DMA-complete ISR.

### Q6.16 — Explain the Observer pattern in embedded C++ and its trade-offs vs. a plain C callback table.

Observer pattern: subjects maintain a list of observers; on event, iterate and notify.

In C++:
```cpp
class Observer { public: virtual void onEvent(Event) = 0; };
class Subject {
    std::vector obs;
public:
    void attach(Observer*); void detach(Observer*);
    void notify(Event e) { for (auto* o : obs) o->onEvent(e); }
};
```

Pros:
- Type-safe, explicit interface.
- RAII-friendly: observer detaches in destructor.
- Multiple observers natively.

Cons:
- Vtable indirection per call.
- `std::vector` heap allocation. Use `etl::vector` or a static array on embedded.
- Iteration during notify when an observer detaches — needs care (snapshot the list, or use careful indexing).

C callback table:
```c
typedef void (*event_cb)(void* ctx, Event e);
struct cb_entry { event_cb cb; void* ctx; };
struct subject { struct cb_entry callbacks[MAX_OBSERVERS]; };
```

Pros:
- No vtable, lower overhead.
- Static layout, no allocation.

Cons:
- Type erased (`void*` ctx). Easy to mess up casts.
- Manual lifecycle management.

For most modern C++ embedded code, the Observer with `etl` containers is clean. For tight ISRs or interop with C drivers, callback tables are fine.

---

## 7. Communication Protocols

### Q7.1 — Compare I2C and SPI. When do you pick each?

| Aspect | I2C | SPI |
|---|---|---|
| Wires | 2 (SCL, SDA) | 4+ (SCK, MOSI, MISO, CS per slave) |
| Speed | 100kHz / 400kHz / 1-3.4MHz | 1-50+ MHz typical |
| Topology | Multi-drop, addressable | Master + per-slave CS |
| Half/full duplex | Half | Full |
| ACK | Yes (per byte) | No |
| Multi-master | Supported (with arbitration) | Rare |
| Pull-ups | Required (open-drain) | Push-pull, no pull-ups |
| Complexity | More (addresses, ACK, clock stretching) | Simple shift register |

Pick I2C when:
- Many devices on the bus, low pin count critical.
- Speed isn't critical (sensors, EEPROMs, RTCs).
- Devices need addresses (multi-drop).

Pick SPI when:
- High speed needed (display, fast ADC, flash memory).
- Few devices.
- Streaming data (full-duplex helpful).

Subtleties:
- I2C clock stretching: slave holds SCL low while it processes. Some controllers don't handle this gracefully — read the errata.
- SPI mode (CPOL, CPHA): four combinations defining clock polarity and which edge samples. Mismatched modes silently corrupt data.
- I2C address conflicts: 7-bit address space is small. Solving via address pins, mux ICs (TCA9548), or moving to I2C with 10-bit addresses.

### Q7.2 — Walk through an I2C read transaction at the wire level. What happens with the START/STOP, addresses, and ACK?

Typical "read N bytes from register X of device at address A":

1. **START condition.** SDA falls while SCL high. Bus claimed.
2. **Address byte 1 (write).** A << 1 | 0. Master writes to slave first to send the register address.
3. **ACK.** Slave pulls SDA low during 9th clock to ACK.
4. **Register address byte.** Master sends X.
5. **ACK.** Slave ACKs.
6. **REPEATED START.** SDA falls while SCL high again, no STOP between — keeps the bus.
7. **Address byte 2 (read).** A << 1 | 1. Now in read direction.
8. **ACK.** Slave ACKs.
9. **Data bytes.** Slave clocks out data on each SCL pulse. After each byte, master ACKs (continues) or NACKs (last byte).
10. **STOP.** SDA rises while SCL high.

Key details:
- Repeated START keeps the bus locked — no other master can grab it. Required for atomic register-then-read.
- A NACK at step 3 means slave doesn't exist or isn't responding (wrong address, no power, bus stuck).
- Clock stretching can happen at any step where the slave needs time.

Scope-trace pattern recognition: SDA/SCL together, START as SDA falling edge, STOP as SDA rising edge while SCL is high. Bus voltage should rest at Vcc; if not, check pull-ups.

### Q7.3 — Explain UART vs USART. What's the difference, and when does it matter?

- **UART** (Universal Asynchronous Receiver/Transmitter) — async serial. No clock signal; both ends use baud-rate-matched timing.
- **USART** (Universal Synchronous/Asynchronous) — superset. Can run synchronously with a clock signal.

In practice on STM32 etc., the peripheral is called USART even when used in UART mode. Async is the dominant use.

Async framing: start bit (low) → 7-9 data bits → optional parity → 1-2 stop bits (high). Baud rates: 9600, 115200, up to a few Mbaud on modern parts.

Sync mode (rare): adds a clock pin. Used for inter-chip communication where you want UART semantics with no baud-rate drift.

Where it matters:
- Long cables: async is more tolerant of timing jitter.
- Inter-MCU on a PCB: sync can be more reliable at high rates.
- ISO 7816 smart cards, IrDA, LIN: USART supports these via mode flags.

Most embedded engineers use the terms interchangeably but the formal distinction is sync vs. async support.

### Q7.4 — What is RS-485 and how does it differ from RS-232 and TTL UART?

- **TTL UART**: 0V/3.3V (or 5V) logic. Single-ended. Short distances (<1m on a board). Used between MCU and modules.
- **RS-232**: ±12V (typically) single-ended, inverted logic. PC serial port. Up to ~15m, 1 device.
- **RS-485**: differential (A and B lines). Multi-drop (up to 32+ devices). Up to 1200m at lower speeds. Half-duplex (2-wire) or full-duplex (4-wire).

RS-485 advantages:
- Differential noise immunity → long cables, industrial environments.
- Multi-drop → bus topology.
- Common in: industrial automation (Modbus), building automation (BACnet), older PROFIBUS.

How it differs from a bus protocol like CAN:
- RS-485 is just the physical layer. The protocol on top (Modbus RTU, DMX512, BACnet MS/TP) is separate.
- No built-in arbitration; requires master-slave or token-passing protocol.
- No ACKs at PHY level.

Implementation gotchas:
- Termination: 120Ω at each end of the bus.
- DE pin: enables transmitter. Must be deasserted promptly after transmission to avoid blocking the bus.
- Bias resistors: keep idle line in a known state.
- Slow turnaround between TX and RX kills throughput; some chips have auto-direction.

### Q7.5 — Explain CAN at the protocol level. What's an arbitration ID, what's the bit-stuffing, and how does ACK work?

CAN frame structure:
1. **SOF**: dominant bit.
2. **Arbitration field**: 11-bit (CAN 2.0A) or 29-bit (CAN 2.0B) identifier + RTR.
3. **Control field**: IDE, reserved, DLC (data length 0-8 for classic, up to 64 for CAN-FD).
4. **Data field**: 0-8 bytes (classic) or 0-64 bytes (CAN-FD).
5. **CRC**: 15-bit or longer.
6. **ACK slot**: transmitter sends recessive; any receiver pulls dominant if it received correctly.
7. **EOF**: 7 recessive bits.

Arbitration:
- Multiple nodes can transmit simultaneously. CSMA/CR (collision resolution): each node monitors the bus while transmitting; if it sends recessive and the bus is dominant, it lost arbitration and stops.
- Lower numerical ID = higher priority (more dominant bits early).
- Lossless: the winner continues; the loser retries on next idle.

Bit stuffing:
- After 5 consecutive same-polarity bits in transmitter output, a stuffing bit of opposite polarity is inserted. Receiver removes it.
- Ensures clock recovery — receivers re-sync on edges.
- Only applies to fields up through CRC (not EOF).

ACK:
- Transmitter outputs recessive in the ACK slot; expects to see dominant from at least one receiver.
- ACK doesn't mean "received correctly by anyone in particular" — it means "at least one node CRC-matched."
- No ACK = transmitter retries, possibly indefinitely. Bus-off after enough errors.

Common gotcha: a single-node bus (one MCU, no other listeners) will never get ACK. Common when bench-testing without a transceiver loopback or another node.

### Q7.6 — What is J1939, and how does it sit on top of CAN?

J1939 is an SAE standard for heavy-duty vehicle networks (trucks, agricultural, off-highway — i.e., your day job at John Deere). It's a higher-layer protocol over CAN 2.0B (29-bit IDs).

Key elements:
- **PGN (Parameter Group Number)** — embedded in the 29-bit ID. Identifies the message type. ~65k space, divided into broadcast (PDU2) and addressed (PDU1).
- **Source address (SA)** — last 8 bits of the ID. Identifies the sender ECU.
- **Priority** — top 3 bits of the ID. Affects CAN arbitration.
- **SPN (Suspect Parameter Number)** — identifies a parameter within a PGN. Standardized in SAE J1939-71.
- **Address claim** — at startup, ECUs claim their address; conflict resolution defined.
- **Transport Protocol (TP)** — for messages > 8 bytes:
  - **BAM (Broadcast Announce Message)** — broadcast multi-packet.
  - **CMDT (Connection Mode Data Transfer)** — point-to-point with flow control (RTS/CTS).
- **NM (Network Management)** — address claim, request messages.

J1939 on Linux:
- Kernel SocketCAN supports J1939 natively (`CAN_J1939`). You write to `socket(AF_CAN, SOCK_DGRAM, CAN_J1939)`, set source/destination addresses, send/receive. The kernel handles TP, address claim.
- `j1939d` userspace daemon manages address claim.

Real-world bug pattern: BAM transport requires precise inter-frame timing (50ms typical between TP.DT packets). On a busy CPU, missing a window causes the receiver to abort. Schedule TX accordingly.

Your J1939 library project (libj1939 / j1939d / SocketCAN bridge) sits at the Linux ecosystem gap where kernel CAN_J1939 exists but a clean userspace API and PGN-aware tools are sparse.

### Q7.7 — Compare CAN classic and CAN-FD. What changed and why?

CAN classic:
- 8-byte max payload.
- Up to 1 Mbit/s.
- Single bit-rate throughout the frame.
- 15-bit CRC.

CAN-FD ("Flexible Data-rate"):
- Up to 64-byte payload.
- Two bit-rates per frame: arbitration phase at classic speeds (e.g., 500kbps), data phase at higher speeds (up to 8 Mbps practical, more on the spec).
- 17 or 21-bit CRC.
- BRS (Bit Rate Switch) bit signals the rate change.

Why:
- Modern vehicles have outgrown 1 Mbit. ECUs send larger payloads (firmware updates, complex diagnostics).
- 8-byte limit caused multi-frame fragmentation overhead. 64-byte cuts most diagnostic messages to single frame.

Compatibility:
- CAN-FD nodes can speak classic, but a classic-only node on a CAN-FD-active bus throws errors and goes bus-off. Mixed buses are not safe.
- Transceivers must be CAN-FD-rated for the higher data-phase rate.

Adoption: heavy vehicles (J1939) are slowly transitioning. CAN XL (newer, even more bandwidth) is on the horizon for some segments.

### Q7.8 — Explain Ethernet from PHY to socket. What are the layers and where do they live in an embedded system?

Layers:
1. **PHY (physical layer).** External chip (or integrated). Encodes/decodes 10/100/1000BASE-T over twisted pair. Auto-negotiates speed/duplex. MDIO management interface for configuration.
2. **MAC (media access control).** Inside SoC. Frames Ethernet packets (preamble, dest, src, type, payload, CRC), handles collision detection (legacy half-duplex CSMA/CD), DMAs frames to/from memory.
3. **Driver.** SoC-specific MAC driver. Manages descriptor rings, IRQs, and exposes a netdev interface to the kernel.
4. **Network stack.** L3 (IP) routes packets, L4 (TCP/UDP) provides streams or datagrams, sockets API exposes them to userspace.
5. **Application.** Reads/writes sockets.

In embedded Linux, this is largely standard. On bare-metal or RTOS, you need an embedded TCP/IP stack: lwIP, FreeRTOS+TCP, NUTNet. They re-implement IP/TCP/UDP with smaller footprint.

PHY-MAC connection: MII / RMII / RGMII / GMII. Choice depends on speed and pin count. Layout-sensitive — RGMII at gigabit needs careful skew control.

Real bring-up issues:
- Wrong RGMII delay tuning → packets sometimes work, sometimes not. Often fixed by adjusting `tx-internal-delay-ps` and `rx-internal-delay-ps` in DT.
- PHY address mismatch — MDIO address differs by board; DT's `reg` property must match.
- Power sequencing — PHY needs power and reset before MAC tries to talk to it.

### Q7.9 — Compare Wi-Fi and BLE. When do you use each in IoT?

Wi-Fi:
- 802.11 a/b/g/n/ac/ax. 2.4/5/6 GHz.
- Up to gigabits/s.
- Tens of mA in active, less in standby. Big batteries or AC.
- IP-based; integrates with internet stack natively.
- Range: 30-100m indoor.
- Need: AP, credentials, sometimes provisioning UX.

BLE (Bluetooth Low Energy):
- 2.4 GHz.
- ~1 Mbps PHY (up to 2 Mbps with BLE 5).
- Microamps in advertising/sleep, mA in connection. Coin-cell capable.
- Not IP; uses GATT (services + characteristics).
- Range: ~10m typical, longer with directional antennas / coded PHY.
- Pairing/bonding; peripheral/central roles.

Decisions:
- Battery-powered sensor reporting once a minute → BLE (or LoRaWAN, NB-IoT for longer range).
- Always-on hub, video, big data → Wi-Fi.
- Smartphone interaction (config, control) → BLE often wins (no AP needed).
- Mesh/many-to-many → Thread, Zigbee, BLE Mesh — protocol stacks above BLE/802.15.4.

Hybrid: Wi-Fi for cloud, BLE for provisioning. Common pattern in consumer IoT.

### Q7.10 — Explain HTTP vs MQTT. When do you pick each on a constrained device?

HTTP:
- Request/response. Stateless. Verbose headers.
- Each request is a TCP connection (or kept-alive). TLS handshake adds 5-20KB and a few RTTs.
- Pull-based; the device polls. No native push.
- Massive ecosystem; works through any firewall.

MQTT:
- Pub/sub. Stateful (persistent connection to broker).
- Tiny header (2 bytes minimum). Designed for constrained networks.
- Push from server to device via subscriptions. Bidirectional via topic.
- Need a broker (Mosquitto, EMQX, AWS IoT, HiveMQ).
- QoS 0/1/2 for reliability levels.

For constrained device sending telemetry every minute:
- HTTP: 1 KB+ of overhead per request, periodic polling. Wakes radio frequently if you also want commands.
- MQTT: persistent connection (battery cost trade-off), tiny per-message overhead, server can push.
- MQTT-SN / CoAP: even smaller, designed for sensor networks.

For one-shot OTA download:
- HTTP. CDN-friendly, range requests for resumable downloads, broad support.

Hybrid pattern: MQTT for telemetry/commands, HTTP/HTTPS for firmware fetches.

### Q7.11 — A team's TCP-based device intermittently drops connections in the field. Walk through diagnosis.

1. **Reproduce.** Find the conditions: time of day, network type, traffic level, distance from AP. Capture with `tcpdump` on the device or a port mirror at the router.
2. **Identify the side terminating.** RST from server? FIN from client? Timeout (no FIN, just silence)? `tcpdump` with `-S` to see absolute sequence numbers.
3. **Common causes:**
   - **NAT timeout.** Carrier or home routers drop idle TCP after 1-30 minutes. Enable TCP keepalives (`SO_KEEPALIVE`) with short intervals, or app-level pings.
   - **Wi-Fi roaming.** Device moves between APs; old connections die. Re-establish on link change.
   - **DHCP lease churn.** IP address changes; existing sockets break.
   - **Server-side load balancer health checks** kicking idle clients.
   - **Buffer overflow on slow consumer.** TCP backs off, RST eventually.
   - **MTU/PMTUD blackhole.** ICMP "fragmentation needed" filtered; TCP retransmits forever.
   - **Cellular modem** specific: PDP context drops, modem resets without notifying app.
4. **Mitigations:**
   - TCP keepalive: `setsockopt(SO_KEEPALIVE)` plus `TCP_KEEPIDLE`, `TCP_KEEPINTVL`, `TCP_KEEPCNT`.
   - App-level heartbeat with timeout.
   - Reconnect with backoff on disconnect.
   - Persistent message queue so disconnections don't lose data.
5. **Use MQTT or QUIC** if you control both ends and the protocol layer matches your needs better than raw TCP.

### Q7.12 — Explain TLS at a high level and the considerations for TLS on a constrained MCU.

TLS provides confidentiality, integrity, and authentication over TCP (or DTLS over UDP). Handshake establishes a session key:
1. Client hello (ciphers, extensions, random).
2. Server hello (chosen cipher, certificate).
3. Key exchange (ECDHE typically).
4. Authentication (client may also auth via cert).
5. Finished — symmetric session begins. AES-GCM, ChaCha20-Poly1305, etc.

Constraints on MCU:
- **RAM.** TLS handshake uses 20-80KB of working memory depending on stack and cipher. Larger if RSA with 4096-bit keys.
- **Flash.** mbedTLS or wolfSSL build with selected ciphers: 50-200KB.
- **CPU.** Asymmetric crypto (RSA, ECDSA) is slow. ECDHE-ECDSA-AES-GCM is the modern fast choice. Hardware acceleration helps (CRYP, HASH peripherals on STM32H7, AES on ESP32).
- **Cert store.** Need root CAs in flash. Avoid letting them be modified post-deploy.
- **Time.** TLS validates cert validity periods. MCUs without RTC need NTP first or rely on a sane initial time.
- **OCSP / revocation.** Hard on constrained devices. Pinning the server cert simplifies.

Libraries:
- **mbedTLS** (formerly PolarSSL). Most popular for embedded. ARM owned.
- **wolfSSL.** Commercial, smaller, FIPS-certified options.
- **BearSSL.** Tiny, constant-time-by-default, no dynamic allocation.

For your IoT pipeline: pre-shared keys for tightly scoped devices, certs (with mutual TLS) for higher security. AWS IoT uses mTLS; Azure IoT supports both.

### Q7.13 — Real-world scenario: SPI bus to a high-speed ADC sometimes corrupts samples. Walk through diagnosis.

1. **Confirm the symptom.** What's "corrupt"? Bit flips? Byte misalignment? Periodic glitches?
2. **Scope it.** Probe SCK, MOSI, MISO, CS. Look for:
   - **Signal integrity.** Ringing, undershoot, slow edges. SPI clocks above 20 MHz are demanding.
   - **CS timing.** Setup/hold violations against SCK.
   - **Polarity/phase mismatch.** Check waveforms against datasheet timing diagrams.
3. **Layout review.** SCK trace length, GND reference, parallel runs to noisy signals. SPI buses are often laid out casually and pay for it at speed.
4. **Driver-side issues:**
   - DMA buffer alignment.
   - Cache coherency (Cortex-M7).
   - Wrong word size (8/16-bit) configured.
   - Clock divisor wrong.
5. **Slave issues:**
   - ADC sampling timing requires specific delays after CS assertion. Some ADCs require N clock cycles before reading.
   - Power supply noise on the ADC's analog rail showing up as data corruption (LSB jitter).
6. **Reproduce on bench.** Reduce SCK to half — does it stop? Bus is at the edge.
7. **Termination.** SPI rarely needs termination at <50 MHz, but at 50+ MHz, source-series resistors (~22Ω) on SCK and MOSI tame ringing.
8. **Crosstalk.** Adjacent unshielded signals coupling. Move them.

I've seen this caused by: a missing decoupling cap on the ADC analog supply (LSB noise), an SPI clock running at 40 MHz where the ADC was rated at 20 MHz despite the part number (wrong ADC variant), and once by an oscilloscope ground lead loop adding noise.

---

## 8. Schematic Capture and PCB Design

### Q8.1 — Compare KiCad, Altium, and EAGLE for embedded product PCB work.

| Aspect | KiCad | Altium Designer | EAGLE (now Fusion Electronics) |
|---|---|---|---|
| Cost | Free, open source | $5-9k/seat/year | Subscription via Fusion 360 |
| Learning curve | Moderate | Steep | Easy at first |
| Library mgmt | Improving (KiCad 7+) | Very strong | OK |
| Schematic | Solid | Excellent | Solid |
| PCB editor | Solid (push & shove from 6.0) | Best-in-class router | Limited |
| Sim | Ngspice integrated | Built-in mixed-signal | Limited |
| ECAD/MCAD | STEP export, decent | Native MCAD link | Tied to Fusion 360 |
| Industry usage | Open source, hobby, growing pro | Aerospace, automotive, RF | Hobby, education, some prod |
| File format stability | Open, future-proof | Proprietary, stable | Proprietary, transitioning |

KiCad: production-capable for most embedded boards. Strong recent improvements (multi-channel, hierarchical schematics, push & shove routing). Free is a huge competitive moat.

Altium: dominant in serious EE shops. Worth the price for complex boards (high-speed, RF, dense FPGA, large team library management).

EAGLE: legacy hold among hobbyists, transitioning to Fusion Electronics. Less common in new pro work.

For a startup or small team: KiCad. For a 50-person hardware team in automotive/aerospace: Altium.

### Q8.2 — Explain the role of decoupling capacitors. How do you choose values and placement?

Decoupling caps provide local charge for fast current transients that the power supply can't deliver because of inductance in the supply traces. Without them, IC supply pins see voltage dips during switching, causing logic errors, EMI, and crashes.

Choosing values:
- **Bulk** (10-100 µF, typically tantalum or large ceramic). Handle low-frequency bulk current, near regulator and at the input of each board section.
- **Decoupling proper** (0.1 µF / 100 nF, X7R or X5R ceramic). One per supply pin of each IC. Handles MHz-range transients.
- **High-frequency** (1-10 nF, sometimes parallel with the 100 nF). For high-speed parts (FPGAs, fast DSPs), tackle transients up to 100+ MHz.
- For really high-speed (multi-GHz, DDR4): even smaller (10-100 pF), and placement matters more than value.

Placement rules:
1. **As close to the IC pin as possible.** Ideally on the same side, smallest loop area between cap-pin-via-ground.
2. **Vias to power/ground planes are short.** Keep the loop tight. Long traces or shared vias add inductance and defeat the cap.
3. **Per supply pin.** Don't share one cap among multiple pins of the same rail unless very low-speed.
4. **Stitch grounds.** Adjacent ground vias on either side of the cap to keep the return loop tight.

Modern thinking: a single 0.1 µF MLCC has a self-resonant frequency around 10-20 MHz; above that, it's inductive. Datasheets often recommend specific cap combinations for power integrity. For high-speed, simulate with a PDN tool (Saturn PCB Toolkit, Cadence Sigrity).

### Q8.3 — What are ground bounce and power supply noise, and what design choices mitigate them?

Ground bounce: when a digital output switches, it pulls current through the IC's ground path. The path's parasitic inductance (`L * di/dt`) creates a transient voltage difference between the IC's internal ground and board ground — the "bounce." Causes false transitions on quiet outputs, EMI radiation, mis-clocking.

Mitigations:
1. **Solid ground plane.** Continuous, unbroken pour. Largest possible.
2. **Local decoupling** (above) to provide instantaneous current locally instead of through the supply path.
3. **Short, low-inductance return paths.** Vias near the cap, not at the far end of a trace.
4. **Slew-rate control.** Modern MCUs have configurable output drive strength and slew rate. Reduce for non-critical signals.
5. **Series termination** on long, fast traces to reduce reflections (which cause `di/dt` spikes).
6. **Avoid large output banks switching simultaneously.** If 32 GPIOs change at once, ground bounce scales.

Power supply noise: similar mechanism on Vcc. Same mitigations. Plus:
- **Separate analog and digital grounds**, joined at one point or via a ferrite. (Modern thinking: solid ground is often better than split ground for most boards. Split ground only when you really know what you're doing — analog signals crossing the split are problematic.)
- **Linear regulators near sensitive analog blocks.** PSRR matters.
- **Clean analog supplies** with LC filters.

For high-speed digital with a single 4-layer board: solid GND plane on layer 2, dedicated power plane on layer 3. For really clean analog, bring in a separate LDO with its own ferrite-filtered input.

### Q8.4 — Explain controlled impedance, why it matters for differential pairs, and how you specify it.

Impedance is the AC resistance to a signal traveling on a transmission line. For a stripline or microstrip on a PCB, it's set by trace width, trace-to-ground distance, dielectric constant, and trace thickness.

When does it matter:
- Signals fast enough that the trace is "long" relative to the rise time. Rule of thumb: if length > rise_time × 0.5 (in mm with rise time in ns and assuming 6 mil/ns velocity), treat as transmission line.
- USB (90Ω diff), Ethernet (100Ω diff), HDMI (100Ω diff), DDR (varies), PCIe (85-100Ω diff), MIPI CSI/DSI (100Ω diff), SDIO at high speed.

Differential pairs:
- Driven complementarily; impedance is between the pair (differential impedance).
- Common-mode rejection: noise that hits both equally is rejected at the receiver.
- Length-matching: skew between the lines causes intra-pair phase difference, hurts common-mode rejection.
- Spacing: tight spacing for strong coupling (better common-mode rejection); loose for less crosstalk to neighbors.

Specification to fab:
- Tell the fab: "100Ω differential, 50Ω single-ended" with tolerances (typically ±10%).
- Provide a stack-up (layer thicknesses, dielectric constants, copper weights). Fab adjusts trace width to hit your impedance targets.
- Fab returns: "for 100Ω diff, use 0.15mm trace width, 0.10mm gap, on 0.20mm dielectric" or whatever.

Tools: Saturn PCB Toolkit (free), KiCad's calculator, Polar Si9000 (industry standard for fabs).

Bring-up bug pattern: wrong stack-up assumption → impedance off by 20% → USB enumerates intermittently. Fixes are expensive (respin or stack-up change).

### Q8.5 — What is a return path, and what happens when it's broken?

For any signal current, there's an equal return current. The return path follows the path of least *impedance* — for high-frequency signals, that's the path of least *inductance*, which is directly under the signal trace on the nearest ground plane.

Broken return path scenarios:
- Signal trace crosses a slot or split in the ground plane. Return current must detour around — large loop area, inductance, EMI.
- Signal jumps layers via a single via, with no ground via nearby for the return current.
- Mixed-signal split ground with a digital signal crossing it (analog ground is "elsewhere" — return current has to find a way).

Consequences:
- **EMI radiation.** Loop antenna at the loop area.
- **Crosstalk.** Return currents of multiple signals in the same loop interact.
- **Signal integrity degradation.** Inductance + reflections.

Fixes:
- **Don't cross splits.** Plan layer transitions over solid ground.
- **Stitching vias.** When a signal vias to another layer, place a ground via nearby so return current can also via.
- **Solid ground plane.** No splits unless absolutely necessary, and signals do not cross them.

Real-world: a board passing FCC EMI in one revision and failing in the next. Diff: a single trace was rerouted across a power-ground split that was created when adding a separate analog supply. Moving the trace fixed it. Lesson: any time you change power topology, audit signal paths against return paths.

### Q8.6 — Explain via types (through, blind, buried, microvia) and when you'd use each.

- **Through-hole via.** Drilled through entire board. Cheapest. Default.
- **Blind via.** From outer layer to an inner layer; doesn't go through. Saves real estate on the opposite side.
- **Buried via.** Between two inner layers; doesn't reach either outer layer.
- **Microvia.** Laser-drilled, very small (~75-100 µm), typically only span one layer. Used for HDI (high-density interconnect) with fine-pitch BGAs.

When to use:
- Most boards: through-hole only. Cheap, well-understood.
- Fine-pitch BGAs (≤0.5mm pitch) — usually need microvias to escape. 0.4mm BGAs almost always.
- Dense designs where inner-layer routing is needed but outer layers are crowded.
- HDI boards: 3+1+3 stack-ups with microvias on outers, regular vias inside.

Costs:
- Blind/buried add fab steps (sequential lamination). 30-100% cost increase.
- Microvias add laser drilling. Higher cost, fab dependent.
- Stacked vs. staggered microvias: stacked is denser but more constrained on stackup.

Rule of thumb: don't reach for blind/buried unless routing demands it. They're not free, and signing up for them locks you into a more expensive fab.

### Q8.7 — Walk through PCB bring-up of a brand-new board. What's your sequence?

Pre-power:
1. **Visual inspection.** Solder bridges, tombstoning, missing parts, rotated components, polarized parts (electrolytics, diodes, ICs) correctly oriented.
2. **DRC against schematic.** Verify final assembly matches schematic (BOM check).
3. **Continuity / shorts.** Multimeter from each power rail to GND. Should be high-impedance (caps charge briefly). Shorts → don't power.
4. **Diode/junction checks.** Verify regulator inputs/outputs, sensitive part orientations.

First power-up:
5. **Current-limited supply.** Set current limit at a safe value (twice expected idle).
6. **Bring up rails one at a time** if possible, or in the designed sequence. Verify each rail's voltage at multiple points.
7. **Watch for excessive current.** If something draws much more than expected, kill power immediately.
8. **Temperature sweep.** Touch ICs (carefully). Hot regulator = high current somewhere. IR camera if available.

Post-power:
9. **Clock check.** Scope the main oscillator. Without clock, nothing else works.
10. **Reset behavior.** Verify reset asserted/deasserted as expected.
11. **JTAG / SWD connection.** Try debugger probe. If chip alive, you can read flash, identify part.
12. **Boot to first instruction.** Often via existing firmware blink or vendor demo.
13. **Bring up peripherals incrementally.** UART first (logging), then the rest.

Troubleshooting sequence: power → clock → reset → boot. Each is a hard prerequisite for the next.

Tools: bench supply with current limit, scope with bandwidth ≥ 5x highest signal, multimeter, IR camera, USB-PD load, signal generator if needed, and the schematic + datasheets within arm's reach.

### Q8.8 — Explain DFM (design for manufacturing) considerations for a production PCB.

DFM is about making a board manufacturable at acceptable yield and cost. Core concerns:

1. **Footprint accuracy.** Use IPC-7351 standards. Wrong footprints cause assembly issues — solder bridges, tombstones.
2. **Component spacing.** Adequate clearance for pick-and-place nozzles and rework. 0.5mm minimum between parts typically.
3. **Polarity markers.** Visible silkscreen for polarized parts (diodes, ICs, electrolytics).
4. **Test points.** Probe-able test points on key signals (power rails, clock, important comm lines). 1mm pads, accessible from one side.
5. **Fiducials.** Three (corner) fiducial markers for pick-and-place machine alignment. 1mm copper, no soldermask.
6. **Solder mask and silkscreen.** Mask between fine-pitch pins (avoids bridges). Silkscreen not on pads.
7. **Panelization.** If you have a panel, breakaway tabs or V-grooves. Coordinate with your fab.
8. **Layer count and stack-up.** Standard stack-ups are cheaper. Fab can quote standard much faster.
9. **Drill and pad sizes.** Avoid sub-standard drills. Min annular ring per fab spec (often 0.15-0.2mm).
10. **Edge clearance.** Components and traces away from board edges (1-2 mm minimum).
11. **BGA escape.** Pads accessible without via-in-pad (cheaper) when possible. Via-in-pad needs filling and plating, big cost adder.
12. **Soldermask sliver.** Avoid <0.1 mm slivers of mask between pins; they peel off.

Process: send your design files to the fab/CM for a DFM review before committing to production. They'll catch issues with their specific process. Iterate.

### Q8.9 — What is EMI, and what design choices reduce it?

EMI (electromagnetic interference) is unintentional radiation or conducted noise. Regulatory limits (FCC Part 15, CISPR 32, EN 55032) cap allowed emissions. Most products must pass EMC testing.

Sources:
- Switching power supplies (high di/dt).
- Fast digital signals on long traces (act as antennas).
- Clocks (especially harmonics).
- Cables exiting the box (common-mode currents become radiation).

Design choices to reduce:
1. **Solid ground planes.** Tight return paths reduce loop area.
2. **Decoupling.** Reduces switching transients before they spread.
3. **Slow edges where possible.** Slew-rate control on outputs.
4. **Short, direct routing.** Especially clocks. Avoid stubs.
5. **Spread-spectrum clocking.** Modulate clock frequency by a small percent — spreads emission peak over a band.
6. **Common-mode chokes** on cables exiting the enclosure.
7. **Filtering.** Capacitors, ferrites at I/O.
8. **Shielding.** Metal enclosure, EMI gaskets at seams.
9. **Cable layout.** Bundle signal returns with their signals.
10. **Switcher choice.** Higher switching frequency moves emissions to higher bands (often less restrictive); switcher with built-in compensation/spread.

Practical: pre-compliance scan in your lab before formal EMC test. Near-field probes, spectrum analyzer, and a TEM cell or open-area test site reveal the worst offenders cheaply. Failing formal EMC means lost weeks; pre-compliance saves money.

### Q8.10 — Explain ESD protection at the board level.

ESD (electrostatic discharge): static voltages from contact, sometimes thousands of volts. Damages thin gate oxides on ICs. Standards: IEC 61000-4-2 (8kV contact, 15kV air for product testing).

Protection layers:
1. **TVS diodes** at every connector and exposed I/O. Clamp to safe levels in <1ns. Choose based on signal bandwidth (low-cap TVS for USB/HDMI to avoid signal degradation).
2. **Series resistance** on slow signals (button inputs, GPIOs). 100Ω-1kΩ limits current during ESD.
3. **PCB layout.** Keep ESD-sensitive traces short. Chassis ground vs. signal ground topology — chassis ground catches ESD, signal ground stays clean.
4. **Connector positioning.** Coupled with mechanical: connector body grounded to chassis, ESD shunted there.
5. **Capacitors** to ground at I/O. Provides a low-impedance path for high-frequency ESD components.

Common bug: an internal IC pin that's connectorized externally without TVS. First customer ESD event blows the part. Always TVS on anything user-touchable.

For high-speed I/O: TVS choice is critical. A TVS with too much capacitance breaks the signal. Ultra-low-cap TVS (0.3 pF for HDMI/USB 3) exist; verify the part you choose is rated for the bandwidth.

### Q8.11 — Real-world scenario: your prototype works on the bench but fails in the customer's environment with comm errors. Walk through diagnosis.

1. **Characterize "fails."** What protocol? When? Frequency? Pattern? Get logs and operating conditions.
2. **Differences from bench.** Compare:
   - **Power.** Customer's supply quality, ripple, droop.
   - **Temperature.** Bench is 22°C; customer might be -20°C or 60°C.
   - **EMI environment.** Industrial floor with VFDs, near a radio tx, near an arc welder.
   - **Cabling.** Different lengths, routing, shielding.
   - **Other equipment.** Sharing a bus or supply with noisy gear.
3. **Reproduce in lab.** Build a setup mimicking customer conditions:
   - Variable temperature (chamber).
   - Inject noise via coupling clamp or signal injector.
   - Use customer's exact cabling and connectors.
4. **Instrument.** Logging on the device. Scope on suspected lines. Spectrum analyzer for ambient noise.
5. **Common patterns:**
   - **CAN errors at temperature extremes.** Crystal frequency drift; check oscillator load caps and tolerance.
   - **Comm errors with certain other equipment running.** EMI/EMC issue.
   - **Errors on cold start.** Power sequencing or boot timing margin.
   - **Errors with long cables.** Termination, signal integrity, ground loops.
   - **Customer's mains is noisier.** PSU rejection insufficient; add input filter.
6. **Iterate.** Hypothesis → test → confirm/refute. Hardware changes often needed (add ferrite, change termination, increase clamp diode).
7. **Field instrumentation.** If you can't reproduce, ship a beefed-up unit with logging to capture in the field.

I've shipped products that worked everywhere except one specific customer, and the answer was a 3-phase motor 30 feet away creating a 50 V/m field. Adding chassis-grounded connector shells fixed it.

### Q8.12 — Explain the role of a stack-up in PCB design and how you choose layer count.

Stack-up: the cross-section of the board — alternating copper layers and dielectric (FR4 typically) layers — including layer functions, copper weights, and dielectric thicknesses.

Choosing layer count:
- **2-layer.** Simple boards, low-speed digital, hobby. No proper power/ground planes; signal integrity limited.
- **4-layer (Sig-GND-PWR-Sig).** Workhorse for embedded. Solid GND plane on L2 gives excellent signal integrity. Most embedded boards use 4 layers.
- **6-layer (Sig-GND-Sig-Sig-PWR-Sig or variations).** When 4L routing density is insufficient. Typical for boards with multiple high-speed buses (DDR3 + Ethernet + USB 2.0).
- **8-layer+.** DDR4, dense BGAs, high-speed FPGAs.

Stack-up choices affect:
- **Impedance.** Trace width to hit 50Ω depends on dielectric thickness.
- **EMI.** Returning planes reduce loop area.
- **Cost.** Layer count, lamination steps, copper weight all add cost.

Conventions:
- Reference plane (GND) adjacent to signal layers.
- Power plane is a return path too — but only for traces over it.
- Avoid "stripline" signals (between two reference planes) being routed without full reference; that's the rule.
- Symmetric stack-ups (mirrored about the center) reduce warping during fabrication.

Discuss stack-up with your fab early. They'll have a list of "in-house" stack-ups that are well-characterized and cheaper. Custom stack-ups cost more and have longer lead times.

### Q8.13 — Compare via-in-pad with regular vias near pads. When do you use each?

Via-in-pad: a via drilled through a component's solder pad. Required for fine-pitch BGAs (≤0.5mm) where there's no room for a "dog-bone" (pad with adjacent via).

Concerns:
- **Solder wicking.** During reflow, solder flows into the via, leaving the joint starved or voided.
- Mitigated by **filling and plating-over** (FPO): via filled with conductive (copper or epoxy) and plated flat. Adds fab cost.

Regular via near pad ("dog-bone"):
- Pad with a short trace to a via outside the pad.
- No solder wicking.
- Takes more space — incompatible with fine-pitch BGAs.
- Cheaper.

Decision:
- Use dog-bone for 0.8mm-pitch BGAs and bigger.
- Use via-in-pad-FPO for 0.5mm-pitch and smaller, or QFN with thermal pad needing many vias.
- Some boards mix: dog-bone where possible, via-in-pad only where space demands.

Consequence: via-in-pad turns a $5/ft² fab into a $15/ft² fab. Avoid unless geometry forces it.

---

## 9. Lab, Troubleshooting, Validation

### Q9.1 — Describe your standard lab setup for embedded firmware bring-up. What instruments and what's their role?

Core instruments:
1. **Oscilloscope** (200+ MHz, 4 channels, MSO/mixed-signal preferred). Look at any waveform, decode common protocols (UART/SPI/I2C/CAN), capture transients.
2. **Logic analyzer** (e.g., Saleae). Higher channel count than scope; less detail per channel. For complex multi-line digital flows.
3. **Multimeter.** Fast, true-RMS, with continuity beep. Voltage, current, resistance, junction.
4. **Bench power supply**, current-limited, multi-channel. Set safe limits during bring-up.
5. **Soldering / rework station.** Hot air, fine tips. Microscope or video inspection helpful.
6. **JTAG/SWD probe** (J-Link, ST-Link, OpenOCD-compatible). Flash firmware, debug, view memory.
7. **USB protocol analyzer** for USB devices (Beagle).
8. **CAN analyzer** (PEAK PCAN, Kvaser, Vector). Wireshark with socketcan also works on Linux.
9. **Spectrum analyzer** for RF or EMI work (sometimes shared / loaned).
10. **Signal generator** for stimulating inputs.
11. **Programmable load.**
12. **IR camera** for thermal hot-spot detection during bring-up.
13. **Network capture** (Ethernet tap, port mirror, Wireshark).
14. **Climate chamber** for temperature testing.

Support:
- **Schematics, datasheets, errata** within arm's reach (printed or on big monitor).
- **Logging laptop** with serial console, scripts.
- **Version control** for board revisions, firmware versions, test scripts.

The instrument that gets used most: scope. The instrument that gets used most often *incorrectly*: scope (improper grounding, wrong probe ratio, wrong bandwidth setting).

### Q9.2 — Walk through how you'd properly probe a high-speed signal with a scope. What are the pitfalls?

Pitfalls happen because the probe + ground lead form an LC circuit that distorts the signal:

1. **Use a real probe.** 10:1 passive probe rated for the bandwidth you need. For >300 MHz signals, active probes.
2. **Short ground.** Standard 6-inch ground clip adds ~150 nH inductance. At 100 MHz that's significant. Use a "spring tip" / ground spring or solder a wire pigtail to a nearby ground via.
3. **Probe directly at the receiver pin.** That's what the IC sees. Probing at the source captures launch but not what arrives.
4. **Compensate the probe.** Adjust the trimmer for a flat square wave. Wrong compensation rolls off edges or creates overshoot.
5. **DC coupling unless you specifically want AC.** AC-coupled hides DC offsets that affect the signal.
6. **Set bandwidth limit appropriately.** A 1 GHz scope viewing a 1 MHz signal benefits from a 20 MHz BW limit (less noise).
7. **Match impedance for high-speed.** Some probes are 50 Ω; need a 50 Ω termination at the scope or matched setup.
8. **Don't load the signal.** A 10x passive probe is ~10 pF. On a 50 MHz clock with 10 pF added to a high-impedance node, you can move the edge.

Always sanity-check: probe a known-good signal first. If it doesn't look right, the problem is your setup.

### Q9.3 — Explain how you'd debug a board that won't boot — completely dead, no UART, no LEDs.

Layered approach:

1. **Power.** Check input voltage. Then each rail at the IC pins (not just the regulator output). Multimeter, then scope (look for ripple, droop, oscillation).
2. **Reset.** Is reset asserted? Check reset pin with scope. Is reset deasserting at boot? Reset supervisor IC functioning?
3. **Clock.** Scope the main oscillator pins. Is it running? At the right frequency? Look for the right amplitude. A non-running crystal is a top-3 dead-board cause.
4. **Boot mode pins.** SoC boot pins set correctly? Verify each according to datasheet table.
5. **Boot device.** Is the eMMC / SPI flash powered? Probing CS or CLK to confirm boot ROM is reading.
6. **JTAG.** Connect debugger. Can you halt the CPU? Read the PC. Is it in BootROM, bootloader, or off in the weeds?
7. **BootROM signs.** Most BootROMs have known patterns: trying various boot sources, USB enumeration, debug pin output.
8. **Fault status.** Read SoC fault registers via JTAG (on Cortex-A: ESR, FAR; on M: CFSR, HFSR).
9. **Secondary checks.**
   - **Solder issues.** Reflow under microscope; look for cold joints especially on QFN/BGA.
   - **Thermal.** IR camera for hot spots indicating shorts.
   - **Scope rails for oscillation.** A regulator going into oscillation can power-off rails periodically.
10. **Compare to known-good.** If you have another board working, swap parts (regulator, clock crystal) to localize.

If even JTAG can't connect: it's almost always power, clock, or reset. Rare exceptions: BGA solder failure (X-ray to confirm).

### Q9.4 — What is signal integrity, and how do you validate it post-fab?

SI is whether signals arrive at receivers correctly: right voltage levels, right edge rates, no excessive ringing or reflections, no crosstalk corrupting them.

Validation:
1. **Eye diagrams.** For high-speed serial (USB, Ethernet, SerDes), capture many bit transitions overlaid. The "eye opening" must meet spec — height (margin to thresholds), width (timing margin).
2. **TDR (Time Domain Reflectometry).** Send a fast edge, observe reflections from impedance discontinuities. Locates problems by time-of-flight.
3. **Crosstalk measurements.** With one line driven and the other quiet, scope the quiet one — measure NEXT (near-end) and FEXT (far-end) crosstalk.
4. **Compliance testing.** Standards (USB-IF, IEEE 802.3) define test procedures. Often required before certification.
5. **Margin testing.** Vary supply voltage, temperature, pattern stress (PRBS sequences). System should still work at the corners.

Common findings:
- Overshoot / undershoot exceeding receiver specs.
- Reflections from missed termination.
- Crosstalk > 5% on parallel pairs.
- Eye closing at temperature extremes (jitter increases).

For embedded teams without a full SI lab, focus on:
- Rise/fall times match expectations.
- Settling within bit period.
- No glitches between transitions.

If symptoms suggest SI and you're out of depth, hire a consultant. SI debug is its own discipline.

### Q9.5 — Real-world scenario: a customer reports your product hangs after 3 days of operation. Walk through.

Long-running issues are often resource leaks, counters overflowing, or drift.

1. **Reproduce.** First step. Is it 3 days? 72 hours? 100 hours? Establish a deterministic test.
2. **Logs.** Get the logs. Crashes/messages near hang? Memory growth? Timer values?
3. **Memory leaks.** Heap growth over time. Add periodic memory dump to logs. Check stack high-water marks.
4. **Counter overflows.** A `uint32_t` counting milliseconds wraps at 49.7 days; `uint32_t` microseconds wraps at 71 minutes; `int32_t` seconds at 68 years (Y2038, but also relevant for absolute time). Subtract-and-compare with `int32_t` math handles wraparound; raw comparison doesn't.
5. **Watchdog.** If watchdog reset is happening but the system fully reboots and continues, you might not realize it's hanging. Check for repeated resets in logs.
6. **Deadlocks / livelocks.** Lock contention pattern that only occurs after specific event sequence.
7. **Time drift.** Clock drift accumulating; eventually a time-based check fails.
8. **Connection / session expiry.** Cloud connections, TLS sessions, OAuth tokens — all have lifetimes. Refresh logic might have a bug.
9. **Filesystem fill.** Logs accumulate, journal fills, can't write. `df -h` over time.
10. **Sensor drift / saturation.** ADC offset drift over thermal cycling. EEPROM wear-leveling failure.

Instrumentation:
- Add lightweight always-on logging that captures the state needed.
- Telemetry to remote service: heap size, queue depths, connection counts, key counters.
- Have a "long-haul" rig running new firmware for 7+ days before release.

I once chased a crash that happened only after ~26.5 days. The culprit: a `signed int` accumulator in a metric that overflowed after 2^31 ticks. Fixed by switching to `int64_t`. The interview gold here is "I know counter overflow class bugs and how to instrument for them."

### Q9.6 — Explain JTAG and SWD. What's the difference, and what can each do?

JTAG (IEEE 1149.1) is a 4-5 pin (TCK, TDI, TDO, TMS, optionally TRST) chain-able interface originally for boundary-scan testing of board interconnects. Adopted for in-circuit emulation by ARM and others.

SWD (Serial Wire Debug) is ARM's 2-pin (SWCLK, SWDIO) replacement for JTAG, same capability for ARM cores. Smaller pin count.

What both do:
- Halt CPU, single-step.
- Read/write CPU registers and memory.
- Set hardware breakpoints (limited number) and watchpoints.
- Flash programming via SoC-specific routines.
- Boundary scan (JTAG only).

JTAG-specific: chain multiple chips; boundary-scan testing for production.
SWD-specific: smaller, has SWO (single-wire trace output) for printf-style logging without UART.

Probes:
- **J-Link** (Segger). Best support. Up to 50 MHz SWD. Real-time terminal RTT. ~$400-1500.
- **ST-Link** (STMicro). For STM32, included on dev boards. Cheap.
- **OpenOCD-compatible.** Many low-cost probes work. Free software.
- **Keil ULINK, Lauterbach.** Pro-tier (Lauterbach for safety-critical heavy lifting).

For day-to-day: ST-Link or J-Link with OpenOCD or Segger software is ubiquitous.

### Q9.7 — What is Linux's `perf`, and how do you use it to profile an embedded application?

`perf` is the kernel-aware Linux profiler:

- **`perf record -g <cmd>`** samples on a timer (or event), captures call stacks. Output to `perf.data`.
- **`perf report`** explores the samples — flame graphs, top-down call trees, hot functions.
- **`perf top`** is realtime top by CPU.
- **`perf stat -e <events> <cmd>`** counts hardware events: cache misses, branch mispredicts, IPC.
- **`perf trace`** is `strace`-like but cheaper.
- **Tracepoints.** `perf record -e sched:sched_switch` captures every context switch.

For embedded:
- Cross-build perf with the kernel headers matching your target. Tricky; some BSPs don't ship it.
- Stripped binaries don't have symbols — keep an unstripped copy for `--symfs`.
- Frame-pointer-omitted code can't unwind stacks. Build with `-fno-omit-frame-pointer` for profile builds, or use DWARF unwinding (`--call-graph dwarf`, more overhead).

Profiling a Qt app:
1. Build with `-fno-omit-frame-pointer` and `-g`.
2. `perf record -g -F 99 -p <pid> -- sleep 30` while exercising the app.
3. `perf script | stackcollapse-perf | flamegraph.pl > graph.svg`.
4. Open the flame graph; identify hot stacks.

Ftrace, BPF (BCC, bpftrace) are also powerful for deeper Linux observability.

### Q9.8 — How do you validate timing-critical code in firmware?

Approaches:

1. **GPIO toggling.** Set a GPIO at start, clear at end of the section. Scope it. Read time off the screen. Crude but bulletproof — no instrumentation overhead beyond a single store.
2. **Cycle counter (DWT->CYCCNT on Cortex-M3+).** Read before/after, subtract. Microsecond resolution. Adds two instructions.
3. **Internal trace** (ITM, ETM on M-series; CoreSight on A-series). Real-time non-intrusive trace via SWO or external trace probe. Lauterbach, J-Trace, Ozone.
4. **`time` command + benchmark loop** for big-picture throughput.
5. **Static WCET analysis tools.** AbsInt aiT, Bound-T. Compute provably-worst-case execution time. Required for some safety-critical work.
6. **Latency stress testing.** Run the system at projected loads and measure tail latencies. `cyclictest` on Linux for scheduling latency.
7. **Statistical sampling.** Hardware performance counters + perf-style sampling (works on A-class; some M-class via DWT).

Anti-pattern: `printf` to time things. The printf takes longer than what you're measuring.

For an interrupt latency budget:
- Worst case from datasheet (12 cycles entry).
- Plus your longest ISR-disable region (`__disable_irq` → instructions → `__enable_irq`).
- Plus any higher-priority pending ISR.
- Validate by GPIO toggle from IRQ source to first ISR instruction.

### Q9.9 — Explain what an EMC pre-compliance scan looks like and what you'd do if you fail.

EMC pre-compliance: an in-house EMI scan to predict whether the product will pass formal certification.

Setup:
- DUT in a representative configuration.
- LISN (line impedance stabilization network) for conducted emissions on AC mains.
- Spectrum analyzer.
- Antennas: log-periodic (rod) for 30-200 MHz, biconical / log-periodic for 200 MHz - 1 GHz, horns for > 1 GHz.
- Anechoic chamber or open-area test site (or your lab if approximated).
- Near-field probes for source localization.

Types tested:
- **Conducted emissions.** Noise on power leads (150 kHz - 30 MHz typically).
- **Radiated emissions.** RF emissions over the air (30 MHz - several GHz).
- **Immunity.** Inject ESD, surge, RF, magnetic field — does DUT keep working?

Failure handling:
1. **Identify the offending frequency.** Broadband noise vs. discrete peaks.
2. **Source localization.** Near-field probe scan over the board. Find the hot spot.
3. **Common culprits:**
   - **Switching regulator.** Emissions at fundamental + harmonics. Add input/output filtering, snubber.
   - **Clock harmonics.** Spread-spectrum the clock; reduce clock-driven trace lengths; enable input slew limiting.
   - **Cables radiating.** Add ferrites, common-mode chokes. Better cable shield bonding.
   - **Enclosure leaks
