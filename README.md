# QEMU and Calvados LNX
This is a post covering the basics of the Calvados LNX  implementation on qemu. It is not intended to be exhaustive nor very detailed, but rather give insightful explanations on how this particular emulator has been made to work and where to focus to understand it.

## Calvados Architecture
See powerpoint presentation and Calvados LNS python scripts to get acquainted with product usage. Here is the gist :
- Iart/Uart communication on local network going through python client/server script
- Easy debugging of mcu/se intercommunications
- 2 QEMU instances (mcu/se) which can themselves be debugged (gdb stub)

About the calvados python script :
- Made to launch proxy, qemu se and qemu mcu. Order matters + `sleep()` to let connections establish
- Uses .json config files to generate command lines and launch subprocesses
- Can launch mcu in boot-mode or recovery
- Possible gdb stub for mcu and se
- Possible to launch only one instance of qemu through this script (`execvp`). Example : launching bolos on se only  `./tools/calvados.py -c ./config/nanox/se_solo.json -n`


## Qemu
See wikipedia page or read powerpoint presentation for an introduction to this tool : what an emulator is, what dynamic translation is, what the principle of hardware simulation is. This project emulates an arm processor on an stm soc and does binary translation with TCG (tiny code generator).

See [this series of post](https://github.com/airbus-seclab/qemu_blog) to learn about qemu internals. The most important subjects to understand for this project are :
- how to create new machines and device, using QOM (Qemu Object Model)
- how memory regions work and are managed through qemu
- how to simulate peripheral behaviours using callbacks on specific memory accesses (e.g registers)
- how to connect interrupt lines
- how the TCG works, what is the execution flow, how to integrate a software mmu inside of that to simulate the one on the st33. 

However the way our emulator is designed differs a bit form the what is done on the `qemu_blog`. Here are some additional informations to start faster ;

### Defining machine and device
QOM is like OOP made to work in C. Since Class implementation is not a language feature there are some subtleties, which are quite thoroughly explained on the series of post (see the first part about setting up the machine and cpu ; useful to understand class/state, init functions, dynamic type cast macro ). But basically it is only about declaring/defining methods and attributes a certain way. An instance of a class with its attributes are encapsulated in a `FoobarState` (which inherits parents’ attributes in memory like in C++), all methods are encapsulated in a single `FoobarClass` struct which inherits methods the same way. Newly created class are registered in qemu type system through a `TypeInfo` struct.

Constructor and destructor are basically the `instance_init` and `finalize` functions, but there is one important nuance here : the `realize` function. A device can be instantiated and terminated without being realized (that can be done by qemu if the user only wants to inspect a particular device). As such, the main work making device ready for practical use is located in the `realize` function.

### Create machine and device
See `nano*_*.c` files for example of machine definition. Do not bother with `armv7m_load_kernel` yet. We can see that we use `qdev_new` to create an instance of device, we set a `Property` then `realize` the device.
See `st*_soc.*` for example of device declaration/definition. Objects are initialized and registered with `object_initialize_child` and realized with `sysbus_realize`. All the device we create inherit from `sysbus_device`. The sysbus abstraction lets us connect interrupt lines and offer a way to map memory used and set up locally in device to the global system memory accessible by the cpu. It is useful for peripherals which register/buffer memory accesses need callback functions to correctly emulate product hardware. Look at code implementation and the `qemu_blog`.

### Memory
The [official documentation](https://qemu.readthedocs.io/en/latest/devel/memory.html) gives a good explanation of the memory API. Basically memory is mmaped through RamBlock.  A RamList holds all RamBlocks. The `MemoryRegion` is front-end : links guest code physical address space and RamBlocks. It can
represent RAM, ROM, I/O memory where r/w callbacks are invoked upon access (valid access size can be specified, qemu will automatically adapt size of desired hardware write).

Important note : a dirty memory bitmap is associated to each ram page to keep track of memory changes. It is useful in this project to keep track of self-modifying code and invalidate TBs (Translated Blocks) so that TCG execution stays correct. We must make use of the functions `memory_region_flush_*` (see flash and mmu code).

### Interrupts
Similar to a system of signal/slot. They are only taken into account after TB execution, but before system state update. Connection can be made between any two « gpio-like » objects. To raise "traditional interrupts" and call kernel's vector handlers, one should connect to the arm cpu interrupt controller, the NVIC (Nested Vector Interrupt Controller), which takes care of cpu interruption management (levels of pre-emption, nested interrupts etc.).
Look at the blog for basic definitions. Our project mainly uses a few functions :
- `qdev_init_gpio_in` creates interrupts line inside a device to which to connect to. A callback is specified and handles interruptions.
-  `qdev_get_gpio_in` retrieves the n-th device input interrupt line ("in" pin).
- `qdev_init_gpio_out` set up an array of `qemu_irq`  ready to be connected to the "in" part of another device. They are raised/lowered with `qemu_set_irq`.
- `qdev_connect_gpio_out` connects n-th "out" pin of a device to another `qemu_irq` (typically retrieved with `qdev_get_gpio_in`. 

Interrupt controllers can be "named" so that a device may hold more of such items. The api is then as easy to use, with a few natural variations.

### Raising Exceptions
As previously said, CPU state updating and interruption handling happens only after block execution. However, once an exception is raised, we want the execution of the TB to immediately stop. Furthermore, it does not seem that we can connect to the nvic interrupt line generating important cpu exceptions (the first few of the vector table). Another approach must be used.
Bolos makes use of the st33 MMU. Basically on page miss a BusFault is generated and the bolos busfault handler must update the component according to the specified error. The best way of emulating this hardware is still unclear. The best approach may be the one described by the `qemu_blog` , since it seems to integrate naturally with qemu workflow. As it was unclear, the naive approach taken for the moment is just a "mmio" object which raises this exception via callback on invalid access. However this is rather very slow since every memory access to system ram or user flash is made by mmio callback (even instruction fetches) and the mmu implementation is not speed-efficient as it has been kept simple and as close as hardware behaviour as possible, to debug it easily. 
See st33*_soc.c code to see how raising exception is handled for the moment. Basically a few attributes of the cpu struct are set to prepare for exception selection and nvic handling, and then cpu execution is interrupted with `raise_exception`. Qemu then calls an arm specific interrupt handler (`arm_v7m_cpu_do_interrupt`) which informs the nvic and let the tcg re-compile the next TB to execute.
- However some tweaks must be performed. When using this approach, there can be a problem with  processor accessing flash to load instructions/vector table data. Indeed, in qemu, if an exception occurs in thread mode, the nvic needs to know that in order for some info to be correctly written (for instance `lr` register) and we cannot yet switch to handler mode in qemu. But then, if the cpu is not in privileged mode either, qemu won't let the cpu access instructions in protected memory and the program does not behave as expected. In real life, once interrupt is raised, the nvic is informed immediately, and cpu execution switches to handler mode, so it is always able to retrieve data through mmu. 
- A workaround I have tried is to put the cpu in privileged mode, which lets cpu access instructions in flash, and then revert back if needed.

However the program still crashes a bit further in the program, sometimes because the "cpu has changed", and sometimes without qemu providing a reason. The mystery remains to be elucidated.

**Note** : the exception must be made "precise" so that execution restarts at the address that raised the busfault exception.

**Another note** : Qemu `ARMCPUState` manages A/R/M types of arm processor, so some attributes/functionality are mixed. For instance, even if the real cpu here does not have "secure" execution, some `v7m` struct attributes plan for both execution states ; the way to be sure to access to the correct `control` register for example is to write `cpu->env.v7m.control[cpu->env.v7m.secure]`.

### Property
Just an abstract way to set attributes of a device instance, see source code.

### Drive
User can specify drives to be used, through command line. In our project this will be files mimicking flash storage, which are going to be mmaped in memory. From the drive name a `BlockBackend` object is fetched, which will let memory be managed (`blk_set_perm`, `blk_pread`, `blk_pwrite`...).
See Iart and Flash implementations.

### Chardev
A chardev (or character device) communicates from qemu to the outside world. This can go through stdio or a tcp server for instance. `qemu_chr_find` verifies that backend/frontend connection has been correctly set up. It is up to the user to define the "backend", that is the outside provider of data, in the command line. Then in the program the "frontend" must be set up, which is primarily done with two handler to define :
- `fd_can_read` which tells how many bytes the frontend can manage once fecthed from backend
- `fd_read` which lets frontend effectively read content provided by backend

In this project, communication through python proxy uses chardev tcp servers.

### Kernel
Kernel is loaded with `-kernel` option in command line (user can provide binaray data or elf file), and `kernel_filename` is used inside qemu with `amrv7m_load_kernel` to load it into memory and also to register `cpu_reset` function. However the way it loads data in memory is still a bit unclear and there seems to require additional work to make loaded data available to cpu address space.

### Arm/Qemu-arm
There are some arm specifics to be aware of. Also remember that mcu and se are cortex m3/m4 ; do not hesitate to read the manual.
- NVIC and interruption execution, `lr` register and return to main program. msp/psp. That should be useful to debug exceptions.
- Bitbanding (qemu natively manages that)
- Privileged/unprivileged mode, handler/thread mode.
- See mcu option bytes : a memory region is aliased to `0x00` (usually always going to be flash). Very easy to implement anyway.
- For se : execution normally starts at `0x00200400` but qemu seems to always look for a vector table at `0x00`(even if changing `init_svtor` property it seems...) . So a false vector table is introduced at this address to correctly launch execution, until the application changes a cpu register that offsets the vector table to a specific flash address.
- Thumb instructions are used by bolos so instructions must end by `0b1`. So for instance se vector table must begin by `0x00602000 0x00200401`
 
## New MCU and SE
Almost all peripherals include small differences so developer must carefully read the datasheets to know how to adapt previous code. It is also important to make the minimal "necessary" component implementation so we accelerate development time and program speed. Relevant implementation choices are inferred from previous implementation, schematics, datasheets and bolos code.

The new mcu (stm32wb55) main difference is the CPU2. That changes behaviour of most components (exti, flash, syscfg…) where CPU2 is generally considered secured cpu. It shares ram/flash with CPU1, managed through new components : IPCC, HSEM.

Some behaviour cannot be emulated by qemu, or with great difficulties (ex : flash can prevent race condition by blocking accesses from a specific cpu). The best way to integrate cpu2 behaviour seems to implement a bluetooth peripheral which would mimick it without needing to load and execute its particular code.

### Todo
- Connect ssd1312 reset pin to mcu (using chardev and going through se ?)
- Load ST Firmware elf file to st33 qemu (still a bit unclear. Should use `armv7m_load_kernel` ? use `-device loader,file=...` ? Which function to use in `hw/core/loader.c` ? What address space to specify ?)
- Component (ST33J2M0) : MMU
- Components (STM32WB55) : usart, spi, dma, timer, usb






