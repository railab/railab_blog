.. title: Apache NuttX and small systems - NuttX Core Size
.. slug: nuttx-and-small-systems-core-os
.. date: 2024-12-11 18:00:00 UTC
.. tags: nuttx, small systems
.. category: Blog
.. description: In this post, we analyze the size of the Apache NuttX core with
   all possible features disabled, and evaluate whether the system remains usable
   in such a minimal state.
.. type: text

.. thumbnail:: /images/posts/2024/12/nuttx-and-small-systems-os-core/1.png
   :alt: Minimal NuttX Core Size
   :align: center

We continue our exploration of Apache NuttX for small embedded systems.
In the  `previous post <link://slug/nuttx-and-small-systems-hello-world>`__,
we examined a simple *"Hello, World!"* example and explored how small it could be
on NuttX.

Now, we take it a step further by disabling all possible NuttX features, allowing
the toolchain to remove as much code as possible. This approach leaves us with
the core of NuttX—components that can't be eliminated through configuration
options and compiler optimizations.

Finally, we implement a **trivial** application using two different approaches—the
POSIX-compliant method and the non-portable alternative—highlighting the trade-offs
between achieving portability and minimal system size.

.. TEASER_END

.. note::
   :class: card

   All presented results are for Apache NuttX with the following commits:

   * nuttx: ``6d629b3b36105e34f97f0439af5acaa8543c78a7``
   * nuttx-apps: ``0c467dc02d1f03f3f9f3defb16f36cb4f53b4c9d``

   Toolchain: ``arm-none-eabi-gcc (Arch Repository) 13.2.0``

================
NuttX core image
================

Ready-to-compile code and configurations are available in this
`repository <https://github.com/railab/railab_nuttx_code/>`_.
Before going any further, I highly recommend viewing the previous
post from this series, as this post builds on the information covered earlier.

Our goal now is to prepare three slightly different configurations.
Each subsequent configuration will strip away key OS components
until the system is no longer POSIX-compliant but remains  functional.

The new application code is the simplest possible loop, ensuring that no
additional OS code is included in our image:

.. code:: C

  int main(int argc, FAR char *argv[])
    {
      while(1);
    }

Config 1
========

We start from the setup we completed last time, but we enable our new
``oscore`` application and boot into it:

.. code:: shell

  CONFIG_ARCH="arm"
  CONFIG_ARCH_BOARD_CUSTOM=y
  CONFIG_ARCH_BOARD_CUSTOM_DIR="boards/arm/stm32/nucleo-f302r8/"
  CONFIG_ARCH_BOARD_CUSTOM_DIR_RELPATH=y
  CONFIG_ARCH_BOARD_CUSTOM_NAME="nucleo-f302r8"
  CONFIG_ARCH_BOARD_NUCLEO_F302R8=y
  CONFIG_ARCH_BUTTONS=y
  CONFIG_ARCH_CHIP="stm32"
  CONFIG_ARCH_CHIP_STM32=y
  CONFIG_ARCH_CHIP_STM32F302R8=y
  CONFIG_ARCH_MINIMAL_VECTORTABLE=y
  CONFIG_ARCH_MINIMAL_VECTORTABLE_DYNAMIC=y
  CONFIG_ARCH_NUSER_INTERRUPTS=5
  CONFIG_BOARD_LOOPSPERMSEC=16717
  CONFIG_DEBUG_FULLOPT=y
  CONFIG_DEBUG_SYMBOLS=y
  CONFIG_DEFAULT_SMALL=y
  CONFIG_DEFAULT_TASK_STACKSIZE=384
  CONFIG_IDLETHREAD_STACKSIZE=384
  CONFIG_INIT_ENTRYPOINT="oscore_main"
  CONFIG_INTELHEX_BINARY=y
  CONFIG_LTO_FULL=y
  CONFIG_NAME_MAX=0
  CONFIG_NFILE_DESCRIPTORS_PER_BLOCK=3
  CONFIG_PATH_MAX=32
  CONFIG_PID_INITIAL_COUNT=3
  CONFIG_RAILAB_MINIMAL_OSCORE=y
  CONFIG_RAM_SIZE=16386
  CONFIG_RAM_START=0x20000000
  CONFIG_RAW_BINARY=y
  CONFIG_SIG_ALLOC_ACTIONS=0
  CONFIG_SIG_PREALLOC_ACTIONS=0
  CONFIG_SIG_PREALLOC_IRQ_ACTIONS=0
  CONFIG_START_DAY=6
  CONFIG_START_MONTH=12
  CONFIG_START_YEAR=2011
  CONFIG_STDIO_BUFFER_SIZE=32
  CONFIG_STM32_JTAG_SW_ENABLE=y
  CONFIG_STM32_USART2=y
  CONFIG_TASK_NAME_SIZE=0
  CONFIG_USART2_RXBUFSIZE=0
  CONFIG_USART2_SERIAL_CONSOLE=y
  CONFIG_USART2_TXBUFSIZE=32

The resource consumption is as follows:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       17688 B        64 KB     26.99%
              sram:        1268 B        16 KB      7.74%


For comparison, let's look at the results we got for "Hello, World!":

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       18256 B        64 KB     27.86%
              sram:        1268 B        16 KB      7.74%


As we can see, there is not much difference here. The empty program is slightly
smaller than the printing one. With the console enabled, the overhead of
``printf`` and ``sleep`` support is negligible.

The console support code is included in the image anyway, leaving no way for
the compiler to optimize it out.

The next step is to remove console support.

Config 2
========

In this configuration we completely disable support for the serial port and
console. If our application doesn't require UART support, this is an easy
optimization. The obvious downside is the lack of printing capabilities;
therefore, printf-debugging becomes impossible.

Without ``/dev/console``, the system won't be able to initialize standard I/O
streams, which is a POSIX violation. Serial port support is not required, but
file descriptors 0, 1, and 2 are reserved for  ``stdin``, ``stdout``, and
``stderr``, respectively.

NuttX allows you to redirect standard streams to ``dev/null`` if the console is
not supported. We just need to enable support for the NULL device.

The modifications in the config are as follows:

.. code:: shell

  CONFIG_SERIAL=n
  CONFIG_STM32_USART2=n
  CONFIG_DEV_CONSOLE=n
  CONFIG_DEV_NULL=y

The memory report is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       11920 B        64 KB     18.19%
              sram:        1072 B        16 KB      6.54%

This saves 5,768 bytes of FLASH and 196 bytes of SRAM compared to the
console-enabled setup with UART—a significant reduction!

Config 3
========

Now let's take one final step and disable both ``/dev/console`` and ``/dev/null``.
This way, we should completely remove file system support, as there are no files
used in our image. Since we don't use files at all, we can also disable file
descriptor cloning when a new task is started. At this point, our system is
intentionally no longer POSIX-compliant.

Changes in configuration:

.. code:: shell

    CONFIG_DEV_CONSOLE=n
    CONFIG_DEV_NULL=n
    CONFIG_FDCLONE_DISABLE=y

While compiling our new program, we notice an additional warning that appears:

.. code:: shell

   external/nuttx/sched/group/group_setupidlefiles.c:115:4: warning: #warning file descriptors 0-2 are not opened [-Wcpp]

There is no available device that can be used as a backend for file descriptors 0-2.
This means that any OS feature using standard I/O streams is no longer allowed.

The result is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:        6860 B        64 KB     10.47%
              sram:         912 B        16 KB      5.57%

We notice a huge reduction in FLASH, and we easily broke the 10KB FLASH barrier.

At this point, everything that can be disabled has been disabled, and everything
the compiler is able to remove has been removed. Without modifying the kernel
sources, we can't go any lower for the architecture used.
I think we can call this the "**NuttX Core**."

Below is the complete list of symbols, with a comment about the OS module to
which each one belongs:

.. code:: shell

  00000001 b g_nx_initstate                   | sched
  00000002 b g_ino                            | fs
  00000002 t oscore_main                      | apps
  00000004 b g_errno                          | libc
  00000004 t g_idle_topstack                  | arch
  00000004 d g_irqmap_count                   | sched
  00000004 b g_lastpid                        | sched
  00000004 b g_mmheap                         | mm
  00000004 b g_npidhash                       | sched
  00000004 b g_pidhash                        | sched
  00000004 b g_reboot_notifier_list           | sched
  00000004 b g_running_tasks                  | sched
  00000004 b g_system_ticks                   | sched
  00000006 T abort                            | libc
  00000008 b g_inactivetasks                  | sched
  00000008 b g_pendingtasks                   | sched
  00000008 b g_readytorun                     | sched
  00000008 b g_sigpendingaction               | sched
  00000008 b g_sigpendingirqaction            | sched
  00000008 b g_sigpendingirqsignal            | sched
  00000008 b g_sigpendingsignal               | sched
  00000008 b g_waitingforsignal               | sched
  00000008 d g_wdactivelist                   | sched
  0000000a t start                            | arch
  0000000c T __assert                         | libc
  0000000c d g_sync_nb                        | fs
  0000000c t tls_get_info                     | libc
  00000010 t panic_notifier_call_chain        | sched
  00000012 t memset.constprop.0               | libc
  00000014 t __errno                          | libc
  00000014 t free                             | mm
  00000014 t sq_remfirst                      | misc
  00000018 t irq_unexpected_isr               | sched
  00000018 t memcpy.constprop.0.isra.0        | libc
  00000018 t sched_lock.isra.0                | sched
  0000001c t arm_svcall                       | arch
  0000001c t nxsched_gettid                   | sched
  0000001e t inode_free                       | fs
  00000020 t strlcpy.isra.0                   | libc
  00000024 t up_release_stack.isra.0          | arch
  00000026 t wd_cancel.isra.0                 | sched
  00000028 b g_irqvector                      | sched
  0000002c T arm_doirq                        | arch
  0000002c t up_mdelay.constprop.0            | arch
  0000002c t zalloc                           | mm
  00000030 T up_saveusercontext               | arch
  00000038 t group_postinitialize             | sched
  00000038 b g_tasklisttable                  | sched
  00000038 t irq_dispatch                     | sched
  0000003e t tls_init_info                    | sched
  00000040 t exception_direct                 | arch
  00000040 t nxtask_start                     | sched
  0000004c t nxsig_release_pendingsigaction   | sched
  0000004c t sync_reboot_handler              | fs
  00000050 b g_last_regs                      | arch
  00000050 t group_initialize                 | sched
  00000050 t irq_attach.constprop.0.isra.0    | sched
  00000050 t stm32_timerisr                   | arch
  00000054 t arm_hardfault                    | arch
  00000054 t nxsched_release_tcb.isra.0       | sched
  00000054 t nxsched_remove_readytorun        | sched
  0000005c t sched_unlock.isra.0              | sched
  00000060 t nxsched_merge_pending            | sched
  00000062 T exception_common                 | arch
  00000062 b g_irqmap                         | sched
  0000006c t up_initial_state                 | arch
  0000007c b g_idletcb                        | sched
  00000096 t task_fssync                      | fs
  0000009a t files_putlist.part.0             | fs
  000000a0 t nxsched_add_readytorun           | sched
  000000a8 b g_kthread_group                  | sched
  000000d0 b g_sigpool                        | sched
  000000dc t mm_unlock                        | mm
  000000e0 t _exit.isra.0                     | sched
  000000e4 T _assert                          | sched
  000000fa t mm_delayfree.constprop.0         | mm
  00000110 t mm_malloc                        | mm
  00000120 t mm_lock                          | mm
  0000013c t group_leave                      | sched
  00000180 T __start                          | arch
  00000188 T _vectors                         | arch
  00000734 t nx_start                         | sched

Now, let's look at memory usage per OS module for FLASH:

.. table:: FLASH usage per OS module
   :class: table table-secondary
   :widths: grid

   ============== ===== ==== ==== ==== ===== ==== ====
   OS module      sched arch mm   fs   libc  misc apps
   ============== ===== ==== ==== ==== ===== ==== ====
   FLAS Size [B]  3744  1424 1094 422  124   20   2
   ============== ===== ==== ==== ==== ===== ==== ====

And next, for SRAM:

.. table:: SRAM usage per OS module
   :class: table table-secondary
   :widths: grid

   ============== ===== ==== ==== ==== =====
   OS module      sched arch fs   libc  mm
   ============== ===== ==== ==== ==== =====
   SRAM Size [B]  795   80   14   4     4
   ============== ===== ==== ==== ==== =====

Most of the symbols come from ``sched`` and ``arch`` which is what you would
expect.

When adding data from the tables, we can see that the results differ from
those returned after compilation. I don't know exactly where this comes from.
Part of the difference may be due to data alignment, but even with that,
the numbers still don't add up. If anyone knows the explanation, please let me know.

In this configuration, we intentionally don't use files, so more OS logic has been
removed by the compiler. We dropped almost all logic related to the file system,
but it's interesting that there are 422 bytes of code left from ``fs``.
We basically removed all kernel code responsible for hardware abstraction,
which allows separation of kernel space from user space.

Now the question is whether the OS in this state makes sense at all and can be
practically used. Are we able to implement any application without files in NuttX?
It depends. If we accept the loss of portability and design an application in
a non-POSIX way, it's possible. In the next section I'll show how.

======
blinky
======

This time, we'll implement the classic *"blinky"* example.
The goal here is to demonstrate the use of NuttX in a non-standard way.

Here, we have two versions of minimalistic "blinky": one implemented in a
portable way using files and the other not POSIX-compliant but with minimal
resource usage. The functionality of both applications will be the same;
the only difference is that one will use a portable interface, while the other
won't. The basis for both examples is "Config 3" from above.

POSIX-way blinky
================

Let's start with the file-based version. In this case, we'll use the user
LED driver available in NuttX. For this, we need to disable the LED control
by the OS and give the control to the user application. An alternative
solution would be to use a GPIO driver, but we won't focus on that here.

The required configuration changes are:

.. code:: shell

   CONFIG_ARCH_LEDS=n
   CONFIG_USERLED=y
   CONFIG_USERLED_LOWER=y

The application code is shown below:

.. code:: C

  #include <nuttx/config.h>
  #include <sys/ioctl.h>
  #include <unistd.h>
  #include <fcntl.h>
  #include <nuttx/leds/userled.h>

  #define LEDS_DEVPATH "/dev/userleds"

  int main(int argc, FAR char *argv[])
  {
    userled_set_t ledset;
    int ret;
    int fd;

    /* Open user LED device */

    fd = open(LEDS_DEVPATH, O_WRONLY);
    if (fd < 0)
      {
        return -1;
      }

    while (1)
      {
        /* Toggle LED */

        ledset ^= 1;

        /* Set LED */

        ret = ioctl(fd, ULEDIOC_SETALL, ledset);
        if (ret != 0)
          {
            return -1;
          }

        /* Wait some time */

        sleep(1);
      }

    return 0;
  }

This version of the example gives us:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       11512 B        64 KB     17.57%
              sram:         996 B        16 KB      6.08%

Non-portable blinky
===================

Now, it's time for a file-free implementation to eliminate the file abstraction
from our firmware. In this case, we'll use the STM32 architecture features directly,
in a non-portable manner.

The user LED driver support is no longer needed, but we have to keep
``CONFIG_ARCH_LEDS=n``.

To access architecture-specific APIs from the application context, we need to
manually add the architecture directory to the build system.
For instance, when compiling NuttX with CMake and our application is called ``blinky2``,
we have to add the following lines to the application's ``CMakeLists.txt``:

.. code:: cmake

  target_include_directories(apps_blinky2 PRIVATE ${CMAKE_SOURCE_DIR}/arch/arm/src/stm32)
  target_include_directories(apps_blinky2 PRIVATE ${CMAKE_SOURCE_DIR}/arch/arm/src/common)

The changes to the previous program are straightforward: we directly configure
the GPIO, and instead of changing the LED state via the file interface, write
the GPIO state directly using the architecture interface. The modified code looks
as follows:

.. code:: C

  #include <nuttx/config.h>
  #include <unistd.h>
  #include "stm32.h"

  #define GPIO_LED1      (GPIO_OUTPUT|GPIO_PUSHPULL|GPIO_SPEED_50MHz| \
                          GPIO_OUTPUT_CLEAR|GPIO_PORTB|GPIO_PIN13)

  int main(int argc, FAR char *argv[])
  {
    bool ledon = false;

    /* Initialize LED GPIO */

    stm32_configgpio(GPIO_LED1);

    while (1)
      {
        /* Toggle led */

        ledon ^= 1;

        /* Set led */

        stm32_gpiowrite(GPIO_LED1, ledon);

        /* Wait some time */

        sleep(1);
      }

    return 0;
  }

It's worth noting that this code can be portable across architectures that
define the same API functions for GPIO.
As a result, it'll work on most STM32 chips supported in NuttX.
However, due to some inconsistencies in architecture ports, it's not compatible
with all STM32 chips at the time of writing this.

The resource usage for this implementation is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:        7132 B        64 KB     10.88%
              sram:         912 B        16 KB      5.57%

The difference between the file-based version and the non-portable version
is 4,380 bytes of FLASH and 84 bytes of SRAM.

This simple example demonstrates the GPIO interface, but NuttX on STM32 offers
other easy-to-use low-level APIs like DMA, timers, PWM, or ADC. Additionally,
bus drivers like SPI or I2C in NuttX are designed to provide low-level
interfaces for kernel drivers that are used without files. We can abuse this
interface and call it directly in user space.
Finally, hardware description headers are available—often of much higher
quality than the code provided by vendors—enabling direct manipulation of
registers.

The presented approach is applicable only in the NuttX FLAT build, where
there's no hardware protected separation between user space and kernel code.
For small systems, however, this is the only sensible architecture because
it requires less powerful chips.

=======
Summary
=======

We have collected some data on the size of the NuttX image under various
simple scenarios. This data can serve as a baseline for future resource usage
analysis in NuttX releases.

If we want to save more space, we can manually modify the kernel code by
excluding functions that we know we won't  use and which can't be removed
by the compiler. For example, eliminating the remaining signal logic.
However, since major improvements are unlikely in this area, I don't discuss
this topic.

Even without the file interface, NuttX still offers many features that
we can use, such as the architecture-specific code, ``libc``, ``libm``,
synchronization mechanisms, and more. By utilizing NuttX in this manner,
we can achieve very low resource usage.

While it's technically possible to use non-portable interfaces, doing so is
generally not recommended for NuttX users, as it sacrifices the primary advantage
of the OS: portability and modularity. For minimal applications, there are certainly
more appropriate tools available. In many small system cases, it's likely you don't
even need an RTOS.

**But...** if, for some reason, you truly want to use NuttX for a small project,
and you're fully aware of the disadvantages of mentioned solutions, do what you
want with your code ;) With a few small tweaks to the build system, you can
easily access architecture-specific APIs and register definitions directly from your
application. Personally, I prefer to work with a single tool when possible (Emacs
fan here), even if it means hacking it to suit my needs. For this reason, I push
NuttX to its limits in my small projects.

That’s all for today. Now that we have explored how small NuttX can be in its
simplest form, it’s time to examine the requirements for more advanced OS
features that we’ve kept disabled so far. In the next post, we’ll take
a closer look at these through simple examples to determine
which ones are suitable for small systems.
