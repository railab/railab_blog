.. title: Hello, world!
.. slug: hello-world
.. date: 2024-01-01 12:00:00 UTC
.. tags: nikola, rst
.. category: Blog
.. description: The first post on this page is mainly for the purpose of verifying
   the generated HTML code
.. type: text
.. has_math: true


Hello, world!

Let's see what the website design looks like and how to use the rST with Nikola.
You probably won't find anything particularly interesting here.

Any tips and comments regarding the appearance of the website are welcome.

.. *****************************************************************************
.. Try to keep it withing 80 characters
.. *****************************************************************************

.. remember about teaser !

.. TEASER_END

Some usefull links:

- `The manual <https://docutils.sourceforge.io/rst.html>`_  for rST 

- `The handbook`__ for Nikola

  __ https://getnikola.com/handbook.html

- Bootswatch ``darkly`` page `[1]`_

  .. _[1]: https://bootswatch.com/darkly/

- Isso documentation: https://isso-comments.de/docs/

Notes:

1. Nikola uses rst2html for generating output from rST

===========
Admonitions
===========

Supported specific admonitions: ``attention``, ``caution``, ``danger``, ``error``,
``hint``, ``important``, ``note``, ``tip``, ``warning``.

.. note::
   :class: card

   this is a note

.. tip::

   this is a tip

.. warning::

   this is a warning

.. admonition:: Generic Admonition

   This is a generic admonition.

======
Tables
======

.. table:: Table 1
   :class: table table-primary
   :widths: grid

   == == === =====
   A  B  C   DEFG
   == == === =====
   x  y  0   xxx
   x  y  1   yyy
   == == === =====

.. table:: Table 2
   :class: table table-secondary
   :widths: auto

   +------------------------+------------+----------+
   | Some                   | table      | xxxxxx   |
   +========================+============+==========+
   | more                   | example    | yyyyyy   |
   +------------------------+------------+----------+
   | complex                | and here some text    |
   +------------------------+-----------------------+

.. csv-table:: CSV table
   :class: table
   :header: "One", "Two", "Tree"

   "AAAA", 0.99,    "xxxxx"
   "BBBB", 0.49,    "yyyyy"
   "CCCC", 0.00001, "zzzzz"

====
Code
====

We use pygments ``stata-dark`` color scheme::

  CODE_COLOR_SCHEME = 'stata-dark'

Here is the code block in Python:

.. code:: python

   def dummy(arg: int) -> int:
      print("hello")
      return 1

Here is the code block in C:

.. code:: C

    #include <stdio.h>

    int main(int argc, char *argv[])
      {
        printf("hello\n");
      }

====
Math
====

.. note::

   Needs ``.. has_math: true`` in metadata

.. math::

   Id = I_\alpha cos(\theta_e) + I_\beta sin(\theta_e)

   Iq = -I_\alpha sin(\theta_e) + I_\beta cos(\theta_e)

Inline Math example: :math:`e^{ix} = \cos x + i\sin x`

======
Images
======

Images from ``/images`` will be processed and have thumbnails created.

Below - thumbnails in a 1x2 grid:


.. |pic1| thumbnail:: /images/posts/2024/1/hello-world/1.jpg
   :alt: Pretty cool high-power inductors 1

.. |pic2| thumbnail:: /images/posts/2024/1/hello-world/2.jpg
   :alt: Pretty cool high-power inductors 2

.. table::
   :align: center

   +--------+--------+
   | |pic1| | |pic2| |
   +--------+--------+

.. force new lines with vertical bars

|
|

Here's a test image with the ``.. image::`` directive:

.. image:: /images/posts/2024/1/hello-world/ao.gif
   :align: center
   :alt: and this is a duck
   :target: https://railab.me
   :width: 150


==============
Header level 0
==============

Under level 0.

#. List item 1

   #. Under item 1
   #. Under item 1
   #. Under item 1

#. List item 2

   #. Under item 2
   #. Under item 2
   #. Under item 2

#. List item 3

   #. Under item 3
   #. Under item 3
   #. Under item 3

#. List item 4

   #. Under item 4
   #. Under item 4
   #. Under item 4

Level 1
=======

Under level 1.

* List item 1
* List item 2
* List item 3
* List item 4

Level 2
-------

Under level 2.

a) List item 1
b) List item 2
c) List item 3
d) List item 4

Level 3
^^^^^^^

Under level 3.

One asterisk for *emphasis*, two asterisks for **boldface**, backquotes for ``code samples``.

Level 4 - the lowest
""""""""""""""""""""

Under level 4.

Level 5 - the same as level 4
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Level 6 - the same as level 4
+++++++++++++++++++++++++++++

Under level 6.

Under level 6.
