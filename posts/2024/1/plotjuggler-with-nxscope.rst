.. title: Visualize data from NuttX NxScope with PlotJuggler
.. slug: plotjuggler-with-nxscope
.. date: 2024-01-14 12:00:00 UTC
.. tags: nxscope, nuttx, plotjuggler
.. category: howto
.. description: How to use PlotJuggler to visualise data from NuttX NxScope
.. type: text

`PlotJuggler <https://github.com/facontidavide/PlotJuggler>`_ is a cool tool for
visualizing time series data, supporting live data streaming.
Since version ``0.5.1``, my `nxscli <https://github.com/railab/nxscli>`_ tool 
has included support for data streaming over UDP port, which can be easily
captured by PlotJuggler.

Let's see how to set up both tools and plot data from the NuttX
`NxScope library <https://github.com/apache/nuttx-apps/tree/master/logging/nxscope>`_.

.. TEASER_END

==========================
Nxslib with a dummy device
==========================

The simplest way to test this new feature is by using built-in nxslib dummy device.
It allows you to simulate the NxScope protocol without the need to run the NuttX
application. This simplifies the development process with nxslib and nxscli tools.

Step by step instructions:

#. Start the nxscli with the dummy device enabled::

      nxscli dummy --streamsleep 0.1 chan 2,9 pudp 0
      
   In this case, select dummy channels 2 and 9 and limit the samples
   frequency with the ``--streamsleep 0.1`` option. The last argument ``... 0`` means
   that the stream will be generated until the Enter key is hit.

   .. thumbnail:: /images/posts/2024/1/plotjuggler-with-nxscope/1.png

#. Configure the PlotJuggler data source: select "UDP stream" and hit the "Start"
   button:

   .. thumbnail:: /images/posts/2024/1/plotjuggler-with-nxscope/2.png

   The default port of the UDP server is 9870, and the supported message protocol
   is ``json``.

   ``timestamp`` **must be** enabled and set to "timestamp".

   .. thumbnail:: /images/posts/2024/1/plotjuggler-with-nxscope/3.png

#. Start the stream on PlotJuggler; some values should appear
   in the "Timeseries List":

   .. thumbnail:: /images/posts/2024/1/plotjuggler-with-nxscope/4.png

#. Finally, split the plot tab horizontally and add the captured data:

   .. thumbnail:: /images/posts/2024/1/plotjuggler-with-nxscope/5.png

=============================
NxScope on a NuttX sim device
=============================

Now, let's communicate with the real NuttX instance. We'll stream data from the
NuttX application running in Linux simulator, so no special hardware is required,
only a machine with Linux or a compatible system.

Step by step instructions:

#. First, build the NuttX ``sim:nxscope`` configuration
   (`link <https://github.com/apache/nuttx/blob/master/boards/sim/sim/sim/configs/nxscope/defconfig>`_).
   Use the following command for CMake::

     cmake -B build_sim -DBOARD_CONFIG=sim:nxscope -GNinja
     cmake --build build_sim

   Verify that the build was successful:

     .. thumbnail:: /images/posts/2024/1/plotjuggler-with-nxscope/6.png

#. Next, establish a communication channel between the NuttX application and nxscli.
   Create a simualted UART device (blocking operation) with ::

     sudo socat PTY,link=/dev/ttySIM0 PTY,link=/dev/ttyNX0

   Then, configure the created PTY to RAW mode in another terminal (we use
   ``/dev/*`` path so privileged access is required)::

     sudo chmod a+rw /dev/ttySIM0
     sudo chmod a+rw /dev/ttyNX0
     stty -F /dev/ttySIM0 raw
     stty -F /dev/ttyNX0 raw

#. Run the NuttX binary and start the ``nxscope`` example application::

     ./build_sim/nuttx
     nsh> nxscope
     nxscope_samples_thr
     nxscope_charlog_thr
     nxscope_crichan_thr

#. Start nxscli::

     nxscli serial /dev/ttyNX0 chan 0 pudp 0

   This connects to the previously created serial device ``/dev/ttyNX0``.

#. Connect to the UDP stream and plot ``chan0`` from the ``Timeseries List``:

   .. thumbnail:: /images/posts/2024/1/plotjuggler-with-nxscope/7.png
