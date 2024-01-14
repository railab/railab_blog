.. title: Segger RTT as an interface for NuttX NxScope library
.. slug: segger-rtt-interface-nxscope
.. date: 2024-01-21 12:00:00 UTC
.. tags: nxscope, rtt
.. category: howto
.. description: How to use Segger RTT as an interface for NxScope library
.. type: text

One of my favorite things when working with Segger J-Link is
`RTT <https://www.segger.com/products/debug-probes/j-link/technology/about-real-time-transfer>`_
support which significalntly improves my debugging experience.
With a single interface (one USB cable), I can run:

* RTT serial console to communicate with the target
* SystemView to capture system events
* gdb for general debugging

The latest addition to this list is real-time data streaming with the NuttX
NxScope library and RTT channel as a communication interface.
This feature is supported with `nxscli <https://github.com/railab/nxscli>`_
tool since version ``0.5.1``. Now we can visualize and debug complex data
relations much more simpler.

This post discusses how to configure the NxScope example with NuttX to work over
an RTT channel with an RTT serial console and SystemView support.

For demonstration, we use Nordic nRF52832-DK board with a built-in J-Link OB
interface, so there is no need to connect any external device.

.. TEASER_END

=============
Configuration
=============

We start with the basic NSH configuration for the ``nrf52832-dk`` board::

     cmake -B build_nxscope_rtt -DBOARD_CONFIG=nrf52832-dk/nsh -GNinja

then, run ``menuconfig`` and apply the changes from below::

     cmake --build build_nxscope_rtt -t menuconfig

First, let's configure Nxscope library and the ``examples/nxscope``
application:

#. Enable and configure NxScope library::

     CONFIG_LOGGING_NXSCOPE=y
     CONFIG_LOGGING_NXSCOPE_DIVIDER=y
     CONFIG_LOGGING_NXSCOPE_INTF_SERIAL=y

#. Disable NSH::

     CONFIG_SYSTEM_NSH=n
     CONFIG_NSH_LIBRARY=n

#. Set the entry point to ``nxscope_main``::

     CONFIG_INIT_ENTRYPOINT="nxscope_main"

#. Enable and configure the NxScope example::

     CONFIG_EXAMPLES_NXSCOPE=y
     CONFIG_EXAMPLES_NXSCOPE_RXBUF_LEN=255
     CONFIG_EXAMPLES_NXSCOPE_STREAMBUF_LEN=2048

#. Configure the sample thread interval::
  
     CONFIG_EXAMPLES_NXSCOPE_TIMER=y
     CONFIG_EXAMPLES_NXSCOPE_TIMER_INTERVAL=2000

#. Enable ``TIMER0`` support for waking up the nxscope samples thread::

     CONFIG_ARCH_BOARD_COMMON=y
     CONFIG_NRF52_TIMER0=y
     CONFIG_TIMER=y

Now we can configure RTT debug interfaces - one for the NuttX console and one
for SEGGER SystemView:

#. Configure the RTT channel for the console::

     CONFIG_SERIAL_RTT0=y
     CONFIG_SERIAL_RTT_CONSOLE=y
     CONFIG_SERIAL_RTT_CONSOLE_CHANNEL=0

#. Disable the serial console on UART0::

     CONFIG_UART0_SERIAL_CONSOLE=n
     CONFIG_OTHER_SERIAL_CONSOLE=y
     CONFIG_ARCH_LOWPUTC=n

#. Enable instrumentation support::
  
     CONFIG_SCHED_INSTRUMENTATION=y
     CONFIG_SCHED_INSTRUMENTATION_IRQHANDLER=y
     CONFIG_DRIVERS_NOTE=y
     CONFIG_DRIVERS_NOTERAM=n

#. Turn on the hardware performance timer and configure task names::

     CONFIG_ARCH_PERF_EVENTS=y
     CONFIG_TASK_NAME_SIZE=32

#. Finally, enable SystemView on RTT channel 1::

     CONFIG_SEGGER_SYSVIEW=y
     CONFIG_SEGGER_SYSVIEW_RTT_CHANNEL=1
     CONFIG_SEGGER_SYSVIEW_RTT_BUFFER_SIZE=2048

The steps below configure NxScope on the RTT channel:

#. Disable UART0 support completely::

     CONFIG_NRF52_UART0=n
     CONFIG_STANDARD_SERIAL=n

#. Enable and configure an additional RTT serial port::

     CONFIG_SERIAL_RTT2=y
     CONFIG_SEGGER_RTT2_BUFFER_SIZE_DOWN=128
     CONFIG_SEGGER_RTT2_BUFFER_SIZE_UP=2048

#. Update the nxscope example configuration to work with RTT::
  
     CONFIG_EXAMPLES_NXSCOPE_SERIAL_PATH="/dev/ttyR2"
     CONFIG_EXAMPLES_NXSCOPE_SERIAL_BAUD=0

Now we can build our configuration::

  cmake --build build_nxscope_rtt

and flash it with nrfjprog::

  nrfjprog --program build_nxscope_rtt/nuttx.bin --chiperase --verify
  nrfjprog --reset

Plot data
=========

The command for the RTT interface looks as follow::

  nxscli rtt <chip_name> <rtt_channel> <rtt_up_buffer_len> ...

For our RTT configuration, it is::

  nxscli rtt nRF52832_xxAA 2 2048 ...

Let's plot real-time animation with 2000 data points for channel 16::

  nxscli rtt nRF52832_xxAA 2 2048 chan 16 pani2 2000

.. thumbnail:: /images/posts/2024/1/segger-rtt-interface-nxscope/1.png

.. thumbnail:: /images/posts/2024/1/segger-rtt-interface-nxscope/2.png

Press ``q`` to close the plot and stop the data stream.
