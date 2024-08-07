# CHAPTER 1 - An Introduction to Device Drivers

## Classes of Devices and Modules 5

- the Linux way of looking at devices distinguishes between three fundamental device types. Each module usually implements one of these types, and thus is classifiable as a __char module__, a __block module__, or a __network module__.
- __character devices__: (char) device
    - is one that can be accessed as a __stream of bytes__ (like a file); a char driver is in charge of implementing this behavior.
    - such a driver usually implements at least the _open_, _close_, _read_, and _write_ system calls.
    - the text console (`/dev/console`) and the serial ports (/`dev/ttyS0` and friends) are examples of char devices, as they are well represented by the stream abstraction.


## Communicating with Hardware

### I/O Ports and I/O Memory

- every peripheral device is controlled by writing and reading its registers.
- they are accessed at consecutive addresses, either in the memory address space or in the I/O address space.
- even if the peripheral bus has a separate address space for I/O ports, not all devices map their registers to I/O ports.
    - while use of I/O ports is common for __ISA__ peripheral boards, most __PCI devices__ map registers into a __memory address region__.
    - this I/O memory approach is generally preferred, because
        - it doesn’t require the use of special purpose processor instructions;
        - CPU cores access memory much more efficiently,
        - and the compiler has much more freedom in register allocation and addressing-mode selection when accessing memory.

#### I/O Registers and Conventional Memory

- main difference between I/O registers and RAM is that __I/O operations__ have __side effects__, while __memory__ operations have __none__.
- a driver must ensure that __no caching__ is performed and no read or write reordering takes place when accessing registers.
- the solution to __compiler optimization__ and __hardware reordering__ is to place a __memory barrier__ between operations that must be visible to the hardware (or to another processor) in a particular order.

### Using I/O Ports

- I/O ports are the means by which drivers communicate with many devices, at least part of the time.

#### I/O Port Allocation

- you should not go off and start pounding on I/O ports without first ensuring that you have exclusive access to those ports. - the kernel provides a registration interface that allows your driver to __claim the ports it needs__.
- the core function in that interface is `request_region`:

```c
#include <linux/ioport.h>
struct resource *request_region(unsigned long first, unsigned long n, const char *name);
```
- all port allocations show up in `/proc/ioports`.

#### Manipulating I/O ports

- most hardware __differentiates__ between 8-bit, 16-bit, and 32-bit ports. Usually you can’t mix them like you normally do with system memory access.
- as suggested in the previous section, computer architectures that support only memory mapped I/O __registers fake port I/O by remapping port addresses to memory addresses__, and the kernel hides the details from the driver in order to ease portability.

#### I/O Port Access from User Space

...

### Using I/O Memory

- despite the popularity of I/O ports in the x86 world, the main mechanism used to communicate with devices is through __memory-mapped registers__ and __device memory__.
- both are called I/O memory because the difference between registers and memory is transparent to software.
- I/O memory is simply a region of RAM-like locations that the device makes available to the processor over the bus.
- this memory can be used for a number of purposes, such as holding video data or Ethernet packets, as well as __implementing device registers that behave just like I/O ports__.
- depending on the computer platform and bus being used, I/O memory may or may not be accessed through page tables.
    - when access passes though page tables, the kernel must first arrange for the physical address to be visible from your driver, and this usually means that you must call __ioremap__ before doing any I/O.
    - if no page tables are needed, I/O memory locations look pretty much like I/O ports, and you can just read and write to them using proper wrapper functions.
- whether or not ioremap is required to access I/O memory, __direct use of pointers__ to I/O memory is discouraged.

#### I/O Memory Allocation and Mapping
