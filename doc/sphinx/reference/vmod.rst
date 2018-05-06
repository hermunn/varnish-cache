.. _ref-vmod:

%%%%%%%%%%%%%%%%%%%%%%
VMOD - Varnish Modules
%%%%%%%%%%%%%%%%%%%%%%

For all you can do in VCL, there are things you can not do.
Look an IP number up in a database file for instance.
VCL provides for inline C code, and there you can do everything,
but it is not a convenient or even readable way to solve such
problems.

This is where VMODs come into the picture:   A VMOD is a shared
library with some C functions which can be called from VCL code.

For instance::

	import std;

	sub vcl_deliver {
		set resp.http.foo = std.toupper(req.url);
	}

The "std" vmod is one you get with Varnish, it will always be there
and we will put "boutique" functions in it, such as the "toupper"
function shown above.  The full contents of the "std" module is
documented in vmod_std(3).

This part of the manual is about how you go about writing your own
VMOD, how the language interface between C and VCC works, where you
can find contributed VMODs etc. This explanation will use the "std"
VMOD as example, having a Varnish source tree handy may be a good
idea.

VMOD Directory
==============

The VMOD directory is an up-to-date compilation of maintained
extensions written for Varnish Cache:

    https://www.varnish-cache.org/vmods

The vmod.vcc file
=================

The interface between your VMOD and the VCL compiler ("VCC") and the
VCL runtime ("VRT") is defined in the vmod.vcc file which a python
script called "vmodtool.py" turns into thaumaturgically challenged C
data structures that does all the hard work.

The std VMODs vmod.vcc file looks somewhat like this::

	$Module std 3
	$Event event_function
	$Function STRING toupper(STRING_LIST)
	$Function STRING tolower(STRING_LIST)
	$Function VOID set_ip_tos(INT)

The first line gives the name of the module and the manual section where
the documentation will reside.

The second line specifies an optional "Event" function, which will be
called whenever a VCL program which imports this VMOD is loaded or
transitions to any of the warm, active, cold or discarded states.
More on this below.

The next three lines define three functions in the VMOD, along with the
types of the arguments, and that is probably where the hardest bit of
writing a VMOD is to be found, so we will talk about that at length in
a moment.

Notice that the third function returns VOID, that makes it a "procedure"
in VCL lingo, meaning that it cannot be used in expressions, right side
of assignments and such.  Instead it can be used as a primary action,
something functions which return a value can not::

	sub vcl_recv {
		std.set_ip_tos(32);
	}

Running vmodtool.py on the vmod.vcc file, produces a "vcc_if.c" and
"vcc_if.h" files, which you must use to build your shared library
file.

Forget about vcc_if.c everywhere but your Makefile, you will never
need to care about its contents, and you should certainly never
modify it, that voids your warranty instantly.

But vcc_if.h is important for you, it contains the prototypes for
the functions you want to export to VCL.

For the std VMOD, the compiled vcc_if.h file looks like this::

	struct vmod_priv;

	VCL_STRING vmod_toupper(VRT_CTX, const char *, ...);
	VCL_STRING vmod_tolower(VRT_CTX, const char *, ...);
	VCL_VOID vmod_set_ip_tos(VRT_CTX, VCL_INT);

	vmod_event_f event_function;

Those are your C prototypes.  Notice the ``vmod_`` prefix on the
function names.

Named arguments and default values
----------------------------------

The basic vmod.vcc function declaration syntax introduced above makes all
arguments mandatory for calls from vcl - which implies that they need
to be given in order.

Naming the arguments as in::

	$Function BOOL match_acl(ACL acl, IP ip)

allows calls from VCL with named arguments in any order, for example::

	if (debug.match_acl(ip=client.ip, acl=local)) { # ...

Named arguments also take default values, so for this example from
the debug vmod::

	$Function STRING argtest(STRING one, REAL two=2, STRING three="3",
				 STRING comma=",", INT four=4)

only argument `one` is required, so that all of the following are
valid invocations from vcl::

	debug.argtest("1", 2.1, "3a")
	debug.argtest("1", two=2.2, three="3b")
	debug.argtest("1", three="3c", two=2.3)
	debug.argtest("1", 2.4, three="3d")
	debug.argtest("1", 2.5)
	debug.argtest("1", four=6);

The C interface does not change with named arguments and default
values, arguments remain positional and default values appear no
different to user specified values.

`Note` that default values have to be given in the native C-type
syntax, see below. As a special case, ``NULL`` has to be given as ``0``.

.. _ref-vmod-vcl-c-types:

VCL and C data types
====================

VCL data types are targeted at the job, so for instance, we have data
types like "DURATION" and "HEADER", but they all have some kind of C
language representation.  Here is a description of them.

All but the PRIV and STRING_LIST types have typedefs: VCL_INT, VCL_REAL,
etc.

ACL
	C-type: ``const struct vrt_acl *``

	A type for named ACLs declared in VCL.

BACKEND
	C-type: ``const struct director *``

	A type for backend and director implementations. See
	:ref:`ref-writing-a-director`.

BLOB
	C-type: ``const struct vmod_priv *``

	An opaque type to pass random bits of memory between VMOD
	functions.

BOOL
	C-type: ``unsigned``

	Zero means false, anything else means true.

BYTES
	C-type: ``double``

	Unit: bytes.

	A storage space, as in 1024 bytes.

DURATION
	C-type: ``double``

	Unit: seconds.

	A time interval, as in 25 seconds.

ENUM
	vcc syntax: ENUM { val1, val2, ... }

	vcc example: ``ENUM { one, two, three } number="one"``

	C-type: ``const char *``

	Allows values from a set of constant strings. `Note` that the
	C-type is a string, not a C enum.

HEADER
	C-type: ``const struct gethdr_s *``

	These are VCL compiler generated constants referencing a
	particular header in a particular HTTP entity, for instance
	``req.http.cookie`` or ``beresp.http.last-modified``.  By passing
	a reference to the header, the VMOD code can both read and write
	the header in question.

	If the header was passed as STRING, the VMOD code only sees
	the value, but not where it came from.

HTTP
	C-type: ``struct http *``

	TODO

INT
	C-type: ``long``

	A (long) integer as we know and love them.

IP
	C-type: ``const struct suckaddr *``

	This is an opaque type, see the ``include/vsa.h`` file for
	which primitives we support on this type.

PRIV_CALL
	See :ref:`ref-vmod-private-pointers` below.

PRIV_TASK
	See :ref:`ref-vmod-private-pointers` below.

PRIV_TOP
	See :ref:`ref-vmod-private-pointers` below.

PRIV_VCL
	See :ref:`ref-vmod-private-pointers` below.

PROBE
	C-type: ``const struct vrt_backend_probe *``

	A named standalone backend probe definition.

REAL
	C-type: ``double``

	A floating point value.

STRING
	C-type: ``const char *``

	A NUL-terminated text-string.

	Can be NULL to indicate a nonexistent string, for instance in::

		mymod.foo(req.http.foobar);

	If there were no "foobar" HTTP header, the vmod_foo()
	function would be passed a NULL pointer as argument.

	When used as a return value, the producing function is
	responsible for arranging memory management.  Either by
	freeing the string later by whatever means available or
	by using storage allocated from the client or backend
	workspaces.

STEVEDORE
	C-type: ``const struct stevedore *``

	A storage backend.

STRING_LIST
	C-type: ``const char *, ...``

	A multi-component text-string.  We try very hard to avoid
	doing text-processing in Varnish, and this is one way we
	to avoid that, by not editing separate pieces of a string
	together to one string, unless we have to.

	Consider this contrived example::

		set req.http.foo = std.toupper(req.http.foo + req.http.bar);

	The usual way to do this, would be be to allocate memory for
	the concatenated string, then pass that to ``toupper()`` which in
	turn would return another freshly allocated string with the
	modified result.  Remember: strings in VCL are ``const``, we
	cannot just modify the string in place.

	What we do instead, is declare that ``toupper()`` takes a "STRING_LIST"
	as argument.  This makes the C function implementing ``toupper()``
	a vararg function (see the prototype above) and responsible for
	considering all the ``const char *`` arguments it finds, until the
	magic marker "vrt_magic_string_end" is encountered.

	Bear in mind that the individual strings in a STRING_LIST can be
	NULL, as described under STRING, that is why we do not use NULL
	as the terminator.

	Right now we only support STRING_LIST being the last argument to
	a function, we may relax that at a latter time.

	If you don't want to bother with STRING_LIST, just use STRING
	and make sure your workspace_client and workspace_backend params
	are big enough.

TIME
	C-type: ``double``

	Unit: seconds since UNIX epoch.

	An absolute time, as in 1284401161.

VOID
	C-type: ``void``

	Can only be used for return-value, which makes the function a VCL
	procedure.


.. _ref-vmod-private-pointers:

Private Pointers
================

It is often useful for library functions to maintain local state,
this can be anything from a precompiled regexp to open file descriptors
and vast data structures.

The VCL compiler supports the following private pointers:

* ``PRIV_CALL`` "per call" private pointers are useful to cache/store
  state relative to the specific call or its arguments, for instance a
  compiled regular expression specific to a regsub() statement or a
  simply caching the last output of some expensive lookup.

* ``PRIV_TASK`` "per task" private pointers are useful for state that
  applies to calls for either a specific request or a backend
  request. For instance this can be the result of a parsed cookie
  specific to a client. Note that ``PRIV_TASK`` contexts are separate
  for the client side and the backend side, so use in
  ``vcl_backend_*`` will yield a different private pointer from the
  one used on the client side.

* ``PRIV_TOP`` "per top-request" private pointers live for the
  duration of one request and all its ESI-includes. They are only
  defined for the client side. When used from backend VCL subs, a NULL
  pointer will be passed.

* ``PRIV_VCL`` "per vcl" private pointers are useful for such global
  state that applies to all calls in this VCL, for instance flags that
  determine if regular expressions are case-sensitive in this vmod or
  similar. The ``PRIV_VCL`` object is the same object that is passed
  to the VMOD's event function.

The way it works in the vmod code, is that a ``struct vmod_priv *`` is
passed to the functions where one of the ``PRIV_*`` argument types is
specified.

This structure contains three members::

	typedef void vmod_priv_free_f(void *);
	struct vmod_priv {
		void                    *priv;
		int			len;
		vmod_priv_free_f        *free;
	};

The "priv" element can be used for whatever the vmod code wants to
use it for, it defaults to a NULL pointer.

The "len" element is used primarily for BLOBs to indicate its size.

The "free" element defaults to NULL, and it is the modules responsibility
to set it to a suitable function, which can clean up whatever the "priv"
pointer points to.

When a VCL program is discarded, all private pointers are checked
to see if both the "priv" and "free" elements are non-NULL, and if
they are, the "free" function will be called with the "priv" pointer
as the only argument.

In the common case where a private data structure is allocated with
malloc would look like this::

	if (priv->priv == NULL) {
		priv->priv = calloc(1, sizeof(struct myfoo));
		AN(priv->priv);
		priv->free = free;	/* free(3) */
		mystate = priv->priv;
		mystate->foo = 21;
		...
	} else {
		mystate = priv->priv;
	}
	if (foo > 25) {
		...
	}

The per-call vmod_privs are freed before the per-vcl vmod_priv.

.. _ref-vmod-event-functions:

Event functions
===============

VMODs can have an "event" function which is called when a VCL which
imports the VMOD is loaded or discarded.  This corresponds to the
``VCL_EVENT_LOAD`` and ``VCL_EVENT_DISCARD`` events, respectively.
In addition, this function will be called when the VCL temperature is
changed to cold or warm, corresponding to the ``VCL_EVENT_COLD`` and
``VCL_EVENT_WARM`` events.

The first argument to the event function is a VRT context.

The second argument is the vmod_priv specific to this particular VCL,
and if necessary, a VCL specific VMOD "fini" function can be attached
to its "free" hook.

The third argument is the event.

If the VMOD has private global state, which includes any sockets or files
opened, any memory allocated to global or private variables in the C-code etc,
it is the VMODs own responsibility to track how many VCLs were loaded or
discarded and free this global state when the count reaches zero.

VMOD writers are *strongly* encouraged to release all per-VCL resources for a
given VCL when it emits a ``VCL_EVENT_COLD`` event. You will get a chance to
reacquire the resources before the VCL becomes active again and be notified
first with a ``VCL_EVENT_WARM`` event. Unless a user decides that a given VCL
should always be warm, an inactive VMOD will eventually become cold and should
manage resources accordingly.

An event function must return zero upon success. It is only possible to fail
an initialization with the ``VCL_EVENT_LOAD`` or ``VCL_EVENT_WARM`` events.
Should such a failure happen, a ``VCL_EVENT_DISCARD`` or ``VCL_EVENT_COLD``
event will be sent to the VMODs that succeeded to put them back in a cold
state. The VMOD that failed will not receive this event, and therefore must
not be left half-initialized should a failure occur.

If your VMOD is running an asynchronous background job you can hold a reference
to the VCL to prevent it from going cold too soon and get the same guarantees
as backends with ongoing requests for instance. For that, you must acquire the
reference by calling ``VRT_ref_vcl`` when you receive a ``VCL_EVENT_WARM`` and
later calling ``VRT_rel_vcl`` once the background job is over. Receiving a
``VCL_EVENT_COLD`` is your cue to terminate any background job bound to a VCL.

You can find an example of VCL references in vmod-debug::

	priv_vcl->vclref = VRT_ref_vcl(ctx, "vmod-debug");
	...
	VRT_rel_vcl(&ctx, &priv_vcl->vclref);

In this simplified version, you can see that you need at least a VCL-bound data
structure like a ``PRIV_VCL`` or a VMOD object to keep track of the reference
and later release it. You also have to provide a description, it will be printed
to the user if they try to warm up a cooling VCL::

	$ varnishadm vcl.list
	available  auto/cooling       0 vcl1
	active     auto/warm          0 vcl2

	$ varnishadm vcl.state vcl1 warm
	Command failed with error code 300
	Failed <vcl.state vcl1 auto>
	Message:
		VCL vcl1 is waiting for:
		- vmod-debug

In the case where properly releasing resources may take some time, you can
opt for an asynchronous worker, either by spawning a thread and tracking it, or
by using Varnish's worker pools.

Cache behavior VMODs
====================

VMODs can also modify the behavior of Varnish that is not strictly
within a single VCL call. For example, a VMOD can provide an alternate
algorithm for selecting object from the cache during lookup (that is
between `vcl_hash` and `vcl_hit` / `vcl_miss`).

Cache lookup, insertion and removal
-----------------------------------

In the core of Varnish Cache is the actual cache. Objects are
inserted into the cache after `vcl_backend_response` (actually, a busy
object is inserted during lookup, and is turned into a "real" object
after vcl_backend_response), retrieved from the cache during lookup
(after `vcl_hash`) and deleted from the cache either to make room for
new objects, or when the expiry thread finds objects that has run
completely stale (ttl + grace + keep has expired). Cache insertion are
affected by VCL in `vcl_backend_response`, and cache lookup relies on
`vcl_recv` and `vcl_hash`.

The ability to affect cache insertion and lookup can be extended by
VMODs, and this enables you to use new caching strategies.

The standard VMOD provides a simple example on how this can be done. /
The VMOD *newsflash*, bundled with Varnish Cache, provides caching
behavior modifications that is useful for news organizations. It
demonstrates how a hashing behavior VMOD can be written and can be
used as a template for new similar VMODs.

Selecting alternate behavior on lookup
--------------------------------------

During `vcl_recv` (or `vcl_hash`), a call to a VMOD can change each
object's fitness for serving to a client, and thus how Varnish will
react during lookup.

For example, consider the following VCL code::

	sub vcl_recv {
		if (my_backend.is_healthy()) {
			// either
			newsflash.set_max_grace(10s);
			// or (more Varnish-like?)
			set_lookup_behavior(newsflash.max_grace(10s));
		}
	}

The result is that, when all the objects for the given hash is
considered, grace is capped at 10 seconds. Apart from that, everything
is as before. Note that if the backend is sick, then the VCL will not
install a custom behavior during lookup, and the object's grace values
are respected.

Behind the scenes, the VMOD will install a set of callbacks that will
be used during cache lookup.

* A callback that will be called once per *candidate object* in the
  cache. A candidate object is an object from the same hash, that
  matches Vary headers. In other words, the *candidate objects* are
  exactly the ones that are considered by the unmodified lookup
  behavior, after filtering for Vary. This callback will get a
  PRIV_TASK object in addition to the reference to the candidate
  object.
* Possibly special callbacks for *busy object*, *hit-for-miss*,
  *hit-for-pass* or similar.
* A *candidate fini* function where the VMOD is asked: "These are all
  the objects, what should we do now?". There are several
  possibilities; HIT, MISS, WAIT, PASS. For HIT and WAIT, the VMOD can
  choose if it wants to initiate a new fetch. (Note: If the VMOD
  returns WAIT but still initiates a fetch, this will be a background
  fetch, and there will be no guarantee that the resulting object will
  be served to the request when the fetch finishes, and this contrasts
  normal Varnish behavior. The VMOD itself then needs to make sure it
  does not go into an infinite loop, where it keeps going inserting
  objects and going back to the waiting list. Always going to the
  waiting list is a good idea if you want *long polling*. Then you
  will use one thread (the fetch thread) to wait for data on the
  socket, and the client thread gets to disembark instead of waiting
  for the backend thread to fetch the object.)

Note that, after lookup has finished, and control is handed over to
`vcl_hit` or `vcl_miss`, the VMOD still holds it PRIV_TASK data. This
means that one can get further information on the result of the
lookup. For example, the VMOD can remember how many object were
considered, and return this information in `vcl_deliver`. However, in
these cases the VMOD must not use the object pointer it got in a
callback (since it has let the *oh lock* go).

Affecting caching through special objects
-----------------------------------------

In `vcl_backend_response`, a VMOD can be ask to insert a special
object that is similar to *hit-for-miss* and *hit-for-pass* objects
(actually these objects are old objects that have a flag set, and
where the body is thrown away after delivery, so something slightly
different is needed here), and that will only be used to inform the
VMOD itself about future caching behavior. For example, if a backend
signals that it cannot produce a new version of an object, and that
stale objects should be used for a while, then such an object can be
inserted. This enables *stale-if-error* behavior with a timeout
similar to the existing *hit-for-miss* and *hit-for-pass*
mechanisms. The *newsflash* VMOD provides this functionality through
the function ??? that can be called in
`vcl_backend_response`. (Previous hacks to get *stale-if-error*
behavior has involved a restart, and have not been able to clear out
the waiting list in a satisfactory way.)

Up for discussion: An important point with these objects is that only
the VMOD itself will see them, so that caching behavior will not
change if the VCL is switched, and subsequent requests are handled
without use of a caching behavior VMOD.

Writing a caching behavior VMOD
-------------------------------

Typically, you should not write a caching behavior VMOD unless you are
intimately familiar with the core of Varnish. If you do, read through
the code of the included `newsflash` VMOD before you start to implement your
own.

However, even if you are not a core Varnish Developer, writing a
caching behavior VMOD should not be completely impossible. Here are
the steps you need to do. From now on we assume that your vmod is
called ``my_vmod``.

Implement an initialization function
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The initialization function function will be called from
`vcl_recv`. Its role is to initialize the PRIV_TASK structure and
install callback functions. It should have the following form::

	VCL_INT my_foo (parameters)
	{
		// 1. Check if we are in vcl_recv
		// 2. Allocate and initialize the priv object from the parameters
		// 3. Install callback functions
	}

Create the callback functions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The callback functions are called under the *OH lock*, where Varnish
loops over all the candidate objects. This means that there are other
rules about what the VMOD can do should not do. These are the most
important changes:

* You must not allocate bytes on the workspace. Instead you should
  allocate the amount you need during the initialization function, and
  use `malloc` if you need more space. Try to avoid `malloc` if you
  can.

* You should not make any calls that will try to grab the OH lock for
  the current transaction, because this will cause a race condition.

* You should not do any significant calculations or potentially
  expensive system calls. This includes everything that might use the
  network.

* You can store pointers to objects (i.e. `struct objcore *`) that are
  handed to you in the callbacks, but you should not grab any
  references to the objects. These pointers will be valid until you
  return from your fini function (see below)

For each normal candidate object, the same vmod function, registered in
vcl_recv, will be called. Implement this function::

	int my_obj_candidate_cb(struct worker *wrk, void *priv, struct objcore *oc)

It should inspect the object (represented by `*oc`) and maybe store it
in the priv structure. The idea is to remember the best candidate for
later. Be very restrictive about inspecting the object (by working on
its headers, for example), since the callback functions may be called
many times per transaction.

For each special object (hit-for-miss, hit-for-pass or the VMODs own
similar object), a different function will be called. In other words,
you should implement::

	int my_obj_meta_cb(struct worker *wrk, void *priv, struct objcore *oc,
	    enum metatype m)

The last parameter is of type ???, which is defined in ???. The role
of this function is to either abruptly stop processing or to change
the reply of the fini function.

If either of `my_obj_candidate_cb` or `my_obj_meta_cb` returns
non-zero (true), then Varnish will skip the rest of the candidate
objects, and go straight to the fini function.

Finally, implement the fini function::

	enum vmod_lookup_fini_e my_lookup_fini_cb(struct worker *wrk,
	    void *priv,	struct objcore **ocp, int *insert_boc)

This will determine the result of the lookup. The possible return values are

* `lookup_fini_miss`: Go straight to `sub vcl_miss`. If an `oc` is
  inserted into `ocp`, this will be used as a 304 candidate.
* `lookup_fini_hit`: Go straight to `sub vcl_hit` and carry out the
  logic there. In this case it is important to notice that the
  built-in VCL can `return (miss)` when you actually want to deliver
  the object. Varnish will panic if `ocp` is not set to a valid object.
* `lookup_fini_pass`: Go straight to `sub vcl_pass` and carry out the
  logic there.
* `lookup_fini_deliver`: Bypass `vcl_hit` and `vcl_miss` entirely and
  deliver the object. Varnish will panic if `ocp` is not set to a
  valid object.
* `lookup_fini_wait`: Go back to the waiting list, and wait for fetch
  to finish. This will assert if there is no busy object and
  `insert_boc` is not set to a non-zero value.

The *out* parameter `insert_boc` is used to indicate if Varnish should
initiate a fetch. It has to be nonzero for `lookup_fini_miss` and can
be either for the other values. Upon entry, it will be `1` if and only
if there no busy object was shown to the VMOD through a call to
`my_obj_candidate_cb`. Note that zero in `*insert_boc` upon entry does
not mean that there are no busy object in the list since the callbacks
have the ability to stop processing by returning 1 in a callback.

The pointer `ocp` is initialized to NULL unless a callback return a
nonzero value, in which `ocp` will point to the corresponding object.

Upon return, if the fini function returns `lookup_fini_hit` and the
corresponding object is either *hit-for-pass* or *hit-for-miss*, then
the corresponding action will happen (pass or miss) instead of
hit. This means that::

	if (boc) {
		return (lookup_fini_hit);
	}

will be valid for most cases (when the `my_obj_meta_cb` function
always returns 0 for meta objects that are not *hit-for-pass* or
*hit-for-miss*).

Create the functions to add meta objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In `vcl_backend_response` it can be useful to insert extra meta
objects that will inform client side tasks about changes in how the
objects should be served. The classical example is a meta object that
extends the grace time of stored objects for a while. This is
exemplified in the `newsflash` vmod.

Create other functions that can be useful on the client side
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The PRIV structure can contain arbitrary information that is kept
during the processing of the request. A VMOD can expose this
information through normal VMOD functions.

When to lock, and when not to lock
==================================

Varnish is heavily multithreaded, so by default VMODs must implement
their own locking to protect shared resources.

When a VCL is loaded or unloaded, the event and priv->free are
run sequentially all in a single thread, and there is guaranteed
to be no other activity related to this particular VCL, nor are
there init/fini activity in any other VCL or VMOD at this time.

That means that the VMOD init, and any object init/fini functions
are already serialized in sensible order, and won't need any locking,
unless they access VMOD specific global state, shared with other VCLs.

Traffic in other VCLs which also import this VMOD, will be happening
while housekeeping is going on.

Updating VMODs
==============

A compiled VMOD is a shared library file which Varnish dlopen(3)'s
using flags RTLD_NOW | RTLD_LOCAL.

As a general rule, once a file is opened with dlopen(3) you should
never modify it, but it is safe to rename it and put a new file
under the name it had, which is how most tools installs and updates
shared libraries.

However, when you call dlopen(3) with the same filename multiple
times it will give you the same single copy of the shared library
file, without checking if it was updated in the meantime.

This is obviously an oversight in the design of the dlopen(3) library
function, but back in the late 1980s nobody could imagine why a
program would ever want to have multiple different versions of the
same shared library mapped at the same time.

Varnish does that, and therefore you must restart the worker process
before Varnish will discover an updated VMOD.

If you want to test a new version of a VMOD, while being able to
instantly switch back to the old version, you will have to install
each version with a distinct filename or in a distinct subdirectory
and use ``import foo from "...";`` to reference it in your VCL.

We're not happy about this, but have found no sensible workarounds.
