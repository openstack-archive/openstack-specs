==========================================
 Chronicles of a distributed lock manager
==========================================

No blueprint, this is intended as a reference/consensus document.

The various OpenStack projects have an ongoing requirement to perform
some set of actions in an atomic manner performed by some distributed set of
applications on some set of distributed resources **without** having those
resources end up in some corrupted state due those actions being performed on
them without the traditional concept of `locking`_.

A `DLM`_ is one such concept/solution that can help (but not entirely
solve) these types of common resource manipulation patterns in distributed
systems. This specification will be an attempt at defining the problem
space, understanding what each project *currently* has done in regards of
creating its own `DLM`_-like entity and how we can make the situation better
by coming to consensus on a common solution that we can benefit from to
make everyone's lives (developers, operators and users of OpenStack
projects) that much better. Such a consensus being built will also
influence the future functionality and capabilities of OpenStack at large
so we need to be **especially** careful, thoughtful, and explicit here.

.. _DLM: https://en.wikipedia.org/wiki/Distributed_lock_manager
.. _locking: https://en.wikipedia.org/wiki/Lock_%28computer_science%29

Problem description
===================

Building distributed systems is **hard**. It is especially hard when the
distributed system (and the applications ``[X, Y, Z...]`` that compose the
parts of that system) manipulate mutable resources without the ability to do
so in a conflict-free, highly available, and
scalable manner (for example, application ``X`` on machine ``1`` resizes
volume ``A``, while application ``Y`` on machine ``2`` is writing files to
volume ``A``). Typically in local applications (running on a single
machine) these types of conflicts are avoided by using primitives provided
by the operating system (`pthreads`_ for example, or filesystem locks, or
other similar `CAS`_ like operations provided by the `processor instruction`_
set). In distributed systems these types of solutions do **not** work, so
alternatives have to either be invented or provided by some
other service (for example one of the many academia has created, such
as `raft`_ and/or other `paxos`_ variants, or services created
from these papers/concepts such as `zookeeper`_ or `chubby`_ or one of the
many `raft implementations`_ or the redis `redlock`_ algorithm). Sadly in
OpenStack this has meant that there are now multiple implementations/inventions
of such concepts (most using some variation of database locking), using
different techniques to achieve the defined goal (conflict-free, highly
available, and scalable manipulation of resources). To make things worse
some projects still desire to have this concept and have not reached the
point where it is needed (or they have reached this point but have been
unable to achieve consensus around an implementation and/or
direction). Overall this diversity, while nice for inventors and people
that like to explore these concepts does **not** appear to be the best
solution we can provide to operators, developers inside the
community, deployers and other users of the now (and every expanding) diverse
set of `OpenStack projects`_.

.. _redlock: http://redis.io/topics/distlock
.. _pthreads: http://man7.org/linux/man-pages/man7/pthreads.7.html
.. _CAS: https://en.wikipedia.org/wiki/Compare-and-swap
.. _processor instruction: http://www.felixcloutier.com/x86/CMPXCHG.html
.. _paxos: https://en.wikipedia.org/wiki/Paxos_%28computer_science%29
.. _raft: http://raftconsensus.github.io/
.. _zookeeper: https://en.wikipedia.org/wiki/Apache_ZooKeeper
.. _chubby: http://research.google.com/archive/chubby.html
.. _raft implementations: http://raftconsensus.github.io/#implementations
.. _OpenStack projects: http://git.openstack.org/cgit/openstack/\
                        governance/tree/reference/projects.yaml

What has been created
---------------------

To show the current diversity let's dive slightly into what *some* of the
projects have created and/or used to resolve the problems mentioned above.

Cinder
******

**Problem:**

Avoid multiple entities from manipulating the same volume resource(s)
at the same time while still being scalable and highly available.

**Solution:**

Currently is limited to file locks and basic volume state transitions. Has
limited scalability and reliability of cinder under failure/load; has been
worked on for a while to attempt to create a solution that will fix some of
these fundamental issues.

**Notes:**

- For further reading/details these links can/may offer more insight.

  - https://review.openstack.org/#/c/149894/
  - https://review.openstack.org/#/c/202615/
  - https://etherpad.openstack.org/p/mitaka-cinder-volmgr-locks
  - https://etherpad.openstack.org/p/mitaka-cinder-cvol-aa
  - (and more)

Ironic
******

**Problem:**

Avoid multiple conductors from manipulating the same bare-metal
instances and/or nodes at the same time while still being scalable and
highly available.

Other required/implemented functionality:

* Track what services are running, supporting what drivers, and rebalance
  work when service state changes (service discovery and rebalancing).
* Sync state of temporary agents instead of polling or heartbeats.

**Solution:**

Partition resources onto a hash-ring to allow for ownership to be scaled
out among many conductors as needed. To avoid entities in that hash-ring
from manipulating the same resource/node that they both may co-own a database
lock is used to ensure single ownership. Actions taken on nodes are performed
after the lock (shared or exclusive) has been obtained (a `state machine`_
built using `automaton`_ also helps ensure only valid transitions
are performed).

**Notes:**

- Has logic for shared and exclusive locks and provisions for upgrading
  a shared lock to an exclusive lock as needed (only one exclusive lock
  on a given row/key may exist at the same time).
- Reclaim/take over lock mechanism via periodic heartbeats into the
  database (reclaims is apparently a manual and clunky process).

**Code/doc references:**

- Some of the current issues listed at `pluggable-locking`_.

  - `Etcd`_ proposed @ `179965`_ I believe this further validates the view
    that we need a consensus on a uniform solution around DLM (vs continually
    having projects implement whatever suites there fancy/flavor of the week).

- https://github.com/openstack/ironic/blob/master/ironic/conductor/task_manager.py#L20
- https://github.com/openstack/ironic/blob/master/ironic/conductor/task_manager.py#L222

.. _state machine: http://docs.openstack.org/developer/ironic/dev/states.html
.. _automaton: http://docs.openstack.org/developer/automaton/
.. _179965: https://review.openstack.org/#/c/179965
.. _Etcd: https://github.com/coreos/etcd
.. _pluggable-locking: https://blueprints.launchpad.net/ironic/+spec/pluggable-locking

Heat
****

**Problem:**

Multiple engines working on the same stack (or nested stack of). The
ongoing convergence rework may change this state of the world (so in the
future the problem space might be slightly different, but the concept
of requiring locks on resources will still exist).

**Solution:**

Lock a stack using a database lock and disallow other engines
from working on that same stack (or stack inside of it if nested),
using expiry/staleness allow other engines to claim potentially
lost lock after period of time.

**Notes:**

- Liveness of stack lock not easy to determine? For example is an engine
  just taking a long time working on a stack, has the engine had a network
  partition from the database but is still operational, or has the engine
  really died?

  - To resolve this a combination of an ``oslo.messaging`` ping used to
    determine when a lock may be dead (or the owner of it is dead), if an
    engine is non-responsive to pings/pongs after period of time (and its
    associated database entry has expired) then stealing is allowed to occur.

- Lacks *simple* introspection capabilities? For example it is necessary
  to examine the database or log files to determine who is trying to acquire
  the lock, how long they have waited and so on.

- Lock releasing may fail (which is highly undesirable, *IMHO* it should
  **never** be possible to fail releasing a lock); implementation does not
  automatically release locks on application crash/disconnect/other but relies
  on ping/pongs and database updating (each operation in this
  complex 'stealing dance' may fail or be problematic, and therefore is not
  especially simple).

**Code/doc references:**

- http://docs.openstack.org/developer/heat/_modules/heat/engine/stack_lock.html
- https://github.com/openstack/heat/blob/master/heat/engine/resource.py#L1307

Ceilometer and Sahara
*********************

**Problem:**

Distributing tasks across central agents.

**Solution:**

Token ring based on `tooz`_.

**Notes:**

Your project here
*****************

Solution analysis
=================

The proposed change would be to choose one of the following:

- Select a distributed lock manager (one that is opensource) and integrate
  it *deeply* into openstack, work with the community that owns it to develop
  and issues (or fix any found bugs) and use it for lock management
  functionality and service discovery...
- Select a API (likely `tooz`_) that will be backed by capable
  distributed lock manager(s) and integrate it *deeply* into openstack and
  use it for lock management functionality and service discovery...

* `zookeeper`_ (`community respected
  analysis <https://aphyr.com/posts/291-call-me-maybe-zookeeper>`__)
* `consul`_ (`community respected
  analysis <https://aphyr.com/posts/316-call-me-maybe-etcd-and-consul>`__)
* `etc.d`_ (`community respected
  analysis <https://aphyr.com/posts/316-call-me-maybe-etcd-and-consul>`__)

Zookeeper
---------

Summary:

Age: around 8 years

* Changelog was created in svn repository on aug 27, 2007.

License: Apache License 2.0

Approximate community size:

Features (overview):

- `Zab`_ based (paxos variant)
- Reliable filesystem like-storage (see `zk data model`_)
- Mature (and widely used) python client (via `kazoo`_)
- Mature shell/REPL interface (via `zkshell`_)
- Ephemeral nodes (filesystem entries that are tied to presence
  of their creator)
- Self-cleaning trees (implemented in 3.5.0 via
  https://issues.apache.org/jira/browse/ZOOKEEPER-2163)
- Dynamic reconfiguration (making upgrades/membership changes that
  much easier to get right)
  - https://zookeeper.apache.org/doc/trunk/zookeeperReconfig.html

Operability:

- Rolling restarts < 3.5.0 (to allow for upgrades to happen)
- Starting >= 3.5.0, 'rolling restarts' are no longer needed (see
  mention of dynamic reconfiguration above)
- Java stack experience required

Language written in: java

.. _kazoo: http://kazoo.readthedocs.org/
.. _zkshell: https://pypi.python.org/pypi/zk_shell/
.. _zk data model: http://zookeeper.apache.org/doc/\
                   trunk/zookeeperProgrammers.html#ch_zkDataModel
.. _Zab: https://web.stanford.edu/class/cs347/reading/zab.pdf

Packaged: yes (at least on ubuntu and fedora)

* http://packages.ubuntu.com/trusty/java/zookeeperd
* https://apps.fedoraproject.org/packages/zookeeper

Consul
------

Summary:

Age: around 1.5 years

* Repository changelog denotes added in april 2014.

License: Mozilla Public License, version 2.0

Approximate community size:

Features (overview):

- Raft based
- DNS interface
- HTTP interface
- Reliable K/V storage
- Suited for multi-datacenter usage
- Python client (via `python-consul`_)

.. _python-consul: https://pypi.python.org/pypi/python-consul
.. _consul: https://www.consul.io/

Operability:

* Go stack experience required

Language written in: go

Packaged: somewhat (at least on ubuntu and fedora)

* Ppa at https://launchpad.net/~bcandrea/+archive/ubuntu/consul
* https://admin.fedoraproject.org/pkgdb/package/consul/ (?)

Etc.d
-----

Summary:

Age: Around 1.09 years old

License: Apache License 2.0

Approximate community size:

Features (overview):

Language written in: go

Operability:

* Go stack experience required

Packaged: ?

Proposed change
===============

Place all functionality behind `tooz`_ (as much as possible) and let the
operator choose which implementation to use. Do note that functionality that
is not possible in all backends (for example consul provides a `DNS`_ interface
that complements its HTTP REST interface) will not be able to be exposed
through a `tooz`_ API, so this may limit the developer using `tooz`_ to
implement some feature/s).

Compliance: further details about what each `tooz`_ driver must
conform to (as in regard to how it operates, what functionality it must support
and under what consistency, availability, and partition tolerance scheme
it must operate under) will be detailed at: `240645`_

It is expected as the result of `240645`_ that
certain existing `tooz`_ drivers will be deprecated and eventually removed
after a given number of cycles (due to there inherent inability to meet the
policy constraints created by that specification) so that the quality
and consistency of there operating policy can be guaranteed (this guarantee
reduces the divergence in implementations that makes plugins that much
harder to diagnosis, debug, and validate).

.. Note::

    Do note that the `tooz`_ alternative which needs to be understood
    is that `tooz`_ is a tiny layer around solutions mentioned above, which
    is an admirable goal (I guess I can say this since I helped make that
    library) but it does favor pluggability over picking one solution and
    making it better. This is obviously a trade-off that must IMHO **not** be
    ignored (since ``X`` solutions mean that it becomes that much harder to
    diagnose and fix upstream issues because ``X - Y`` solutions may not have
    the issue in the first place); TLDR: pluggability comes at a cost.

.. _DNS: http://www.consul.io/docs/agent/dns.html
.. _tooz: http://docs.openstack.org/developer/tooz/
.. _240645: https://review.openstack.org/#/c/240645/

Implementation
==============

Assignee(s)
-----------

- All the reviewers, code creators, PTL(s) of OpenStack?

Work Items
----------

Dependencies
============

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Mitaka
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0 Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode
