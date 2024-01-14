.. title: HBI3260-2C integrated servo teardown
.. slug: hbi32602c-servo-teardown
.. date: 2024-01-07 12:00:00 UTC
.. tags: servo, FOC
.. category: Teardown
.. description: HBI3260-2C integrated servo teardown
.. type: text

.. thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/1.jpg
   :alt: And this is our patient

In this post, we'll take a look inside the HBI 3260-2C integrated servo that I
bought for pennies some time ago.

.. TEASER_END

The first thing I usually do before disassembling the device is conduct a brief
research to find any related documentation. I discovered that this servo was part of the
`Mettler Toledo XS3 <https://www.mt.com/my/en/home/phased_out_products/Product-Inspection_1/checkweighing/checkweigher-x-series/xs3.html>`_
`checkweigher machine <https://en.wikipedia.org/wiki/Check_weigher>`_
(part number - ``MT-GA Prt. No. 24133741``).

Despite the device being labeled Mettler-Toledo, the manufacturer is
ENGEL Elektroantriebe. Upon examining Engle products, I found a similar device with
the same markings, but a different connector. The Manual in PDF can be found
`here <https://www.engelantriebe.de/pdf/HBI22_37_BA_engl.pdf>`_.
I see two explanations for this: either the part I have is some kind of old design,
or it's a device specially manufactured for Mettler-Toledo with potentially limited
functionality.

Now, let's see what's inside.

.. thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/2.jpg

We can observe neatly arranged electronics based on rigid-flex PCB design, featuring
high quality electrolytic capacitors from Nippon Chemi-Con glued to the enclosure
to prevent vibration.
The power transistors on the bottom board are covered with some kind of thermal
conductive material.
In the center, there's a board with most likely a magnetic encoder.

.. thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/3.jpg

Motor connectors are soldered directly to the PCB. In the middle of the PCB,
there are screws pressing down the power transistors, securing the entire PCB in place.
The first shunt resistor for measuring motor phase currents is visible in the
lower right corner. On the left, there are half-bridge drivers from Intersil - ISL2101.

Take a closer look at the MCU.

.. thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/4.jpg

It's an old-school DSP from TI - `TMS320LF2406A <https://www.ti.com/product/TMS320LF2406A>`_.
In the top left corner of the DSP we can observe the EPCOS common-mode choke for
CAN network.

..  thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/5.jpg

Here, we have a magnetic encoder from IMM Photonic - IC-MH
(`PDF <https://www.imm-photonics.de/wp-content/uploads/2021/09/MH_datasheet_C2en.pdf>`_)

After a quick analysis of the electronics inside, I managed to find the M12 connector
pinout necessary to start the motor:

.. thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/6.jpg

I connected the power supply, set the ``ENABLE`` pin high (+24V) and the
motor started spinning. Unfortunately, there was no traffic on the CAN bus.
This, I conclude that OpenCAN interface was permanently disabled, and the servo is
configured in constant speed mode (according to the manual I found, it's possible).
I don't have access to Engel ``DSerV`` configuration software, so there's
nothing more I can do here. A bit of a bummer, as OpenCAN support was the main
reason I bought this motor in the first place :)

In this case, we can proceed with the teardown...

After some gymnastics with desoldering motor wires and unscrewing Torx screws,
we can see the entire PCB design.

.. thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/7.jpg

Now, we can finally see the other side of the board:

.. thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/8.jpg

Interesting chips here:

* RS232 transceiver - `MAX3323 <https://www.analog.com/en/products/max3323e.html>`_

* CAN transceiver (marked as VP233) - `SN65HVD233 <https://www.ti.com/product/SN65HVD233?qgpn=sn65hvd233>`_

* Current sense opamp - `OPA4364 <https://www.ti.com/product/OPA4364>`_

* Serial EEPROM - `AT24C16D <https://www.microchip.com/en-us/product/at24c16d>`_

We can spot the second shunt resistor, so this inverter is in two-shunt low-side
current sensing topology.

.. thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/9.jpg

Power transistors from Infineon: `IPA040N06N <https://www.infineon.com/cms/en/product/power/mosfet/n-channel/ipa040n06n/>`_.

There's nothing interesting left in the servo housing.

.. thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/10.jpg

In the future, I'll try to run the motor with a NuttX-based controller.

.. thumbnail:: /images/posts/2024/1/hbi32602c-servo-teardown/11.jpg

   B-G431B-ESC1 fits quite well here, don't you think?
