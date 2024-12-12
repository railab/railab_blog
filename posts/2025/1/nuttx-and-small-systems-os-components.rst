.. title: Apache NuttX and small systems - OS components
.. slug: nuttx-and-small-systems-os-components
.. date: 2025-01-28 12:00:00 UTC
.. tags: nuttx, small systems
.. category: Blog
.. description: Analyzing OS component sizes in Apache NuttX
.. type: text

.. thumbnail:: /images/posts/2025/1/nuttx-and-small-systems-os-components/1.png
   :alt: A little hamster with the NuttX logo on its back
   :align: center

In earlier posts in this series, we explored the resource requirements of trivial
examples. Now, it's time to dive deeper and examine how specific NuttX interfaces
affect resource consumption in small embedded systems.

As part of this analysis, we'll develop small applications that utilize specific
NuttX features and evaluate the memory footprint of each. This knowledge can be
useful when designing new applications, helping to define initial constraints and
requirements for the OS interfaces.

.. TEASER_END

.. note::
   :class: card

   All presented results are for Apache NuttX with the following commits:

   * nuttx: ``6d629b3b36105e34f97f0439af5acaa8543c78a7``
   * nuttx-apps: ``0c467dc02d1f03f3f9f3defb16f36cb4f53b4c9d``

   Toolchain: ``arm-none-eabi-gcc (Arch Repository) 13.2.0``

=============
OS components
=============

Before we begin, it's important to note that the final memory usage of
an application depends on several factors. These include how a specific OS
interface is used, the compiler version and its optimization settings, and many
complex interactions between enabled features. While the tests presented here
won't cover every possible scenario, they provide a solid starting point for
understanding the memory cost of adding new functionality to an OS image.

Testing all available NuttX features for resource consumption is a massive and
likely impractical task. That's why only the features most appropriate for small
systems are selected here. They are limited to those that, based on my
experience, are most suitable for small applications.

Some features are better suited for small systems than others, but thanks to
NuttX's modular design, we can precisely control the available interfaces
through Kconfig options. The assumptions regarding the tested components are as
follows:

#. Ignore all debug features: Debugging features are excluded to minimize
   resource usage. However, we use the console over the serial port because our
   examples are based on ``printf()``.
   
#. Focus on single-core, single-task use cases with optional multi-thread support:
   Most IPC mechanisms and features designed for SMP applications are
   excluded.

#. No shell support: NSH is disabled, and the system boots directly to the
   application.

#. Adhere to POSIX standard: There are no deliberate violations of POSIX
   standard to save memory.

With this in mind, five simple applications will be presented here:

* Application 1: Impact of the ``CONFIG_DEFAULT_SMALL`` option.

* Application 2: Waiting for events from files.

* Application 3: Testing useful Pthread features.

* Application 4: Comparing signaling methods.

* Application 5: ``printf()`` and ``libm`` size.

Each application is designed so that we can easly enable various components
and observe their influence on memory requirements. Application test begins
with a *"base code"* where all tested features are disabled.
We don't include error handling in the code; all examples are kept as simple
as possible.

All applications are based on the same core configuration, which is the result
of the earlier experiments done in the `Hello, World! <link://slug/nuttx-and-small-systems-hello-world>`__:

.. code:: shell

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
   CONFIG_ARCH_MINIMAL_VECTORTABLE=y
   CONFIG_ARCH_MINIMAL_VECTORTABLE_DYNAMIC=y
   CONFIG_ARCH_NUSER_INTERRUPTS=5
   CONFIG_BOARD_LOOPSPERMSEC=16717
   CONFIG_DEBUG_FULLOPT=y
   CONFIG_DEBUG_SYMBOLS=y
   CONFIG_DEFAULT_SMALL=y
   CONFIG_DEFAULT_TASK_STACKSIZE=512
   CONFIG_IDLETHREAD_STACKSIZE=512
   CONFIG_INIT_ENTRYPOINT="componentsX_main" # app dependent
   CONFIG_INTELHEX_BINARY=y
   CONFIG_LTO_FULL=y
   CONFIG_NAME_MAX=0
   CONFIG_NFILE_DESCRIPTORS_PER_BLOCK=3
   CONFIG_PATH_MAX=32
   CONFIG_PID_INITIAL_COUNT=3
   CONFIG_RAILAB_MINIMAL_COMPONENTSx=y       # app dependent
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

The base code for applications, used as a reference point looks like this:

.. code:: C

  int main(int argc, char *argv[])
  {
    printf("componentsX examples\n");

    while (1)
      {
        sleep(1);
      }

    return 0;
  }

Any changes in configuration and code for specific examples are documented in
their respective sections. All applications, along with ready-to-use configurations,
can be found in the `railab NuttX examples <https://github.com/railab/railab_nuttx_code/>`_
repository.

Be preapre for a lot of memory usage reports here. For better readability, the results
are also summarized in tables.

Application 1: Impact of the ``CONFIG_DEFAULT_SMALL`` option
============================================================

`Application 1 Sources <https://github.com/railab/railab_nuttx_code/blob/master/apps/mini_components1>`__

Earlier examples in this series relied heavily on the ``CONFIG_DEFAULT_SMALL``
option without delving into its specific effects. This time, let's look at
the individual components it affects.

We start with the base configuration and the code from above, which gives us
the following initial report:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       18256 B        64 KB     27.86%
              sram:        1268 B        16 KB      7.74%

Now let's disable individual options that are enabled by default
with ``DEFAULT_SMALL``:

#. Enable Environment variables with ``CONFIG_DISABLE_ENVIRON=n``

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       19924 B        64 KB     30.40%
                 sram:        1284 B        16 KB      7.84%

#. Enable POSIX timers with ``CONFIG_DISABLE_POSIX_TIMERS=n``

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       18556 B        64 KB     28.31%
                 sram:        1540 B        16 KB      9.40%

#. Enable Pthreads with ``CONFIG_DISABLE_PTHREADS=n``

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       21368 B        64 KB     32.60%
                 sram:        1348 B        16 KB      8.23%

#. Enable POSIX message queue with ``CONFIG_DISABLE_MQUEUE=n``

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       18440 B        64 KB     28.14%
                 sram:        1852 B        16 KB     11.30%

#. Enable System V message queue with ``CONFIG_DISABLE_MQUEUE_SYSV=n``

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       18352 B        64 KB     28.00%
                 sram:        1484 B        16 KB      9.06%

#. Enable all above with ``CONFIG_DISABLE_OS_API=n``:

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       23580 B        64 KB     35.98%
                 sram:        2204 B        16 KB     13.45%

#. Enable FILE stream with ``CONFIG_FILE_STREAM=y``:

   .. code:: shell

     Memory region         Used Size  Region Size  %age Used
                flash:       19132 B        64 KB     29.19%
                 sram:        1604 B        16 KB      9.79%

Summary
-------

The difference between the base configuration and the individual options is
show below:

.. table:: Table 1: OS interfces support
   :class: table table-primary
   :widths: auto

   +------------------------------------+------------+----------+
   | Test case                          | FLASH      | SRAM     |
   +====================================+============+==========+
   | Base image                         | 18256 B    | 1268 B   |
   +------------------------------------+------------+----------+
   | CONFIG_DISABLE_ENVIRON=n           | +1668 B    | +16 B    |
   +------------------------------------+------------+----------+
   | CONFIG_DISABLE_POSIX_TIMERS=n      | +300 B     | +272 B   |
   +------------------------------------+------------+----------+
   | CONFIG_DISABLE_PTHREADS=n          | +3112 B    | +80 B    |
   +------------------------------------+------------+----------+
   | CONFIG_DISABLE_MQUEUE=n            | +184 B     | +584 B   |
   +------------------------------------+------------+----------+
   | CONFIG_DISABLE_MQUEUE_SYSV=n       | +96 B      | +216 B   |
   +------------------------------------+------------+----------+
   | CONFIG_DISABLE_OS_API=n            | +5324 B    | +936 B   |
   +------------------------------------+------------+----------+
   | CONFIG_FILE_STREAM=y               | +876 B     | +336 B   |
   +------------------------------------+------------+----------+

Pthread support is the most significant factor here, so it's always worth
evaluating whether our small application truly requires threads or can be
designed as a single-threaded program.

Message queues and environment variables are likely unnecessary in small,
single-task applications. Support for ``FILE *`` may introduce unnecessary
overhead unless stream functionality is essential in our application.

On the other hand, POSIX timers can be useful in small applications and
their initial cost is minimal. We'll explore this later.

Application 2: Waiting for events from files
============================================

`Application 2 Sources <https://github.com/railab/railab_nuttx_code/blob/master/apps/mini_components2>`__

Any practical application requires interaction with the outside world. In the
case of POSIX, the portable interface is provided by the abstraction of files.

Choosing the right method to wait for file events is crucial for many
small applications. Here, we test three methods for waiting on file data:

#. Blocking wait with ``read()``,

#. ``poll()`` interface,

#. ``epoll()`` interface.

Although ``epoll()`` is not part of POSIX interface, it's supported in NuttX,
making it worth testing. As the source of events, we use file descriptors
created with the ``timerfd_create()``.

Base image
----------

The base code in this case is the same as in the introduction, but from
the beginning, we update the configuration to support additional file
descriptors:

.. code:: shell

   CONFIG_NFILE_DESCRIPTORS_PER_BLOCK=5

This gives us:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       18252 B        64 KB     27.85%
              sram:        1300 B        16 KB      7.93%

Step 1 - add TimerFD support
----------------------------

From now we have to enable ``TimerFD`` support with:

.. code:: shell

  CONFIG_TIMER_FD=y

We add two timer instances to the code, which wake up periodically
every 2 seconds, and use busy-wait to read from them:

.. code:: C

  static int timer_read(int fd)
  {
    timerfd_t tdret;
    int       ret;

    ret = read(fd, &tdret, sizeof(timerfd_t));
    if (ret > 0)
      {
        printf("fd=%d\n", fd);
      }

    return ret;
  }

  int main(int argc, char *argv[])
  {
    struct itimerspec tms;
    int               tfd1;
    int               tfd2;
    int               ret;

    printf("components2 examples\n");

    tfd1 = timerfd_create(CLOCK_MONOTONIC, 0);
    tfd2 = timerfd_create(CLOCK_MONOTONIC, 0);

    tms.it_value.tv_sec     = 2;
    tms.it_value.tv_nsec    = 0;
    tms.it_interval.tv_sec  = 2;
    tms.it_interval.tv_nsec = 0;

    timerfd_settime(tfd1, 0, &tms, NULL);
    timerfd_settime(tfd2, 0, &tms, NULL);

    while (1)
      {
        timer_read(tfd1);
        timer_read(tfd2);

        sleep(1);
      }

    return 0;
  }
  
The result is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       19424 B        64 KB     29.64%
              sram:        1332 B        16 KB      8.13%

Step 2 - wait for events with ``poll()``
----------------------------------------

Now, we modify the loop code to wait for notifications using the ``poll()``
interface:

.. code:: C

  struct pollfd fds[2];
  memset(fds, 0, sizeof(struct pollfd)*2);

  fds[0].fd      = tfd1;
  fds[0].events  = POLLIN;

  fds[1].fd      = tfd2;
  fds[1].events  = POLLIN;

  while (1)
    {
      fds[0].revents = 0;
      fds[1].revents = 0;

      ret = poll(fds, 2, -1);
      if (ret > 0)
        {
          if (fds[0].revents == POLLIN)
            {
              timer_read(tfd1);
            }

          if (fds[1].revents == POLLIN)
            {
              timer_read(tfd2);
            }
        }

      sleep(1);
    }

The result is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       19832 B        64 KB     30.26%
              sram:        1332 B        16 KB      8.13%

Step 3 - wait for events with ``epoll()``
-----------------------------------------

Finally, we use ``epoll()`` to handle events from files:

.. code:: C

  struct epoll_event ev;
  struct epoll_event events[MAXFDS];
  int                epollfd;

  epollfd = epoll_create1(EPOLL_CLOEXEC);

  ev.events = EPOLLIN;
  ev.data.fd = tfd1;
  epoll_ctl(epollfd, EPOLL_CTL_ADD, tfd1, &ev);

  ev.events = EPOLLIN;
  ev.data.fd = tfd2;
  epoll_ctl(epollfd, EPOLL_CTL_ADD, tfd2, &ev);

  while (1)
    {
      ret = epoll_wait(epollfd, events, MAXFDS, -1);
      if (ret > 0)
        {
          if (events[0].events == POLLIN)
            {
              timer_read(events[0].data.fd);
            }

          if (events[1].events == POLLIN)
            {
              timer_read(events[1].data.fd);
            }
        }

      sleep(1);
    }

In this case we get:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       20736 B        64 KB     31.64%
              sram:        1364 B        16 KB      8.33%

Summary
-------

The difference between the base configuration and subsequent modifications is
shown below:

.. table:: Table 2: File descriptor events handling
   :class: table table-primary
   :widths: auto

   +------------------------------------+------------+----------+
   | Test case                          | FLASH      | SRAM     |
   +====================================+============+==========+
   | Base image                         | 18252 B    | 1300 B   |
   +------------------------------------+------------+----------+
   | TimerFD with busy read             | +1172 B    | +32 B    |
   +------------------------------------+------------+----------+
   | TimerFD with poll                  | +1580 B    | +32 B    |
   +------------------------------------+------------+----------+
   | TimerFD with epoll                 | +2484 B    | +64 B    |
   +------------------------------------+------------+----------+

Using ``poll()`` doesn't cost much compared to busy read, while the overhead of
``epoll()`` is three times higher than that of ``poll()``. However, the usability
of ``epoll()`` in small systems is questionable, as it's unlikely that a large
number of file descriptors will be used, making the benefits of this feature
negligible.

Application 3: Testing useful Pthread features
==============================================

`Application 3 Sources <https://github.com/railab/railab_nuttx_code/blob/master/apps/mini_components3>`__

Now, let's move on to multithreaded applications. When multiple threads are
needed, they often share common resources that require protection.
Additionally, signaling changes in shared resources can be useful feature.
Let's check how much memory this costs.

Base image
----------

The base image has been slightly modified:

.. code:: C

  int main(int argc, char *argv[])
  {
    printf("components3 examples\n");

    while (g_val < 3)
      {
        sleep(1);
      }

    printf("done!\n");

    return 0;
  }

This simple program waits in a loop until the global counter passes some value.
Currently, this code is stuck in a loop because there is nothing that updates
the counter.

We enable Pthreads support from the beginning:

.. code:: shell

   # CONFIG_DISABLE_PTHREAD is not set

The initial memory report is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       21364 B        64 KB     32.60%
              sram:        1348 B        16 KB      8.23%

Step 1: add threads
-------------------

First of all, we add threads to the code that simply print a message:

.. code:: C

  #define THREADS 3
  static pthread_t g_th[THREADS];

  static void *thread(void *data)
  {
    int id  = (int)((intptr_t)data);

    printf("hello from %d\n", id);

    return NULL;
  }

  static void threads_init(void)
  {
    pthread_attr_t attr;

    for (int i = 0; i < THREADS; i++)
      {
        pthread_attr_init(&attr);
        attr.priority = PTHREAD_DEFAULT_PRIORITY - i;
        pthread_create(&g_th[i], &attr, thread, (pthread_addr_t)i);
      }
  }

  int main(int argc, char *argv[])
  {
    printf("components3 examples\n");

    threads_init();

    while (g_val < 3)
      {
        sleep(1);
      }

    printf("done!\n");

    return 0;
  }

Memory report for this code is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       21516 B        64 KB     32.83%
              sram:        1348 B        16 KB      8.23%

Step 2: use atomic variable
---------------------------

The first data protection method we test is the use of atomic variables.
In this way, operations on the counter are atomic, so there are no race
conditions. The code is:

.. code:: C

  static atomic_uint g_val;

  static void *thread(void *data)
  {
    int id  = (int)((intptr_t)data);

    printf("hello from %d\n", id);

    atomic_fetch_add(&g_val, 1);

    return NULL;
  }

  int main(int argc, char *argv[])
  {
    printf("components3 examples\n");

    atomic_init(&g_val, 0);

    threads_init();

    while (atomic_load(&g_val) < 3)
      {
        sleep(1);
      }

    printf("done!\n");

    return 0;
  }

This version of code gives us:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       21592 B        64 KB     32.95%
              sram:        1364 B        16 KB      8.33%

Step 3: use mutex
-----------------

In this scenario we use mutex to protect the ``uint32_t`` counter:

.. code:: C

  static uint32_t g_val;
  static pthread_mutex_t g_mut;

  static void *thread(void *data)
  {
    int id  = (int)((intptr_t)data);

    printf("hello from %d\n", id);

    pthread_mutex_lock(&g_mut);
    g_val++;
    pthread_mutex_unlock(&g_mut);

    return NULL;
  }

  int main(int argc, char *argv[])
  {
    uint32_t tmp;

    printf("components3 examples\n");

    pthread_mutex_init(&g_mut, NULL);

    threads_init();

    do
      {
        sleep(1);

        pthread_mutex_lock(&g_mut);
        tmp = g_val;
        pthread_mutex_unlock(&g_mut);
      } while (tmp < 3);

    printf("done!\n");

    return 0;
  }

The memory report is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       21668 B        64 KB     33.06%
              sram:        1380 B        16 KB      8.42%

Step 4: use condition variable
------------------------------

This time we use a condition variable to signal the change of the counter value:

.. code:: C

  static uint32_t g_val = 0;
  static pthread_mutex_t g_mut;
  static pthread_cond_t  g_cond;

  static void *thread(void *data)
  {
    int id  = (int)((intptr_t)data);

    printf("hello from %d\n", id);

    pthread_mutex_lock(&g_mut);
    g_val++;
    pthread_cond_signal(&g_cond);
    pthread_mutex_unlock(&g_mut);

    return NULL;
  }

  int main(int argc, char *argv[])
  {
    printf("components3 examples\n");

    pthread_cond_init(&g_cond, NULL);
    pthread_mutex_init(&g_mut, NULL);

    threads_init();

    pthread_mutex_lock(&g_mut);

    while (g_val < 3)
      {
        pthread_cond_wait(g_cond, &g_mut);
      }

    pthread_mutex_unlock(&g_mut);

    printf("done!\n");

    return 0;
  }

Additionally, we have to increase the default stack size:

.. code:: shell
 
   CONFIG_DEFAULT_TASK_STACKSIZE=640

The result is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       21668 B        64 KB     33.06%
              sram:        1396 B        16 KB      8.52%

Step 4: use rwlock
------------------

And finally, we use ``rwlock`` to protect resources:

.. code:: C

  static uint32_t g_val = 0;
  static pthread_rwlock_t g_rw;

  static void *thread(void *data)
  {
    int id  = (int)((intptr_t)data);

    printf("hello from %d\n", id);

    pthread_rwlock_wrlock(&g_rw);
    g_val++;
    pthread_rwlock_unlock(&g_rw);

    return NULL;
  }

  int main(int argc, char *argv[])
  {
    uint32_t tmp;

    printf("components3 examples\n");

    pthread_rwlock_init(&g_rw, NULL);

    threads_init();

    do
      {
        sleep(1);

        pthread_rwlock_rdlock(&g_rw);
        tmp = g_val;
        pthread_rwlock_unlock(&g_rw);
      } while (tmp < 3);

    printf("done!\n");

    return 0;
  }

The default stack size is also increased in this case:

.. code:: shell
 
   CONFIG_DEFAULT_TASK_STACKSIZE=640

Which gives us:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       21992 B        64 KB     33.56%
              sram:        1412 B        16 KB      8.62%

Summary
-------

The difference between the base configuration and subsequent modifications is
shown below:

.. table:: Table 3: Thread resources protection
   :class: table table-primary
   :widths: auto

   +------------------------------------+------------+----------+
   | Test case                          | FLASH      | SRAM     |
   +====================================+============+==========+
   | Base image                         | 21364 B    | 1348 B   |
   +------------------------------------+------------+----------+
   | Create threads                     | +152 B     | +0 B     |
   +------------------------------------+------------+----------+
   | Data with atomic                   | +228 B     | +16 B    |
   +------------------------------------+------------+----------+
   | Data with mutex                    | +304 B     | +32 B    |
   +------------------------------------+------------+----------+
   | Data with condition variable       | +304 B     | +48 B    |
   +------------------------------------+------------+----------+
   | Data with rwlock                   | +628 B     | +64 B    |
   +------------------------------------+------------+----------+

Interestingly, the version with mutex and the version with mutex and conditional
variable take up the same amount of FLASH.

The overhead of the data protection API and conditional variables is small.
Most of these features are already used in the kernel, so the logic for these
mechanisms is alredy in the image. The readers-writer lock is the most advanced
mechanism, and it's also the heaviest.

Application 4: Comparing signaling methods
==========================================

`Application 4 Sources <https://github.com/railab/railab_nuttx_code/blob/master/apps/mini_components4>`__

In this section we want to see the overhead of signals generated with the POSIX
timer and compare it with simple signaling using a semaphore.

Base image
----------

In our base code, we simply wait for the semaphore to be released. This time,
we check the return error code to catch interruptions caused by signals:

.. code:: C

  static sem_t g_sem;

  int main(int argc, char *argv[])
  {
    int ret;

    printf("components4 examples\n");

    while (1)
      {
        ret = sem_wait(&g_sem);
        if (ret < 0)
          {
            if (errno == EINTR)
              {
                printf("EINTR\n");
              }
          }
        else
          {
            printf("sem!\n");
          }
      }

    return 0;
  }

The result is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       18236 B        64 KB     27.83%
              sram:        1284 B        16 KB      7.84%

Step 1: thread with semaphore
-----------------------------

In the first test we wake up the main with a simple thread:

.. code:: C

  static void *thread(void *data)
  {
    while (1)
      {
        sleep(1);
        sem_post(&g_sem);
      }

    return NULL:
  }

  int main(int argc, char *argv[])
  {
    pthread_t th;
    int ret;

    printf("components4 examples\n");

    pthread_create(&th, NULL, thread, &sync);

    while (1)
      {
        ret = sem_wait(&g_sem);
        if (ret < 0)
          {
            if (errno == EINTR)
              {
                printf("EINTR\n");
              }
          }
        else
          {
            printf("sem!\n");
          }
      }

    return 0;
  }

We have to enable threads support:

.. code:: shell

   # CONFIG_DISABLE_PTHREAD is not set

This gives us:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       21464 B        64 KB     32.75%
              sram:        1348 B        16 KB      8.23%

Step 2: POSIX timer with signal
-------------------------------

This time, we use a POSIX timer to periodically interrupt waiting for the
semaphore. We don't care about catching the signal here. ``sem_wait()`` will be
interrupted with an ``EINTR`` error, and that's sufficient:

.. code:: C

  int main(int argc, char *argv[])
  {
    struct sigevent notify;
    struct itimerspec timer;
    timer_t timerid;
    int ret;

    printf("components4 examples\n");

    notify.sigev_notify            = SIGEV_SIGNAL;
    notify.sigev_signo             = SIGRTMIN;
    notify.sigev_value.sival_int   = 0;

    timer_create(CLOCK_REALTIME, &notify, &timerid);

    timer.it_value.tv_sec     = 2;
    timer.it_value.tv_nsec    = 0;
    timer.it_interval.tv_sec  = 2;
    timer.it_interval.tv_nsec = 0;

    timer_settime(timerid, 0, &timer, NULL);

    while (1)
      {
        ret = sem_wait(&g_sem);
        if (ret < 0)
          {
            if (errno == EINTR)
              {
                printf("EINTR\n");
              }
          }
        else
          {
            printf("sem!\n");
          }
      }

    return 0;
  }

The result is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       20836 B        64 KB     31.79%
              sram:        1540 B        16 KB      9.40%

Step 3: POSIX timer with ``SIGEV_THREAD``
-----------------------------------------

Now, we slightly modify the previous code by changing the notification type to
``SIGEV_THREAD`` and adding a notification callback:

.. code:: C

  static void sigev_callback(union sigval value)
  {
    sem_post(&g_sem);
  }

  int main(int argc, char *argv[])
  {
    struct sigevent notify;
    struct itimerspec timer;
    timer_t timerid;
    int ret;

    printf("components4 examples\n");

    notify.sigev_notify            = SIGEV_THREAD;
    notify.sigev_signo             = SIGRTMIN;
    notify.sigev_value.sival_int   = 0;
    notify.sigev_notify_function   = sigev_callback;
    notify.sigev_notify_attributes = NULL;

    timer_create(CLOCK_REALTIME, &notify, &timerid);

    timer.it_value.tv_sec     = 2;
    timer.it_value.tv_nsec    = 0;
    timer.it_interval.tv_sec  = 2;
    timer.it_interval.tv_nsec = 0;

    timer_settime(timerid, 0, &timer, NULL);

    while (1)
      {
        ret = sem_wait(&g_sem);
        if (ret < 0)
          {
            if (errno == EINTR)
              {
                printf("EINTR\n");
              }
          }
        else
          {
            printf("sem!\n");
          }
      }

    return 0;
  }

Support for ``SIGEV_THREAD`` is enabled with:

.. code:: shell

  CONFIG_SCHED_HPWORK=y
  CONFIG_SIG_EVTHREAD=y

The result is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       22072 B        64 KB     33.68%
              sram:        1764 B        16 KB     10.77%

Summary
-------

The difference between the base configuration and subsequent modifications is
shown below:

.. table:: Table 4: Signaling methods
   :class: table table-primary
   :widths: auto

   +------------------------------------+------------+----------+
   | Test case                          | FLASH      | SRAM     |
   +====================================+============+==========+
   | Base image                         | 18236 B    | 1284 B   |
   +------------------------------------+------------+----------+
   | Thread with semaphore              | +3228 B    | +64 B    |
   +------------------------------------+------------+----------+
   | POSIX timer signal                 | +2600 B    | +256 B   |
   +------------------------------------+------------+----------+
   | POSIX timer SIGEV_THREAD signal    | +3836 B    | +480 B   |
   +------------------------------------+------------+----------+

If we need to wake up the main thread at fixed intervals, using a POSIX timer
with signals can be a useful approach. On the other hand, if we opt for a thread
and semaphore mechanism, the main memory cost is associated with the kernel's
Pthread support. If our application is multi-threaded, the cost of this solution
is negligible and this'll be the lightest solution.

Application 5 - ``printf()`` and ``libm`` size
==============================================

`Application 5 Soureces <https://github.com/railab/railab_nuttx_code/blob/master/apps/mini_components5>`__

In our final tests, we evaluate ``printf()`` optimizations enabled by
``CONFIG_DEFAULT_SMALL`` and the math implementations available in NuttX.

Using ``printf()`` on small systems is generally discouraged because it
consumes precious FLASH memory. The purpose of these tests is mainly to
satisfy my curiosity ;)

All float-related tests are performed in two versions:

#. FPU disabled:

   .. code:: shell

     # CONFIG_ARCH_FPU is not set

#. FPU enabled:

   .. code:: shell

     CONFIG_ARCH_FPU=y
     CONFIG_ARMV7M_LIBM=y # hw acceleration for some libm funtions
                          # NuttX libm option only

Base image
----------

Once again, we use the standard base code. The only modification for now is that
we increase the stack size:

.. code:: shell

  CONFIG_DEFAULT_TASK_STACKSIZE=640
  CONFIG_IDLETHREAD_STACKSIZE=640

This adjustment is for cases where FPU support is enabled, which requires a
slightly increased stack size because more registers must be saved
during context switches. To simplify configuration changes, we increase the
default stack size from the beginning, but this doesn't affect the base memory
report:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       18256 B        64 KB     27.86%
              sram:        1268 B        16 KB      7.74%

``printf()`` features
---------------------

Printing floating point numbers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``CONFIG_LIBC_FLOATINGPOINT`` option enables ``float`` support for ``libc``
functions. In this case, we are only interested in printing feature, our test
code is as follows:

.. code:: C

  int main(int argc, char *argv[])
  {
    printf("components5 examples\n");

    while (1)
      {
        float f1 = 0.1f;
        float f2 = 0.2f;
        float f3 = 0.3f;

        printf("float %.2f %.2f %.2f\n", f1, f2, f3);

        sleep(1);
      }

    return 0;
  }

Configuration changes:

.. code:: shell

  CONFIG_LIBM=y                 # prerequisite, use standard NuttX libm
  CONFIG_LIBC_FLOATINGPOINT=y

The results for FPU disabled:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       23132 B        64 KB     35.30%
              sram:        1268 B        16 KB      7.74%

and with FPU enabled:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       23228 B        64 KB     35.44%
              sram:        1404 B        16 KB      8.57%

Printing long-long numbers
~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``CONFIG_LIBC_LONG_LONG`` option enables ``long long`` support for ``libc``
functions. Once again, we are only interested in printing, so the code looks
like this:

.. code::

  int main(int argc, char *argv[])
  {
    printf("components5 examples\n");

    while (1)
      {
        uint64_t x1 = 0xdeadbeefdead;
        uint64_t x2 = 0xbeafdeadbeef;

        printf("long %" PRIx64 " %" PRIx64 "\n", x1, x2);

        sleep(1);
      }

    return 0;
  }

In the configuration, we just need to add:

.. code:: shell

  CONFIG_LIBC_LONG_LONG=y

The result is:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       19164 B        64 KB     29.24%
              sram:        1268 B        16 KB      7.74%

Summary
~~~~~~~

The difference between the base configuration and subsequent modifications is
shown below:

.. table:: Table 5: Printf features
   :class: table table-primary
   :widths: auto

   +---------------------------------------------+------------+----------+
   | Test case                                   | FLASH      | SRAM     |
   +=============================================+============+==========+
   | Base image                                  | 18256 B    | 1268 B   |
   +---------------------------------------------+------------+----------+
   | CONFIG_LIBC_FLOATINGPOINT (FPU disabled)    | +4876 B    | +0 B     |
   +---------------------------------------------+------------+----------+
   | CONFIG_LIBC_FLOATINGPOINT (FPU enabled)     | +4972 B    | +136 B   |
   +---------------------------------------------+------------+----------+
   | CONFIG_LIBC_LONG_LONG                       | +908 B     | +0 B     |
   +---------------------------------------------+------------+----------+

As we can see, printing floats is very FLASH-intensive. Interestingly,
the version without FPU support is slightly less resource-heavy, though
the difference is negligible.

libm
----

Now, let's see how the math library implementations available in NuttX differ
in size. For this case, we perform some random math operations and print
the results for various options. The code looks like this:

.. code:: C

  int main(int argc, char *argv[])
  {
    printf("components5 examples\n");

    while (1)
      {
        float f1 = 0.1f;
        float f2 = 0.2f;
        float f3 = 0.3f;

        f1 = sin(f2);
        f2 = cos(f1);
        f3 = powf(f1, f2);
        f3 = fabsf(f3);
        f3 = sqrtf(f3);
        f1 = expf(f3);
        f2 = asinf(f3);
        f1 = acosf(f3);
        f1 = tanhf(f2);
        f2 = logf(f1);
        f3 = atan2f(f1, f2);

        printf("float %.2f %.2f %.2f\n", f1, f2, f3);

        sleep(1);
      }

    return 0;
  }

We consider two ``libm`` solutions here:

#. The custom ``libm`` implementation from NuttX, enabled with the ``CONFIG_LIBM=y`` option.

#. The library implementation from Newlib, enabled with the ``CONFIG_LIBM_NEWLIB=y`` option.

We ignore two other possible options:

#. ``CONFIG_LIBM_TOOLCHAIN`` - because on my host, this option is the same as using Newlib.

#. ``CONFIG_LIBM_OPENLIBM`` - because it doesn't work with the CMake build at the time of writing.

For all cases, printing floats is enabled with:

.. code:: shell

  CONFIG_LIBC_FLOATINGPOINT=y

The results for all cases are as follows:

#. ``libm`` from NuttX with FPU disabled:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       27252 B        64 KB     41.58%
              sram:        1268 B        16 KB      7.74%

#. ``libm`` from NuttX with FPU enabled:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       25460 B        64 KB     38.85%
              sram:        1404 B        16 KB      8.57%

#. ``libm`` from Newlib with FPU disabled:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       37960 B        64 KB     57.92%
              sram:        1268 B        16 KB      7.74%

#. ``libm`` from Newlib with FPU enabled:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       35712 B        64 KB     54.49%
              sram:        1404 B        16 KB      8.57%

Summary
~~~~~~~

The results are summarized in the table below:

.. table:: Table 6: Libm comparison
   :class: table table-primary
   :widths: auto

   +---------------------------------------------+------------+----------+
   | Test case                                   | FLASH      | SRAM     |
   +=============================================+============+==========+
   | Math library from NuttX (FPU disabled)      | 27316 B    | 1268 B   |
   +---------------------------------------------+------------+----------+
   | Math library from NuttX (FPU enabled)       | 25460 B    | 1404 B   |
   +---------------------------------------------+------------+----------+
   | Math library from Newlib (FPU disabled)     | 38024 B    | 1268 B   |
   +---------------------------------------------+------------+----------+
   | Math library from Newlib (FPU enabled)      | 35768 B    | 1404 B   |
   +---------------------------------------------+------------+----------+

We can see that the math library from NuttX is a much lighter solution.
Unlike in the previous test, here we observe a positive impact on resource
usage when the FPU is enabled.

Floating-point math and fixed-point math
----------------------------------------

The last test for today is a comparison of math operations for ``libm`` and
fixed-point math library in NuttX (``include/fixedmath.h``).

For the fixed-point math test, we keep the FPU disabled with:

.. code:: shell

  # CONFIG_ARCH_FPU is not set

The test code is:

.. code:: C

  int main(int argc, char *argv[])
  {
    printf("components5 examples\n");

    while (1)
      {
        volatile b16_t b1 = ftob16(0.1f);
        volatile b16_t b2 = ftob16(0.2f);
        volatile b16_t b3 = ftob16(0.3f);

        b3 = b16sin(b1);
        b2 = b16cos(b1);
        b1 = b16atan2(b3, b2);
        b2 = b16sqr(b1);
        b3 = b16mulb16(b1, b2);
        b1 = b16divb16(b3, b2);

        sleep(1);
      }

    return 0;
  }
  
This gives us:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       19604 B        64 KB     29.91%
              sram:        1268 B        16 KB      7.74%

For floating point operations we do the same math operations:

.. code:: C

  int main(int argc, char *argv[])
  {
    printf("components5 examples\n");

    while (1)
      {
        volatile float fl1 = 0.1f;
        volatile float fl2 = 0.2f;
        volatile float fl3 = 0.3f;
        fl3 = sinf(fl1);
        fl2 = cosf(fl1);
        fl1 = atan2f(fl3, fl2);
        fl2 = sqrtf(fl1);
        fl3 = fl1 * fl2;
        fl1 = fl3 / fl2;

        sleep(1);
      }

    return 0;
  }

With FPU disabled we get:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       22984 B        64 KB     35.07%
              sram:        1268 B        16 KB      7.74%

And with FPU enabled:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       21340 B        64 KB     32.56%
              sram:        1404 B        16 KB      8.57%


The results for floating-point math look suspiciously large. If we examine the
generated binary, the reason becomes clear: the internal calculations for
``atan2f()`` use double-precision arithmetic, which significantly increases
FLASH usage.

We can address this by applying a simple patch to the NuttX sources,
sacrificing the accuracy of calculations (any other side effects of this
change haven't been tested):

.. code:: shell

  diff --git a/libs/libm/libm/lib_asinf.c b/libs/libm/libm/lib_asinf.c
  index 4cc1ed646e..b9b183aa71 100644
  --- a/libs/libm/libm/lib_asinf.c
  +++ b/libs/libm/libm/lib_asinf.c
  @@ -42,7 +42,7 @@

   static float asinf_aux(float x)
   {
  -  double y;
  +  float y;
     float y_sin;
     float y_cos;

  @@ -52,7 +52,7 @@ static float asinf_aux(float x)
     while (fabsf(y_sin - x) > FLT_EPSILON)
       {
         y_cos = cosf(y);
  -      y -= ((double)y_sin - (double)x) / (double)y_cos;
  +      y -= ((float)y_sin - (float)x) / (float)y_cos;
         y_sin = sinf(y);
       }

New results with FPU disabled:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       21560 B        64 KB     32.90%
              sram:        1268 B        16 KB      7.74%

And new result with FPU enabled:

.. code:: shell

  Memory region         Used Size  Region Size  %age Used
             flash:       19884 B        64 KB     30.34%
              sram:        1404 B        16 KB      8.57%

Summary
~~~~~~~

The results are summarized in the table below:

.. table:: Table 7: Fixed-point math vs. floating-point math
   :class: table table-primary
   :widths: auto

   +-----------------------------------------------+------------+----------+
   | Test case                                     | FLASH      | SRAM     |
   +===============================================+============+==========+
   | Fixed-point NuttX library (FPU disabled)      | 19604 B    | 1268 B   |
   +-----------------------------------------------+------------+----------+
   | Float NuttX libm (FPU disabled)               | 22984 B    | 1268 B   |
   +-----------------------------------------------+------------+----------+
   | Float NuttX libm (FPU enabled)                | 21340 B    | 1404 B   |
   +-----------------------------------------------+------------+----------+
   | Float NuttX libm (FPU disabled with patch)    | 21560 B    | 1268 B   |
   +-----------------------------------------------+------------+----------+
   | Float NuttX libm (FPU enabled with patch)     | 19884 B    | 1404 B   |
   +-----------------------------------------------+------------+----------+

We can clearly see that if our MCU doesn't support an FPU, the better solution
in terms of resource usage is to use fixed-point math. Unfortunately,
the number of available functions for the fixed-point library in NuttX is
limited.

If our target has FPU support, it's worth examining the compiled image to
ensure we aren't using any double-specific functions, as they can be
expensive in terms of FLASH usage. Unwanted double-precision calculations
are a common problem when dealing with ``float`` type, but usually easy to
fix.

Conclusions
===========

The term **"small embedded system"** is broad and can vary in meaning depending on
one's background and experience. In the context of NuttX, a *"small system"*
doesn't refer to a specific size of available resources. Instead, it means a
system where everything unnecessary is turned off, and all OS settings are
tuned to their minimum values.

Based on what we've gathered so far, we have a good idea of how low resource
consumption can go for small applications in NuttX. The examples used in
this analysis are minimal, so most of the reported results show the overhead
of the RTOS itself. With this in mind, we can estimate which small targets
might still work well with NuttX.

It should be possible to run small multi-threaded applications with as little as
**32KB of FLASH** and **6KB of SRAM**, which is a common resource range for many
small microcontrollers. For very basic, single-task applications, it should be
doable to fit within **20KB of FLASH** and **4KB of SRAM**. However, dropping
below this range would likely require breaking some POSIX abstractions or using
other "dirty hacks."

As we've seen, precise configuration of NuttX is a critical step in building
small applications. Thankfully, the ``CONFIG_DEFAULT_SMALL`` option simplifies
this process by handling most of the initial adjustments, leaving developers to
fine-tune the configuration to suit their specific application needs. The most
optimal approach in the case of small systems design is to use the OS features
that are also used to implement core OS components. This way, the code in the
image can be efficiently reused.

It would be great to automate the generation of all these memory reports, present
them in a more readable way, and build a tool to compare results. I might look
into this in the future—it would make tracking memory usage across different
versions of NuttX and different compilers much easier.

At this point, we've got a decent understanding of NuttX's memory requirements
for small systems. Now is a good time to put this knowledge to practical use.
The plan for the future is to design and develop small NuttX applications
that actually do something useful. Along the way, we'll test other useful
OS components that weren't covered in this analysis.
