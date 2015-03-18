========================================================
 Enabling Python 3 for Integration and Functional Tests
========================================================

The 2.x series of the C Python interpreter on which OpenStack releases
through Kilo are built is reaching the end of its extended support
period, defined by the upstream developers. This spec describes
motivation for porting fully to Python 3 and some of the work we will
need to enable testing applications as they move to Python 3.

Problem description
===================

There are a lot of small motivations for moving to Python 3, including
better unicode support and new features in the language and standard
library. The primary motivation, however, is that Python 2 is reaching
its end-of-life for support from its developers.

Just as we expect our users to update to new versions of OpenStack in
order to continue to receive support, the python-dev team expects
users of the language to update to reasonably modern and supported
versions of the interpreter in order to receive bug and security
fixes. When Python 3 was introduced, the support period for Python 2
was extended beyond the normal length of time to allow projects plenty
of time to migrate, and to allow the python-dev team to receive
feedback to make changes to the language so that migration is
easier. That period is coming to an end, and we need to consider
migration seriously.

  "Python 3.0 was released in 2008. The final 2.x version 2.7 release
  came out in mid-2010, with a statement of extended support for this
  end-of-life release. The 2.x branch will see no new major releases
  after that. 3.x is under active development and has already seen
  over five years of stable releases, including version 3.3 in 2012
  and 3.4 in 2014. This means that all recent standard library
  improvements, for example, are only available by default in Python
  3.x." -- Python2orPython3_

That said, we cannot expect all of OpenStack to be ported at one
time. It's likely that we could not port everything in a single
release cycle, given the other work going on. So we need a way to
stage the porting work so that projects can port when they are ready,
without having to wait for any other projects to finish their ports.

Proposed change
===============

Our services communicate through REST APIs and the message bus. This
means they are decoupled enough that we can port them one at a time,
if our tools support running some services on Python 2 and some on
Python 3. Our unit test tool, tox, supports multiple Python versions
already, and in fact most of our library projects are testing under
Python 2.6, 2.7, and 3.4 today. Our integration tests, however, do not
yet support multiple Python versions, so that's the next step to take.

General Strategy
----------------

#. Update devstack to install apps with the "right" version of the
   interpreter.

   * Use the version declared to be supported by the project through
     its trove classifiers.
   * Allowing apps to be installed with the right version of the
     interpreter independently of other apps means we can port one
     app at a time.

#. Port each application to 3.4, but support both 2.7 and 3.4.

   * Set up an appropriate devstack-gate job using Python 3 as
     non-voting for projects when they start to port.

   * Make incremental changes to the applications until the non-voting
     job passes reliably, then update it to make it a voting job.

   * Technically there is no need to run integration tests for an
     application under both versions, since they only need to be
     deployed under one version at a time. However, different
     packagers and deployers may want to choose to wait to move to
     Python 3 and so we can continue to run the tests under both
     versions.

.. note::

   Even after all applications are on 3.x, we need to maintain some
   python 2.7 support for client libraries and the Oslo libraries they
   use. We should consider the deprecation policy of Python 2 for the
   client libraries independently of porting the applications to 3.

Which version of Python to use?
-------------------------------

We have discussed this before, and it continues to be a moving
target. Version 3.4 seems to be our best goal for now.

- 3.0 - 3.2 are no longer actively supported
- 3.3 is not available on all distros
- **3.4 is (or soon will be) available on all distros**
- 3.5 is in beta and so is not ready for us to use, yet

Functional Tests for Libraries
------------------------------

Besides the functional and integration tests for applications, we also
have functional tests for libraries. I propose that we configure the
test jobs to run those only under Python 3, to avoid duplication and
expose porting issues that would have an impact on applications as
early as possible.

Alternatives
------------

Stay with C Python 2
~~~~~~~~~~~~~~~~~~~~

Commercial support is likely to be available from distros for longer
than it is available upstream, but even that will stop at some point.

Use PyPy or Another Implementation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some applications may benefit from PyPy's JIT compiler. It currently
supports 2.7.8 and 3.2.5, which means our Python 2 code would probably
run but code designed for Python 3.4 will not. I'm not aware of any
large deployments using PyPy to run services, so I'm not sure this is
really a problem. Given the expected long time frame for porting to
Python 3, it is likely that PyPy will be able to catch up to the
language level needed to run OpenStack by the time we are fully moved
to Python 3.

Wait for Python 3.5
~~~~~~~~~~~~~~~~~~~

Moving from 3.4 to 3.5 should require much less work than moving from
2.7 to 3.4. We can therefore start now, and monitor adoption of 3.5 by
distributions to decide whether to ultimately use 3.4 or a later
version.

Use Separate Virtualenvs in devstack
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We have discussed installing applications into virtualenvs a couple of
times. Doing that is orthogonal to these proposed changes, since we
would still need to use the correct version of Python within the
virtualenv.

Functional tests for libraries on 2 and 3
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We could run parallel test jobs configured to run the functional tests
for libraries under both Python 2 and 3. This would largely duplicate
effort, though it might uncover some inconsistent handling of bytes
vs. strings. We shouldn't start out trying to do this, but if we do
uncover problems we can add more test jobs.

Implementation
==============

Assignee(s)
-----------

Primary assignee: Doug Hellmann

Work Items
----------

1. Update devstack to install pip for both Python 2 and Python 3.

2. Update devstack to look at the supported Python versions for a
   project, and choose the correct copy of pip to install it and its
   dependencies.

   This may be as simple as::

       python setup.py --classifiers | grep 'Language' | cut -f5 -d: | grep '\.'

3. When installing libraries from source using the ``LIBS_FROM_GIT``
   feature of devstack, ensure that the libraries are installed for
   both Python 2 and Python 3.

4. Begin porting applications to Python 3.

   * Unit tests can be run under Python 3 for applications just as
     they are for libraries, by enabling the appropriate job. Having
     the unit tests working with Python 3 is a good first step, before
     enabling the integration tests.
   * Integration tests can be run by submitting a patch updating the
     trove classifier.
   * Some projects will have dependencies blocking them from moving to
     Python 3 at first, and those should be tracked separately from
     this proposal.

Some functions in Oslo libraries have been identified as having
incompatibilities with Python 3. As these cases are reported, we will
need to decide, on a case-by-case basis whether it is feasible to
create versions of those functions that work for both Python 2 and 3,
or if we will need to create some new APIs for use under Python 3 (see
``oslo_utils.encodeutils.safe_decode``,
``oslo_utils.strutils.mask_password``, and
``oslo_concurrency.processutils.execute`` as examples).

References
==========

- A proof-of-concept patch to devstack: https://review.openstack.org/181165

- Our notes about the state of Python 3 support:
  https://wiki.openstack.org/wiki/Python3

- Advice from the python-dev community about choosing a Python
  version: Python2orPython3_

- Summit discussions

  - `Havana <https://etherpad.openstack.org/p/havana-python3>`__
  - `Icehouse <https://etherpad.openstack.org/p/IcehousePypyPy3>`__
  - `Juno <https://etherpad.openstack.org/p/juno-cross-project-future-of-python>`__

- Project-specific specs related to Python 3

  - `Heat <http://specs.openstack.org/openstack/heat-specs/specs/liberty/heat-python34-support.html>`__
  - `Keystone <https://review.openstack.org/#/c/177380/>`__
  - `Neutron <https://review.openstack.org/#/c/172962/>`__
  - `Nova <https://review.openstack.org/#/c/176868>`__

.. _Python2orPython3: https://wiki.python.org/moin/Python2orPython3

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Liberty
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0 Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode
