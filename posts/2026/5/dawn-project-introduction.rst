.. title: Dawn: Data Acquisition With Apache NuttX
.. slug: dawn-project-introduction
.. date: 2026-05-06 12:00:00 UTC
.. tags: nuttx, daq, dawn
.. category: Blog
.. description: Introduction to Dawn, a DAQ/control framework for Apache NuttX.
.. type: text

.. image:: /images/posts/2026/5/dawn-project-introduction/dawn-logo.svg
   :alt: Dawn project logo
   :align: center
   :width: 360


Dawn is a new open-source project for building embedded data acquisition and
control devices on top of Apache NuttX.

It started from a common problem: many embedded projects need the same kind of
firmware infrastructure, and too much of that work still gets rebuilt from
scratch. I wanted to turn more of that repeated work into something reusable.

Dawn is the result of that effort. It took much longer than I expected,
changed shape several times, and grew far beyond the original idea, but it has
finally reached a point where it is worth introducing publicly.

.. TEASER_END

Motivation and history
======================

Dawn is a new project, but some of its roots go back to my master's thesis. Back
then, I designed and built a pulse generator for WEDM (`electrical discharge
machining <https://en.wikipedia.org/wiki/Electrical_discharge_machining>`_):
a 1.5 kW full-bridge switching power supply with an additional pulse-forming
block, controlled by an STM32F334 microcontroller.

That project made me think about a more universal platform for similar
power-electronics applications: a reusable control board and firmware built from
common blocks. I called that idea UniSP (Universal SMPS Platform).

The reusable control board was built, but the general firmware was left for
later. During the thesis project there wasn't enough time for that part,
especially because a lot of time went into debugging high-power switching
issues. Many MOSFETs died along the way :)

After finishing my studies, my interest in power electronics slowly moved into
other topics. I put UniSP aside, but the idea of building embedded devices from
reusable firmware blocks stayed with me.

.. thumbnail:: /images/posts/2026/5/dawn-project-introduction/unisp-before-dawn-idea.jpg
   :alt: Full-bridge power supply with attached UniSP module

   Full-bridge power supply with UniSP module.

Later, near the beginning of my embedded career, I was introduced to NuttX
through a task at my first job. What caught my attention was its modular
architecture, standardized APIs, and the possibility of writing applications
that are more portable and reusable than typical project-specific embedded
firmware. This matched the kind of software I always liked: modular
applications, reusable tools, and systems that help build other systems.

Much later, in October 2023, I created a NuttX issue that became the first
direct step toward Dawn: `apache/nuttx#11088
<https://github.com/apache/nuttx/issues/11088>`_. At that point, this was
still supposed to be just a NuttX application. I wanted to demonstrate NuttX
on Nordic Thingy boards (Thingy:52, Thingy:53, and Thingy:91), while keeping
the application generic enough to also work on similar sensor boards, such
as ST SensorTile.

The Thingy application didn't end up in NuttX. The challenge was bigger
than just a demo application: how to describe sensors, generated values,
configuration, local processing, and communication protocols in a reusable way?

That is where Dawn started. The name is a simple acronym::

   Data Acquisition With NuttX.

At first, the goal was simply to build embedded node devices from reusable
parts. The details changed over time, but the direction stayed the same.

I even kept an initial draft of this post in my local repository for almost two
years. The problem was much bigger than I initially thought, and everything was
built in my spare time. The design kept changing, important missing pieces were
added to NuttX along the way, and new use cases kept showing up.

.. thumbnail:: /images/posts/2026/5/dawn-project-introduction/first-dawn-mention-on-this-blog.png
   :alt: The first mention of Dawn project on this blog repository

The project kept growing while I was trying to make the overall direction clear.
I kept adding more and more things, and it started to feel like a never-ending
spare-time project. At some point, the hard part was no longer adding another
feature, but keeping the whole project testable and moving forward.

For a project like Dawn, manual testing is not realistic. There are too many
components that require different test setups. NTFC (`NuttX Testing Framework
<https://github.com/apache/nuttx-ntfc>`_) became the missing piece here: it made
it possible to automate complex integration tests on the NuttX simulator, QEMU,
and hardware-oriented configurations.

This is how Dawn reached its current shape: reusable embedded building blocks,
NuttX portability, automated testing, and tools that make the project manageable.
The goal is still the same as at the beginning: spend less time rebuilding
firmware infrastructure and more time building funny things.

Project scope
=============

Dawn targets descriptor-defined embedded nodes: devices that read inputs, write
outputs, run local processing, and expose selected data or control points
through one or more protocols.

This model fits many DAQ/control-style devices, from simple sensor or actuator
nodes to custom lab tools, protocol demos, and test devices.

Basic model
-----------

Dawn devices are composed from three main object families:

- IO objects,
- Program objects,
- Protocol objects.

IO objects represent data sources, outputs, configuration values, control
points, or virtual containers.

Program objects implement local processing, routing, filtering, buffering, and
other kinds of work that can be done on the edge.

Protocol objects expose selected IO and program data to the outside world
through interfaces such as shell, BLE, Modbus, CAN, and others. In some cases,
the "outside world" can also mean another application in the same firmware image
or in an AMP system, for example, a GUI application.

Descriptors
-----------

Dawn devices are defined by descriptors. A descriptor defines which IO objects,
programs, and protocols exist, how they are configured, how they are connected,
and how the device is structured internally.

Internally, a descriptor is just a 32-bit-word array. This is the format used by
the firmware runtime itself and a key element of the entire architecture.

On top of this format, Dawn provides higher-level representations for
development and tooling. YAML is the main human-friendly format, while generated
C++ source is used during the firmware build process. In the initial phase of
the project, I wrote the descriptors in C++, but the introduction of YAML
completely changed the game.

For example, a small blinky device controlled over Modbus RTU can be described
like this:

.. code:: yaml

   metadata:
     title: Blinky Modbus RTU Demo
     os_required:
       - /dev/leds0
       - /dev/ttyS1

   ios:
     - id: led1
       type: leds
       dtype: uint32
       config:
         device: 0

     - id: ctrl_blinky
       type: control
       config:
         targets:
           - blinky_seq1
         allowed:
           - start
           - stop

   programs:
     - id: blinky_seq1
       type: sequencer
       config:
         targets:
           - led1
         states:
           - value: 0
             dwell_us: 500000
           - value: 1
             dwell_us: 500000

   protocols:
     - id: modbus1
       type: modbus_rtu
       config:
         path: /dev/ttyS1
         baudrate: 115200
         registers:
           - type: coil
             start: 0
             bindings:
               - ctrl_blinky
           - type: holding
             start: 0x0100
             bindings:
               - led1

This descriptor creates an LED IO object, a sequencer program that toggles the
LED, and a Modbus RTU protocol object that exposes control and state to an
external master.

The normal build flow converts YAML into generated C++, and then into the final
binary descriptor embedded into the firmware image.

Descriptors can also be loaded and switched dynamically at runtime, although
this is still an experimental feature. The long-term direction is to make
descriptors more independent from firmware images and enable more dynamic device
configuration workflows.

Development ecosystem
---------------------

Dawn is more than firmware runtime code. A significant part of the project lives
in the surrounding development ecosystem: Python tooling for descriptor
handling, project and build management, transport-specific host tools, and QA
infrastructure.

That ecosystem is what makes the descriptor-based approach practical in day-to-day
development. It covers descriptor validation and generation, project setup and
build helpers, communication with running devices over supported transports, and
repeatable automated testing across simulator, QEMU, and hardware targets.

It also supports out-of-tree, user-specific extensions, so custom objects and
project-specific integrations can be added without forking the main project.
Over time, I also want this ecosystem to grow toward template generators for
common device patterns, producing YAML descriptors as a starting point instead
of writing everything by hand.

Implementation and tradeoffs
----------------------------

The firmware part is written in C++. The project uses object-oriented
abstractions quite heavily, but within fairly strict limits. C++ features are
used with care: no exceptions, no dynamic allocation during normal runtime,
and a general preference for deterministic and reliable behavior. Dynamic
allocation is allowed only during the initial startup phase.
Host-side tools are written in Python, because it is my main language for host
applications.

Dawn adds an abstraction layer on top of Apache NuttX, and that has a cost:
more memory usage and some CPU overhead. The goal is to keep these costs as low
as possible using the usual NuttX approach: make components configurable with
Kconfig and compile in only what is needed.

In this case, having many Kconfig options is not a flaw but part of the design.
Some parts of Dawn are difficult for the compiler to eliminate automatically,
so explicit configuration is needed to keep firmware size under control.

Where the project is now
========================

Dawn is still early-stage, but it is already useful for experiments, rapid
prototyping, simulator-based development, protocol demos, and custom
development or test tools.

There are already enough building blocks to create useful applications:

- `IO objects <https://railab.github.io/dawn/components/io/index.html#supported-io>`_
- `Programs <https://railab.github.io/dawn/components/prog/index.html#supported-programs>`_
- `Protocols <https://railab.github.io/dawn/components/proto/index.html#supported-protocols>`_

I'm not sure yet what Dawn's long-term role will be. It may remain mostly a
developer and lab tool, or it may grow into a base for more product-oriented
systems.

For now, the focus is more practical: more ready-to-use descriptors, more
examples, and more features for building custom DAQ/control devices. At some
point, this may also mean returning to the UniSP idea mentioned earlier and
trying to realize it with the more general approach now available with Dawn.

The public roadmap is available in the documentation:

- `Dawn roadmap <https://railab.github.io/dawn/roadmap.html>`_

The private idea list is much longer. If this topic is interesting to you,
testing, feedback, and contributions are welcome.

Links
=====

This post is only a short introduction. For details, see the documentation and
source code:

- `Dawn GitHub repository <https://github.com/railab/dawn>`_
- `Dawn documentation <https://railab.github.io/dawn/>`_
