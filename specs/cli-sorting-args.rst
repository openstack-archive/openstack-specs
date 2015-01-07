==========================================
 CLI Sorting Argument Guidelines
==========================================

To varying degrees, the REST APIs for various projects support sort keys
and sort directions; these sorting options are exposed as python client
arguments. This specification defines the syntax for these arguments so
that there is consistency across clients.

Problem description
===================

Different projects have implemented the CLI sorting options in different
ways. For example:

- Nova: --sort key1[:dir1],key2[:dir2]
- Cinder: --sort_key <key> --sort_dir <dir>
- Ironic: --sort-key <key> --sort-dir <dir>
- Neutron: --sort-key <key1> --sort-dir <dir1>
           --sort-key <key2> --sort-dir <dir2>
- Glance (under review): --sort-key <key1> --sort-key <key2> --sort-dir <dir>

Proposed change
===============

Based on mailing list feedback (see References sections), the consensus is to
follow the syntax that nova currently implements: --sort <key>[:<direction>]

Where the --sort parameter is comma-separated and used to specify one or more
sort keys and directions. A sort direction is optionally appended to each key
and is either 'asc' for ascending or 'desc' for descending.

For example:

  * nova list --sort display_name
  * nova list --sort display_name,vm_state
  * nova list --sort display_name:asc,vm_state:desc
  * nova list --sort display_name,vm_state:asc

Unfortunately, the REST APIs for each project support sorting to different
degrees:

- Nova and Neutron: Multiple sort keys and multiple sort directions
- Cinder and Ironic: Single sort key and single sort direction (Note: approved
  kilo spec in Cinder for adding adding for multiple key and direction
  support)
- Glance: Multiple sort keys and single sort direction

In the event that the corresponding REST APIs do not support multiple sort
keys and multiple sort directions, the client may:

- Support a single key and direction
- Support multiple keys and directions and implement any remaining sorting
  in the client

Alternatives
------------

Each sort key and associated direction could be supplied independently, for
example:

--sort-key key1 --sort-dir dir1 --sort-key key2 --sort-dir dir2

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * Cinder: Steven Kaufer (kaufer)
  * Glance: Mike Fedosin (mfedosin)

Work Items
----------

Cinder:
  * Deprecate --sort_key and --sort_dir and add support for --sort
  * Note that the Cinder REST API currently supports only a single sort key
    and direction so the CLI will have the same restriction, this restriction
    can be lifted once the following is implemented:
    https://blueprints.launchpad.net/cinder/+spec/cinder-pagination

Ironic/Neutron:
  * Deprecate --sort-key and --sort-dir and add support for --sort

Glance:
  * Modify the existing patch set to adopt the --sort parameter:
    https://review.openstack.org/#/c/120777/
  * Note that Glance supports multiple sort keys but only a single sort
    direction.


Dependencies
============

- Cinder BP for multiple sort keys and directions:
  https://blueprints.launchpad.net/cinder/+spec/cinder-pagination

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Kilo
     - Introduced

References
==========

- Nova review that implemented the --sort argument:
  https://review.openstack.org/#/c/117591/
- Glance client review: https://review.openstack.org/#/c/120777/
- http://www.mail-archive.com/openstack-dev@lists.openstack.org/msg42854.html
- http://www.mail-archive.com/openstack-dev%40lists.openstack.org/msg42954.html

.. note::

  This work is licensed under a Creative Commons Attribution 3.0 Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode
