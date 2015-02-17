..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Return request ID to caller
===========================

Currently there is no way to return `X-Openstack-Request-Id` to the user from
individual python-clients, python-openstackclient and python-openstacksdk.


Problem description
===================

Most of the OpenStack RESTful API returns `X-Openstack-Request-Id` in the API
response header but this request id is not available to the caller from the
python client. When you run command on the command prompt using client with
debug option, then it displays `X-Openstack-Request-Id` on the console but
if I'm using python-client in some third party applications and if some
api fails due to some unknown reason then there is no way to get
`X-Openstack-Request-Id` from the client. This request id is very useful
to get quick support from infrastructure support team.

Use Cases
---------

1. Our users are asking for `X-Openstack-Request-Id` to be returned from
   python-clients which would help them to get support from service provider
   quickly.

2. Log request id of the caller and callee on the same log message in case of
   api request call crossing service boundaries. This particular use case is
   a future work once the above use case is implemented.

Proposed change
===============

Add a wrapper class around response to add a request id to it which can be
returned back to the caller. We have analyzed 5 different python client
libraries to understand what different types of return values are returned back
to the caller.

1. python-novaclient - Version 2.26.0
2. python-glanceclient - Version 0.19.0
3. python-cinderclient - Version 1.2.3
4. python-keystoneclient - Version 1.6.0
5. python-neutronclient - Version 2.6.0

We have documented the details of return types in the below google spreadsheet.

https://docs.google.com/spreadsheets/d/1al6_XBHgKT8-N7HS7j_L2H5c4CFD0fB8xT93z6REkSk/edit?usp=sharing

There are 9 different types of return values:

**1 List**

Presently, there is no way to return `X-Openstack-Request-Id` for list type.
Add a new wrapper class inherited from list to return request-id back to
the caller.

.. code:: python

    class ListWithMeta(list):
        def __init__(self, values, req_id):
            super(ListWithMeta, self).__init__(values)
            self.request_ids = []
            if isinstance(req_id, list):
                self.request_ids.extend(req_id)
            else:
                self.request_ids.append(req_id)

**2 Dict**

Similar to list type above, there is no way to return `X-Openstack-Request-Id`
for dict type. Add a new wrapper class inherited from dict to return request-id
back to the caller.

.. code:: python

    class DictWithMeta(dict):
        def __init__(self, values, req_id):
            super(DictWithMeta, self).__init__(values)
            self.request_ids = [req_id]

**3 Resource object**

There are various methods that returns different resource objects from clients
(volume/snapshot etc. from python-cinderclient). These resource class don't
have request_ids attribute. Add a request_ids attribute to the resource class
and populate it from the HTTP response object during instantiating resource
objects.

.. code:: python

    # code snippet to add request_id to Resource class
    class Resource(object):
        def __init__(self, manager, info, loaded=False, req_id=None):
            self.manager = manager
            self._info = info
            self._add_details(info)
            self._loaded = loaded
            self.request_ids = [req_id]

**4 Tuple**

Most of the actions returns tuple containing 'Response' object and 'response
body'. Add a new wrapper class inherited from tuple to return request-id
back to the caller. For few actions, it’s returning None at present.
Make changes at all such places to return new wrapper class of tuple.

.. code:: python

    class TupleWithMeta(tuple):
        def __new__(cls, values, request_id):
            obj = super(TupleWithMeta, cls).__new__(cls, values)
            obj.request_ids = [req_id]
            return obj

**5 None**

Mostly all delete/update methods don’t return any value back to the caller.
In most of the clients response and body tuple is returned by api call but
it is not returned back to the caller. Make changes at all such places
and return TupleWithMetaData object which will have request_ids as a
attribute for delete and update cases. There are some corner cases like
deleting metadata where it’s not possible to return request-id back to the
caller as internally it iterates through the list and deletes metadata key
one by one. For such cases, list of request-id's will be returned back to
the caller.

**6 Exception**

For python-cinderclient, python-keystoneclient and python-novaclient provision
is made to pass request-id when exception is raised as base exception class
has attribute request-id. Make similar changes in python-glanceclient,
and python-neutronclient to add request-id to base exception class so that
request-id will be available in case of failure.

**7 Boolean (True/False)**

Couple of python-keystoneclient methods for V3, like check_in_group to check
user is in group or not are returning bool (True/False). Add new wrapper
class inherited from int to return request-id back to the caller.

.. code:: python

    class BoolWithMeta(int):
        def __new__(cls, value, req_id):
            obj = super(BoolWithMeta, cls).__new__(cls, bool(value))
            obj.request_ids = [req_id]
            return obj

        def __repr__(self):
            return ['False', 'True'][self]

**8 Generator**

All list api's are returning generator from python-glanceclient.
In order to return list of request id's in the generator, add a
new wrapper class to wrap the existing generator and implement the iterator
protocol. New wrapper class will have the attribute as 'request_id' of list
type. In the next method of iterator (wrapper class), request_id will be
added to the list based on page size and limit.

.. code:: python

    # code snippet to add request_id to GeneratorWrapper class
    class GeneratorWrapper(object):
        def __init__(self, paginate_func, url, page_size, limit):
            self.paginate_func = paginate_func
            self.url = url
            self.limit = limit
            self.page_size = page_size
            self.generator = None
            self.request_ids = []

        def _paginate(self):
            for obj, req_id in self.paginate_func(
                self.url, self.page_size, self.limit):
                yield obj, req_id

        def __iter__(self):
            return self

        # Python 3 compatibility
        def __next__(self):
            return self.next()

        def next(self):
            if not self.generator:
                self.generator = self._paginate()

            try:
                obj, req_id = self.generator.next()
                if req_id and (req_id not in self.request_ids):
                    self.request_ids.append(req_id)
            except StopIteration:
                raise StopIteration()

            return obj

**9 String**

Couple of nova api's are returning String as a response to the user.
Add a new wrapper class inherited from str to return request-id back to
the caller.

.. code:: python

    class StrWithMeta(str):
        def __new__(cls, value, req_id):
            obj = super(StrWithMeta, cls).__new__(cls, value)
            obj.request_ids = [req_id]
            return obj

**Note:**

To start with, we are proposing to implement this solution in two steps.

*Step 1: Add request-id attribute to base exception class.*

request-id is most needed when api returns anything >= 400 error code.
As of now python-cinderclient, python-keystoneclient and python-novaclient
already has a mechanism to return request-id in exception. Make similar
changes in remaining clients to return request-id in exception.

*Step 2: Add request-id for remaining return types*

Add new wrapper class in common package of oslo-incubator (apiclient/base.py)
and sync oslo-incubator in python-clients to return request-id for remaining
return types.

Alternatives
------------

**Alternative Solution #1**

Step 1:

We are proposing to add 'get_previous_request_id()' method in python-clients,
python-openstackclient and python-openstacksdk to return request id to the
user.

Design

When a caller make a call and get a response from the OpenStack service, it
will extract `X-Openstack-Request-Id` from the response header and store it
in the thread local storage (TLS). Add a new method 'get_previous_request_id()'
in the client to return `X-Openstack-Request-Id` stored in the thread local
storage to the caller. We need to store request id in the TLS because same
client object could be used in multi-threaded application to interact with
the OpenStack services.

.. code:: python

    from cinderclient import client

    cinder = client.Client('2', 'demo', 'admin', 'demo',
                           'http://21.12.4.342:5000/v2.0')

    cinder.volumes.list()
    [<Volume: 88c77848-ef8e-4d0a-9bbe-61ac41de0f0e>,
     <Volume: 4b731517-2f3d-4c93-a580-77665585f8ca>]

    cinder.get_previous_request_id()
    'req-a9b74258-0b21-49c2-8ce8-673b420e20cc'

Notes:

1. If authentication fails or succeeds, in both the cases, request_id is
   set to None in thread local storage because authenticate method will give
   a call to the keystone service, and response header returned will contain
   request_id of keystone service.

2. There might be possibility that request might fail with an exception
   (timeout, service down etc.) before it gets response with the request_id.
   In this case get_previous_request_id() will return request_id of previous
   request and not of current request. To avoid these kind of issues,
   request_id need to be set to None in thread local storage before new
   request is made.

Pros:

* Doesn't break compatibility.
* Minimal changes are required in the client.

Cons:

* Liable to bugs where folk make two calls and then look at the wrong id or
  deletes where N calls are involved - that implies buffering lots of ids
  on the client, which implies an API for resetting it.

Step 2:

Logging request-id of the caller and callee on the same log message.

Once step 1 is implemented and `X-Openstack-Request-Id` is made available in
the python-client, it will be an easy change to log request id of the caller
and callee on the same log message in OpenStack core services where API request
call is crossing service boundaries. This is a future work for which we will
create another specs if required but it's worth mentioning it here to explain
the usefulness of returning `X-Openstack-Request-Id` from python-clients.


**Alternative Solution #2**

An alternative is to register a callback method with the client which will
be invoked after it gets a response from the OpenStack service. This callback
method will contain the response object which contains `X-Openstack-Request-Id`
and URL.

.. code:: python

    def callback_method(response):
        # get `X-Openstack-Request-Id` and URL from response and log
        # it for trouble shooting.

    c = cinder.Client(...)
    c.register_request_id_callback(request_id_mapping)
    volumes = c.list_volumes()

Pros:

* Doesn't break compatibility (meaning OpenStack services consuming python
  client libraries requires no changes in the code if a newer version of
  client library is used).
* Minimal changes are required in the client.
* With this approach, we can log caller and callee request-id in the same log message.

Cons:

* Forces consumers to try to match the call they made to the event,
  which is complex.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------
Primary assignee:
  abhijeet-malawade(abhijeet.malawade@nttdata.com)

Other contributors:
  ankitagrawal(ankit11.agrawal@nttdata.com)

Work Items
----------

* Add request_id attribute in base exception for following projects:

1) python-glanceclient
2) python-neutronclient

* Add new wrapper classes in oslo-incubator openstack/common/apiclient to add
  request-id to the caller.

  **Note:**

  All of the new wrapper classes will be added in the common package of
  oslo-incubator (openstack/common/apiclient/) and later synced with individual
  python clients. It is decided in cross-project meeting [*] to mark openstack
  package as private mainly in python clients which is syncing apiclient python
  package from oslo-incubator project. For example, oslo-incubator/openstack
  should be synced with python-glanceclient as
  glanceclient/_openstack/common/apiclient. For syncing, we will add a new
  config parameter '--private-pkg' in update.py of oslo-incubator. Marking
  openstack python package as private will have impact on all import statements
  which will be refactored in the individual python clients.

* Sync openstack.common.apiclient.common module of oslo-incubator with
  following projects:

1) python-cinderclient

2) python-glanceclient

3) python-novaclient

4) python-neutronclient

5) python-keystoneclient


Dependencies
============

None


Testing
=======

* Unittests for coverage


Documentation Impact
====================

None

References
==========

[*] http://eavesdrop.openstack.org/meetings/crossproject/2015/crossproject.2015-08-04-21.01.log.html

Etherpad

https://etherpad.openstack.org/p/request-id

Blueprints/Bugs

[1] Return request ID to caller(Cinder)

https://blueprints.launchpad.net/python-cinderclient/+spec/return-req-id

[2] Return request ID to caller(Glance)

https://blueprints.launchpad.net/python-glanceclient/+spec/expose-get-x-openstack-request-id

[3] Return request ID to caller(Neutron)

https://blueprints.launchpad.net/python-neutronclient/+spec/expose-get-x-openstack-request-id

[4] Return request ID to caller(Nova)

https://blueprints.launchpad.net/python-novaclient/+spec/expose-get-x-openstack-request-id

[5] Return request ID to caller(Keystone)

https://blueprints.launchpad.net/python-keystoneclient/+spec/expose-get-x-openstack-request-id

[6] python-openstackclient and python-openstacksdk bug

https://bugs.launchpad.net/python-openstacksdk/+bug/1465817

Discussions on cross-project weekly meeting

[1] http://eavesdrop.openstack.org/meetings/crossproject/2015/crossproject.2015-07-28-21.03.log.html
    #topic Return request-id to caller (use thread local to store request-id)
