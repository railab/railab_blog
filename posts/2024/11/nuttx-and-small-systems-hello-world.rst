.. title: Apache NuttX and small systems - Hello, World !
.. slug: nuttx-and-small-systems-hello-world
.. date: 2024-11-24 12:00:00 UTC
.. tags: nuttx, small systems
.. category: Blog
.. description: Let's see how low we can go with the memory usage for NuttX
   with a simple "Hello, World!" example.
.. type: text

.. thumbnail:: /images/posts/2024/11/nuttx-and-small-systems-hello-world/1.png
   :alt: Smoll NuttX
   :align: center

In the world of small embedded systems, balancing functionality with strict
resource limitations is a constant challenge. Apache NuttX, with its scalable and
modular design, allows developers to select only the features needed for a given
application and then fine-tune the system to minimize resource usage. However,
the vast number of configuration options can be overwhelming, and without
exploring the OS implementation or analyzing the generated binaries, it can be
difficult to effectively optimize resource consumption.

I've been curious for some time about how minimal we can make NuttX while still
implementing useful applications. It's time to check it out.
This is the first post in the series *"Apache NuttX and small systems"*, where
we'll experiment with reducing the size of the final NuttX image, explore how
low we can push resource requirements, and, if all goes well, implement
some useful applications that fit into some small embedded targets.

We're going to start with the classic "Hello, World!" example, examine its
memory consumption, and see how different configuration options influence
the final result.

.. TEASER_END

.. note::
   :class: card

   All presented results are for Apache NuttX with the following commits:

   * nuttx: ``6d629b3b36105e34f97f0439af5acaa8543c78a7``
   * nuttx-apps: ``0c467dc02d1f03f3f9f3defb16f36cb4f53b4c9d``

   Toolchain: ``arm-none-eabi-gcc (Arch Repository) 13.2.0``

=============
Hello, World!
=============

For this demo, we use the ``nucleo-f302r8`` board based on the STM32F302R8 chip
with 64 Kbytes of FLASH memory and 16 Kbytes of SRAM. Ready-to-compile code and
configuration is available in this
`repository <https://github.com/railab/railab_nuttx_code/>`_.

Our initial configuration is shown below, and it's devoid of
application-specific optimizations, which we'll address later.

.. code:: bash

  # CONFIG_ARCH_FPU is not set
  # CONFIG_STM32_SYSCFG is not set
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
  CONFIG_BOARD_LOOPSPERMSEC=16717
  CONFIG_DEBUG_FULLOPT=y
  CONFIG_DEBUG_SYMBOLS=y
  CONFIG_DEFAULT_SMALL=y
  CONFIG_DEFAULT_TASK_STACKSIZE=382
  CONFIG_IDLETHREAD_STACKSIZE=382
  CONFIG_INIT_ENTRYPOINT="hello_main"
  CONFIG_INTELHEX_BINARY=y
  CONFIG_RAILAB_MINIMAL_HELLO=y
  CONFIG_RAM_SIZE=16386
  CONFIG_RAM_START=0x20000000
  CONFIG_RAW_BINARY=y
  CONFIG_START_DAY=6
  CONFIG_START_MONTH=12
  CONFIG_START_YEAR=2011
  CONFIG_STM32_JTAG_SW_ENABLE=y
  CONFIG_STM32_USART2=y
  CONFIG_SYSTEM_TIME64=y
  CONFIG_USART2_SERIAL_CONSOLE=y

Our simple application looks like this:

.. code:: C

  int main(int argc, FAR char *argv[])
  {
    int i;

    for (i = 0; ; i++)
      {
        printf("Hello, World %d!!\n", i);

        sleep(2);
      }

    return 0;
  }

This simple program prints text in a loop along with the loop counter and waits
2 seconds. This way, we can better verify if the OS is working properly, instead of
writing a single line of static text, as is the case with the classic version
of "Hello, World!".

Since `PR#14620 <https://github.com/apache/nuttx/pull/14620>`_, NuttX
automatically prints the memory used by the image when the linker we
use supports the ``--print-memory-usage`` option. This makes our work
much easier.

Our first report and the basis for further comparisons look like this:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       23860 B        64 KB     36.41%
              sram:        3328 B        16 KB     20.31%

Important note: the SRAM report from the linker doesn't include memory used
for stacks for any of the OS components. In our case it's 382 bytes for both
``hello_main`` and ``idle_task`` tasks.

Configuration
=============

Now let's take a closer look at the system configuration and see the impact
of the most important options on memory.

#. ``CONFIG_DEFAULT_SMALL=y`` is the first option that we should set when we're
   dealing with small systems. This option gives us a minimal RTOS by
   default and sets many pre-allocated object to reasonable values, which
   helps a lot.

   To check what exactly ``CONFIG_DEFAULT_SAMLL`` does, you can use
   a simple ``git grep`` expression in ``nuttx`` or ``nuttx-apps`` directories:

   .. code:: shell

     git grep -A 2 -B 2 DEFAULT_SMALL -- '*Kconfig'

   Memory consumption without this option looks like this:

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       31640 B        64 KB     48.28%
                 sram:        5856 B        16 KB     35.74%

   This easy way we save 7,780 bytes of FLASH and 2,528 bytes of SRAM.

#. NSH completely disabled. The entry point is set directly to our application.

   NSH is a powerful tool that gives us a Linux-like console with many useful
   features. But it's a completely optional tool (like the whole ``nuttx-apps``),
   which may not be obvious to new NuttX users, since most upstream examples
   use NSH by default. When we're talking about really small systems, we most
   likely don't need NSH and can disable it.

   Let's check how much we save by disabling NSH and consequently the required
   support for built-in applications. We apply these changes to our base
   configuration:

   .. code:: bash

     CONFIG_BUILTIN=y
     CONFIG_INIT_ENTRYPOINT="nsh_main"
     CONFIG_INIT_STACKSIZE=768
     CONFIG_NSH_BUILTIN_APPS=y
     CONFIG_SYSTEM_NSH=y

   and this is the result:

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       28040 B        64 KB     42.79%
                 sram:        3360 B        16 KB     20.51%

   The difference is 4,180 bytes of FLASH and 32 bytes of SRAM more.

   Not that much, but it's worth mentioning that NSH in this state is not very
   useful. All commands are disabled by default with ``CONFIG_DEFAULT_SMALL``
   except ``help``, so each needed command must be enabled individually.

#. No debug features are enabled, which includes no debug logs and no assertions.

#. Optimization level set to ``CONFIG_DEBUG_FULLOPT=y``, which in NuttX
   corresponds to ``-Os`` GCC optimization level - "Optimize for size".

#. The only enabled peripheral is ``USART2``, so we can print to the console.

#. ``CONFIG_SYSTEM_TIME64`` is set, which enables 64-bit system clock.
   I must mention that this option doesn't help when it comes to small systems,
   but was included because it may soon be mandatory in NuttX, which is requried
   by the latest version of the POSIX standard (IEEE Std 1003.1-2024).
   For those who want to stick with 32-bit system clock, disabling the
   option gives us:

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       22960 B        64 KB     35.03%
                 sram:        3304 B        16 KB     20.17%

   It's 900 bytes of FLASH and 24 bytes of SRAM less. Quite a bit of savings
   considering that many small system applications won't care about the system
   time.

#. ``CONFIG_DEFAULT_TASK_STACKSIZE=382`` is the smallest value that works for our
   example. I won't focus more on stack configuration here.

#. FPU support is disabled with ``# CONFIG_ARCH_FPU is not set``.

   Reverting this change gives us:

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       23972 B        64 KB     36.58%
                 sram:        3464 B        16 KB     21.14%

   FPU support costs us 112 bytes of FLASH and 136 bytes of SRAM. More
   importantly, the stack sizes must be larger so that the CPU context
   increased by the FPU registers will fit on any used stack. In the case
   of ``armv7-m`` it's additional 72B (``SW_FPU_REGS  * 4B``) aligned
   up to 64B, which is 128 bytes more in total.

====================
Let's optimize more!
====================

The initial result, without much effort, is **23,812 bytes of FLASH** and
**3,344 bytes of SRAM**.
Now, it's time for some deeper optimization.

Now, the magic of Link Time Optimization (LTO) comes into play. By enabling it
with the ``CONFIG_LTO_FULL=y`` option, we can expect significant improvements in
FLASH usage - and that's exactly what happens:

.. code:: shell

     Memory region         Used Size  Region Size  %age Used
             flash:       19784 B        64 KB     30.19%
              sram:        3296 B        16 KB     20.12%

We reduce FLASH usage by 4,076 bytes and SRAM by 32 bytes, breaking
the 20KB FLASH barrier. LTO with CMake seems to finally be working properly
in NuttX - with no errors when compiling.

With LTO enabled, the number of symbols in the image should be limited.
Now is a good time to look at how the memory is consumed. Using the ``nm`` tool,
let's print the 30 larges symbols:

.. code:: shell

  arm-none-eabi-nm --size-sort build/nuttx | tail -n 30

Here's the output, with interesting symbols marked by me:

.. code:: shell

  000000d4 t _exit.isra.0
  000000d6 t dir_read
  000000dc t rawoutstream_puts
  000000e8 T _assert
  000000ee t uart_close
  000000f8 b g_kthread_group
  000000fa t mm_delayfree.constprop.0
  00000100 b g_usart2rxbuffer              <<
  00000100 b g_usart2txbuffer              <<
  00000100 t uart_register.isra.0
  00000110 t mm_malloc
  00000118 t dir_allocate
  00000124 t group_leave
  0000012c t up_setup
  0000014c t uart_write
  00000158 t nxsched_set_priority
  00000170 t nxsig_clockwait.constprop.0
  00000188 T _vectors
  000001a0 b g_sigpool                     <<
  000001a4 t stm32_configgpio.isra.0
  000001c0 T __start
  00000208 d g_pathbuffer                  <<
  00000212 t uart_ioctl
  00000236 t up_interrupt
  00000256 t uart_read
  00000260 t nx_vopen
  000002a0 T __udivmoddi4                  <<
  00000310 b g_irqvector                   <<
  000004c8 t lib_vsprintf
  00000894 t nx_start

At this point it's also worth looking at the decompiled code and see if there're
any "ifdefs" left that can be disabled. I usually use ``objdump`` in combination
with ``less`` for this purpose:

.. code:: shell

   arm-none-eabi-objdump -d -S build/nuttx | less

There's not much left to do, let's take a few final steps:

#. Serial buffers ``g_usart2rxbuffer`` and ``g_usart2txbuffer`` can be easly
   reduced with:

   .. code:: shell

     CONFIG_USART2_RXBUFSIZE=0
     CONFIG_USART2_TXBUFSIZE=32
     CONFIG_STDIO_BUFFER_SIZE=32

#. We don't care about signals, so we can set ``g_sigpool`` to minimal size with
   ``CONFIG_SIG_ALLOC_ACTIONS=0`` and ``CONFIG_SIG_PREALLOC_IRQ_ACTIONS=0``.
   As we're talking about singals, we can also set ``CONFIG_SIG_PREALLOC_ACTIONS=0``.

#. ``g_pathbuffer`` is much too big, reduce it with ``CONFIG_PATH_MAX=32``.

#. ``__udivmoddi4`` is the result of 64-bit timer support, let's disable it
   now with ``CONFIG_SYSTEM_TIME64=n``.

#. ``g_irqvector`` is the table that holds the interrup vector informations
   in NuttX. At default, all interrupts are supported, which is often not required.
   For small systems we most likely need small part of this and in NuttX it's
   easy to configure. For details please refer to
   `NuttX documentation <https://nuttx.apache.org/docs/latest/guides/smaller_vector_tables.html>`_.

   We set the dynamic version of the minimal vector table, which is esaier to
   use at the cost of some extra code:

   .. code:: shell

      CONFIG_ARCH_MINIMAL_VECTORTABLE=y
      CONFIG_ARCH_MINIMAL_VECTORTABLE_DYNAMIC=y
      CONFIG_ARCH_NUSER_INTERRUPTS=5

#. finally let's reduce various buffers and pre-allocated arrays:

   .. code:: shell

      CONFIG_TASK_NAME_SIZE=0
      CONFIG_NAME_MAX=0
      CONFIG_PID_INITIAL_COUNT=3
      CONFIG_NFILE_DESCRIPTORS_PER_BLOCK=3

After applying all suggestions from above, we get this:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       18256 B        64 KB     27.86%
              sram:        1268 B        16 KB      7.74%

All symbols addressed above have been reduced:

.. code:: shell

   arm-none-eabi-nm --size-sort build/nuttx | tail -n 30

   000000be t dir_seek                                                                             
   000000c2 t file_dup3.constprop.0
   000000c4 t inode_search
   000000c4 t nxsem_post
   000000c8 t nxsem_wait
   000000ce t uart_poll
   000000d0 b g_sigpool                        <<
   000000d4 t _exit.isra.0
   000000d6 t dir_read
   000000e0 t rawoutstream_puts
   000000e8 T _assert
   000000e8 t group_leave
   000000ec t uart_close
   000000fa t mm_delayfree.constprop.0
   00000100 t uart_register.isra.0
   00000110 t mm_malloc
   00000114 t dir_allocate
   0000012c t up_setup
   00000134 t nxsig_clockwait.constprop.0
   0000014c t uart_write
   00000158 t nxsched_set_priority
   00000188 T _vectors
   000001a4 t stm32_configgpio.isra.0
   000001c0 T __start
   00000210 t uart_ioctl
   00000236 t up_interrupt
   00000256 t uart_read
   00000268 t nx_vopen
   000004c8 t lib_vsprintf
   000007b4 t nx_start

Of the 30 largest symbols, only ``g_sigpool`` is not in the ``text`` section.

At this point, it seems like there isn't much left to optimize.

One more thing worth nothing is that ``_vectors`` for small MCUs, which
generally support fewer periperal interrupts, should be smaller.

=======
Summary
=======

In this post, we explored a simple "Hello, World!" example to gain a general
understanding of NuttX's memory consumption. The example utilized OS interfaces
in a very limited way, allowing the compiler to optimize much of the unused
code, resulting in a final image size of **18,256 bytes of FLASH** and
**1,268 bytes of SRAM**. While GCC with LTO provides excellent results,
it's worth noting that the binary still includes some POSIX-related code unused
by our application - such as logic related to signals - that our tools cannot
optimize. Unfortunately, there's little we can do about this without significant code
modifications. Long ago, NuttX offered an option to disable signals,
but due to non-POSIX compliance, it was removed (see this 
`commit <https://github.com/apache/nuttx/commit/abf6965c24f25146bde368f28ee49df704945915>`_).

But why care about small systems in NuttX at all? Does this make sense when
memory is relatively inexpensive these days? There are several compelling reasons:

#. It allows us to use our favorite embedded tool even for simple applications
   on resource-constrained chips or to create a minimalist bootloader.
   This way, we don't have to rely on external projects.

#. Designing an OS with small systems in mind promotes better memory management
   and overall code quality.

#. *"Small footprint"* is a good marketing slogan for the project.

#. By carefully tuning the kernel, we can save memory for our application
   and potentially reduce costs by using MCUs with fewer resources - though such
   savings are typically minor and most impactful in large-scale production.

Looking ahead, I plan to delve deeper into individual NuttX components and
analyze their memory costs. In the next post, we'll take NuttX image stripping
a step further, attempting to build the smallest possible image and exploring
what remains in the binary.
