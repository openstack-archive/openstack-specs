=========================
 Requirements management
=========================

Our current requirements management policy is causing significant gate
downtime, developer issues and packaging headaches. We can do better.

Problem description
===================

There are a number of interacting things we do today which are causing issues.

We run our tests with unpinned dependencies. This means that any patch being
tested actually tests two things: the change in the patch, and all new
releases of dependencies (within the ranges relevant to the branch in
question). With nearly 300 (transitive) libraries in use, an incompatibility
rate of only one per library per year can break us daily. We are suffering
regular firedrills, and we've burnt out many gate fixers so far.

We require that projects have their install_requires exactly match
our top level package specifiers that we use in gate jobs. This leads to
releases of stable branches that have much narrower dependency specifiers than
may work in practice. This becomes a problem when one package of a set needs
to upgrade a given dependency post-release to fix a bug: the new package can
be outside the set of versions dictated by the union of specifiers across all
our packages that use it, which causes a cascade where new releases are
required across the entire set to permit it to be used.

We override project local install_requires during testing, which means that
our co-installability check is often returning a false positive: our actual
install_requires may be incompatible but the gate won't report on this.

Additionally we have some constraints on solutions:

1. Provide “some” expression to downstream packagers/users

2. Configure test jobs (unit, integration, & functional) (devstack &
   non-devstack)

3. Encourage convergence/maintain co-installability

4. Not be fiction

5. pip install python-novaclient works

6. Stop breaking all the time (esp. stable)

7. Don’t make library release management painful

Finally there are plenty of things that could be done that aren't addressed in
this specification: it's the minimal self consistent set of improvements to
address the ongoing firedrills we currently suffer.

Proposed change
===============

tl;dr: Use exact pins for testing. Use open ended specifiers in project
install_requires.

Globally we need to maintain global-requirements as we do today. This remains
our policy control for which libraries are acceptable in OpenStack projects.
As we start using extras, we'll need to track extras[1] in global-requirements
as well. We want to preserve an axiom: that projects have wider-or-equal
requirements than the coordinated release which has wider-or-equal
requirements to the test pinned list.

We'll add a new pip freeze file to openstack/requirements, called
`upper-constraints.txt`.  This will contain a pinned list of the entire set of
transitive requirements for that branch of OpenStack (defined by the projects
within the projects.txt file in openstack/requirements). All CI jobs will use
this to constrain the versions of Python projects that can be tested with.
Changes to that file will be tested by running a subset of the same jobs that
would consume it, as well as a policy checker that checks it is compatible
with `global-requirements.txt`. Changes to either `global-requirements.txt` or
`upper-constraints.txt` will have to be compatible with each other.

We'll tighten up the policy checks on projects to require that there be no
dependencies outside of those listed in global-requirements. This is needed to
allow centralised calculations about potential upgrades and co-installability
calculations. We'll change the check from 'identical line to global-requires'
to be 'compatible with both global-requires and upper-constraints'. The
existing requirements sync job will continue to propose converged requirements
for projects that have opted into converged dependency management.

We'll create a periodic job that takes global-requirements, expands it out
transitively, and then proposes a merge to global-requirements bringing in
any new releases (and any new or removed transitive-only dependencies that that
implies) as a patch to upper-constraints.

Releases are a multi week process in OpenStack as different servers fork and
setup their branches one at a time. During that period we'll gate any
requirements changes on both master and any branched projects, branching
openstack/requirements last when we're finally ready to decouple the release
from master. This is aimed at the changes involved in using new releases of
oslo etc.

Lastly, the pip resolver work will increase the accuracy of our constraints
pinning, but this spec is not dependent on that.

Alternatives
------------

The null option will continue to burn people out.

No other options have been proposed.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  lifeless

Work Items
----------

- Open up install-requires in master and stable/kilo. [lifeless]

- Generate pip glue to honour upper-constraints.txt. There are a few different
  ways this might be cut, lifeless will figure a working low hanging fruit
  implementation with upstream and/or infra. [lifeless]

- Create an initial upper-constraints.txt for master and stable/kilo.
  [lifeless]

- Change g-r self test to ensure consistency between g-r and
  upper-constraints.txt. [lifeless]

- Create jobs to prevent project local install_requires being incompatible
  with global-requirements. [fungi/lifeless]

- Teach devstack how to honour a constraints file. [lifeless]

- Teach unittests / tox how to honour a constraints file. [lifeless]

- Generate zuul glue to mangle constraints files when a project from within
  the constraints is part of the queue being assessed. [fungi/lifeless]

- Create jobs that use constraints files for everything. experimental for
  local project contexts, non-voting when triggered from g-r. [fungi]

- Create script to determine transitive dependencies of global-requirements
  and propose upgrades to upper-constraints.txt. [lifeless]

- Turn that script into a periodic job. [fungi]

- Debug and fix as needed until we get reasonably reliable passes on
  the non-voting g-r jobs. [lifeless]

- Flip the constraints using jobs to voting on g-r, and non-voting
  everywhere else. [fungi]

- Switch over to the constraints using jobs everywhere else in a more
  controlled fashion. [fungi]

- Update release documentation to branch openstack/requirements last, and
  setup the job that will validate requirements changes against the projects
  that have branched already, as well as master itself. [fungi]


Dependencies
============

- None.



History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Liberty
     - Introduced

References
==========

1. https://pythonhosted.org/setuptools/setuptools.html#declaring-extras-optional-features-with-their-own-dependencies

.. note::

  This work is licensed under a Creative Commons Attribution 3.0 Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode
