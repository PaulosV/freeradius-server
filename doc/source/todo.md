= To Do before v4 is releases

## Nested attributes

We need this for DHCPv6, Diameter, and json support.  Nested
attributes are different from TLVs in that TLVs define a child
namespace.  Nested attributes are instead a way to group any
attributes from the parent namespace.

We need to define a new data type `group`.  It can contain any
attribute.  We may also need to add a dictionary to the group, so that
we can do DHCPv4 in RADIUS or vice versa.

Unlang syntax for an attribute called `Grouped-Foo`

```unlang
update request {
	Grouped-Foo {
		User-Name = bar
		Foo = baz
		...
	}
}
```

We can also define a `short circuit` syntax for things like the
`users` file or `sql` module, which don't natively allow for nested grouping.


```
Grouped-Foo.User-Name = bar
.Foo = baz
```

i.e. the inital `.` means "part of the previous group at the level of
the last attribute.

This syntax means that we need to define an API which parses attribute
names / values, but in a *sequential* fashion.  That is, we can't just
assume that the pair list is linear.

The best solution is likely to make this a function which uses maps.
We may need a high-level wrapper / cursor around maps, too.

We will also need an API (cursor, etc. ) to recurse into groups.
Ideally without changing the existing APIs?  But that involves
creating a stack of cursors, instead of one.  This likely means
requiring that cursors are talloc'd, and that we can create `stacks`
of cursors.  e.g.

```
vp_cursor_t *fr_pair_cursor_recurse_child(TALLOC_CTX *ctx, vp_cursor_t *cursor)
```

Which will recurse into the current VP *if* that VP is of
`FR_TYPE_GROUP`, and allocate a new cursor.  That new cursor can then
be used with all of the normal cursor functions.

We can create an API which *automatically* walks down all of the
attributes, but *switching* to that API is likely too hard.  99% of
the code needs simple lists, so leaving the existing API is fine.

If we do want to walk over all of the attributes, we can create a new
API for that. That API should create a `talloc`d cursor, and then bury
the implementation details inside of the cursor.

The good news is that 99% of the existing code doesn't use groups, so
we can add a new API without changing any existing code.

We MAY want the `vp->vp_group` field be a structure which includes a
`fr_dict_t*` root dictionary, a `VALUE_PAIR*` attribute, and perhaps
another pointer describing what's allowed here.

Though to be honest, the "allowed" field is likely not necessary.
This limitation should really be enforced by something else
(e.g. `attr_filter`), and not by the server core.  Adding this
functionality to the server core can make it significantly more
complex.

## xlat functions, with discreet argument value boxes

Arran to update.

## back-end socket abstraction

See `rlm_radius`.  The code is partially there, it just needs to be
separated from the module.

The back-end socket abstraction is the v4 equivalent of the v3
connection pools.  It allows modules to "push" requests to sockets,
and then get the replies.

Once this is done, then doing RADIUS over TCP / TLS / DTLS should be
straightforward.

## redis/couchbase/sql/ldap async

These modules need to be converted to be async.  But they need the
socket abstraction from above.

## Control Socket

We need equivalent functionality to v3, or close enough.

## Named processing sections

Instead of `authorize`, etc., allow for `recv Access-Request`

The modules should export a list of names they support and functions, such as:

* CF_IDENT_ANY
* recv CF_IDENT_ANY
* recv Access-Request
* session start
* session update
* session stop
* session check

_Module export is done_.

We may also need per-method instantitation / parsing functions, as
with `rlm_rest`.  That is still TBD.

We also need a way to call named methods for a module.  Right now, we
do "module.METHOD", which doesn't work well for two names.  We then
extend this to "module.NAME1.NAME2".

_calling module methods is done_

We could also do named subsections?  i.e.

```unlang
recv Access-Request {
	...
	session start {
		module
	}
}
```

_Named subsections is done_

We should have a registry, so that the `fr_app_worker_t` can export a
list of sections it accepts, and what is allowed in those sections.
Then also what dictionary is used, and what packet types are allowed
in those sections.

We will hope that the packet names are different in every protocol.

The processing modules can now export a list of *additional* methods
that they take.  The `*module_by_name_and_method()` function walks
down that list:

  * if method[COMPONTENT] is set, then return that
  * otherwise walk over the named methods for a module, looking
    for a match to the current section we're compiling.
    e.g. "recv Access-Request"
  * if that still isn't found, then look up the list of allowed
    methods for this processing section.  Then, walk over that list
    and the module list in `O(N*M)`, to see if there's a matching
    method

The last step is more rare, so it shouldn't affect speed much.

The processing sections don't (yet) export such additional methods.
