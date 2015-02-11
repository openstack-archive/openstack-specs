========================================
 Managing Stable Branches for Libraries
========================================

Problem description
===================

We want to restrict stable branches to a narrow range of allowed
versions of all dependencies, to increase our chances of avoiding
breaking changes in the stable branches.

This means rather than always rolling all library releases forward to
the most current release, we want to start using stable releases for
libraries more consistently.

We also want to express the dependency range in a way that does not
require that we explicitly modify the allowed version when we produce
patch releases. So we want to place a cap on the upper bound of the
version range, rather than pinning to a specific version.

The Oslo team has been considering this problem for a while, but we
now have more teams producing libraries, either server-support
libraries or clients, so we need to bring the plan to a wider audience
and apply a consistent set of procedures to all OpenStack libraries,
including Oslo, project-specific clients, the SDK, middleware, and any
other projects that are installed as subcomponents of the rest of
OpenStack. (The applications follow a stable branch process already,
so this process describes how to handle stable branches for projects
that are not applications.)

All libraries should already be using the `pbr version of SemVer`_ for
version numbering. This lets us express a dependency within the range
of major.minor, and allow any patch release to meet the
requirement. This spec discusses the process for ensuring that we have
valid stable branches to work from.

.. _pbr version of SemVer: http://docs.openstack.org/developer/pbr/semver.html

Proposed change
===============

When we reach the feature freeze at the end of the cycle, libraries
should freeze development a little before the application freeze date
(Oslo freezes 1 week prior, which should be enough time for all
projects to follow these procedures). At that point, the release
manager for each library should follow the steps below, substituting
the release series name (for example, "kilo") for ``$SERIES``.

#. Update the global requirements list to make the current version of
   the library the minimum supported version, and express the
   requirement using the "compatible version" operator (``~=``) to
   allow for future patch releases (see the `Compatible Releases`_
   section of :pep:`440`).

   To avoid churn in the applications, we probably want to do this in
   one, or at least just a very few, patches, so we will need to
   coordinate between the release managers to get that set up.

#. Create a ``stable/$SERIES`` branch in each library from the same
   commit that was tagged, to provide a place to back-port changes for
   the stable branch.

   The release team for each project will be responsible for this
   step, so we will want to automate it as much as possible.

The rest of the requirements management for the release is largely
unchanged, with one final step added:

#. Freeze the master branch of the global requirements repository at
   the Feature Freeze date.

#. All projects merge the latest global requirements before issuing
   their RC1.

#. When projects tag their RC1 they create ``proposed/$SERIES``
   branches.

#. When all integrated projects have done their RC1, we create a
   requirements ``proposed/$SERIES`` branch and unfreeze master

#. After all applications have released their RC1 and have created
   their ``proposed/$SERIES`` branches, the caps on the global
   requirements list can be removed on the master branch.

.. _Compatible Releases: https://www.python.org/dev/peps/pep-0440/#compatible-release

New stable releases of the library can then proceed as before, tagging
new patch releases in the stable branch instead of master. Stable
releases of libraries are expected to be exceptions, to support
security or serious bug fixes. Trivial bug fixes will not necessarily
be back-ported. As with applications stable releases of libraries
should not include new features and should have a high level of
backwards-compatibility.

The global requirements updates for our own libraries should be merged
into the applications requirements list before their RC1 is produced
to ensure that we don't have any releases with conflicting
requirements.

The next release on master for each library should use a new minimum
version number to move it out of the stable release series.  We will
have cut the stable branch at that point, so bug fixes will have to be
a back-ported anyway. Historically feature patches that didn't make it
before the freeze have merged early in the next cycle. Taking both of
those factors together means it will just be simpler to always cut a
release with a new minor version to avoid any issues with later
back-ports or with accidentally including features in the release
going to the new stable branch.

Management of the stable branches is left up to the projects to
decide, but it should not be assumed that the stable maintenance team
will directly handle all back-ports.

Alternatives
------------

Use a "proposed" Branch before the Stable Branch
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We could follow the two-step process the applications use and create a
``proposed/$SERIES`` branch before the final ``stable``
branch. However, the library code bases are smaller and tend to have
fewer changes in flight at any one time than the applications, so this
would be extra overhead in the process. We haven't found many cases in
the past where we need to back-port changes from master to the stable
branches, so it shouldn't be a large amount of work to do that as
needed.

The branches for libraries are also created *after* a release, and so
they are not a "proposed" release.

Create Stable Branches as Needed
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As mentioned above, we waited to create stable branches for some of
the Oslo libraries until they were needed. This introduced extra time
into the process because a back-port patch couldn't be submitted for
review until the branch existed.

Create Numbered Branches
~~~~~~~~~~~~~~~~~~~~~~~~

We could also create branches like ``stable/1.2`` or using some other
prefix. However, this makes it more difficult to set up the test jobs
using regexes against the branch names, and especially the job that
tests proposed changes to stable branches of libraries will be more
difficult to configure properly. Using the release name as the branch
name lets all of this work "automatically" using our existing tools.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Doug Hellmann

Work Items
----------

1. Review and update the scripts in ``openstack-infra/release-tools``
   to find any that need to be updated to support libraries.

.. I'll need to talk to Thierry about which scripts might need to be
   updated and if there are any other written instructions that we
   need to update, but I wanted to get the first draft of this spec
   out for review.


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
