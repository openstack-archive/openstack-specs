=========================
 Eventlet Best Practices
=========================

No blueprint, this is intended as a reference document.

Eventlet is used in many of the OpenStack projects as the default concurrency
model, and there are some things we've learned about it over the years that
currently exist only as tribal knowledge.  This is an attempt to codify those
in a central location for everyone's benefit.

It is worth noting that while there has been a push from some members of the
community to move away from eventlet entirely, there is currently no approved
plan to do so.  Even if there were, it will likely take a long time to
implement, so eventlet will be something we have to care about for at least
the short and medium term.

Problem description
===================

In some ways eventlet behaves much differently from other concurrency models
and can even change the behavior of the Python standard library.  This means
that scenarios exist where a bad interaction between eventlet and some other
code, often code that is not eventlet-aware, can cause problems.  We need some
best practices that will minimize the potential for these issues to occur.

Proposed change
===============

Guidelines for using eventlet:

Monkey Patching
---------------

* When using eventlet.monkey_patch, do it first or not at all.  In practice,
  this means monkey patching in a top-level __init__.py which is guaranteed
  to be run before any other project code.  As an example, Nova monkey patches
  in nova/cmd/__init__.py and nova/tests/unit/__init__.py so that in both the
  runtime and test scenarios the monkey patching happens before any Nova code
  executes.

  The reasoning behind this is that unpatched stdlib modules may not play
  nicely with eventlet monkey patched ones.  For example, if thread A is
  started, the application monkey patches, then starts thread B, now you've
  mixed native threads and green threads and the results are undefined but
  most likely bad.

  It is not practical to expect developers to recognize all such
  possible race conditions during development or review, and in fact it is
  impossible because the race condition could be introduced by code we
  consume from another library.  Because of this, it is safest to
  simply eliminate the races by monkey patching before any other code is run.

* Monkey patching should also be done in a way that allows services to run
  without it, such as when an API service runs under Apache.  This is the
  reason for Nova not simply monkey patching in nova/__init__.py.

  Another example is Keystone, which recommends running under Apache but also
  supports eventlet.  They have a separate eventlet binary 'keystone-all' which
  handles monkey patching before running any other code.  Note that
  `eventlet is deprecated`_ in Keystone as of the Kilo cycle.

.. _`eventlet is deprecated`: http://lists.openstack.org/pipermail/openstack-dev/2015-February/057359.html

* Monkey patching with thread=False is likely to cause problems.  This is done
  conditionally in many services due to `problems running under a debugger`_
  with the threading module monkey patched.  Unfortunately, even simple
  concurrency scenarios can result in deadlocks with this sort of setup.  For
  example, the following code provided by Josh Harlow will cause hangs::

        import eventlet

        eventlet.monkey_patch(os=False, thread=False)


        import threading

        import time


        thingy_lock = threading.Lock()


        def do_it():
            with thingy_lock:
                time.sleep(1)


        threads = []
        for i in range(0, 5):
            threads.append(eventlet.spawn(do_it))
        while threads:
            t = threads.pop()
            t.wait()

  It is unclear at this time whether there is a way to enable debuggers and
  also have a sane monkey patched environment.  The `eventlet backdoor`_ was
  mentioned as a possible alternative.

.. _`problems running under a debugger`: http://lists.openstack.org/pipermail/openstack-dev/2012-August/000693.html
.. _`eventlet backdoor`: http://lists.openstack.org/pipermail/openstack-dev/2012-August/000873.html

* Monkey patching can cause problems running flake8 with multiple workers.
  If it does, the monkey patching can be made conditional based on an
  environment variable that can be set during flake8 test runs.  This should
  not be a problem as monkey patching is not needed for flake8.

  For example::

      import os

      if not os.environ.get('DISABLE_EVENTLET_PATCHING'):
          import eventlet
          eventlet.monkey_patch()

  Even though os is being imported before monkey patching, this should be safe
  as long as no other code is run before monkey patching occurs.

Greenthread-aware Modules
-------------------------

* There is a greenthread-aware subprocess module in eventlet, but it does
  *not* get patched in by eventlet.monkey_patch.  Code that has interactions
  between green threads and the subprocess module must be sure to use the
  green subprocess module explicitly.  A simpler alternative is to use
  processutils from oslo.concurrency, which selects the appropriate module
  depending on the status of eventlet's monkey patching.

Database Drivers
----------------

* Eventlet can cause deadlocks_ in some Python database drivers.  The current
  plan is to move our recommended and default driver_ to something that is more
  eventlet-friendly.

.. _deadlocks: https://wiki.openstack.org/wiki/OpenStack_and_SQLAlchemy#MySQLdb_.2B_eventlet_.3D_sad
.. _driver: https://wiki.openstack.org/wiki/PyMySQL_evaluation#MySQL_DB_Drivers_Comparison

Tools for Ensuring Monkey Patch Sanity
--------------------------------------

* The oslo.utils project has an eventletutils_ module that can help ensure
  proper monkey patching for code that knows what it needs patched.  This
  could, for example, be used to raise a warning when a service is run under
  a debugger without threading patched.  At least that way the user will have
  a clue what is wrong if deadlocks occur.

.. _eventletutils: http://docs.openstack.org/developer/oslo.utils/api/eventletutils.html

Alternatives
------------

* Continue to have each project implement eventlet in its own way.  This is
  undesirable because it will result in projects hitting bugs that may have
  been solved in another project.

Implementation
==============

Assignee(s)
-----------

Primary assignee: bnemec

Additional contributors: harlowja, ihrachyshka

Work Items
----------

* Audit the use of eventlet in OpenStack projects and make any changes
  necessary to abide by these guidelines.

* Follow up with the eventlet team on whether the green subprocess module
  not being included in monkey patching is intentional.


Dependencies
============

None

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Kilo
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0 Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode
