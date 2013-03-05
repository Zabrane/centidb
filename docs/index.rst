
centidb
=======

`http://github.com/dw/centidb <http://github.com/dw/centidb>`_

.. toctree::
    :hidden:
    :maxdepth: 2

`centidb` is a tiny database that provides a tradeoff between the minimalism of
a key/value store and the convenience of SQL. It wraps any store that provides
an ordered-map interface, adding features that often tempt developers to use
more complex systems.

Functionality is provided for forming ordered composite keys, managing and
querying secondary indices, and a binary encoding that preserves the ordering
of tuples of primitive values. Combining the simplicity of a key/value store
with the convenience of a DBMS's indexing system, while absent of any
storage-specific protocol/language/encoding/data model, or the impedence
mismatch that necessitates use of ORMs, it provides for a compelling
programming experience.

Few design constraints are made: there is no enforced value type or encoding,
key scheme, compressor, or storage engine, allowing integration with whatever
best suits or is already used by a project.

Batch value compression is supported, trading read performance for improved
compression ratios, while still permitting easy access to data. Arbitrary key
ranges can be selected for compression and the batch size is configurable.

Since it is a Python library, key and index functions are written directly in
Python rather than some unrelated language.

Why `centi`-db? Because it is over 100 times smaller than alternatives with
comparable features (<400 LOC excluding speedups vs. ~152 kLOC for Mongo).


Basics
######

Store object
++++++++++++

The user's configured storage engine is first wrapped in a `Store` instance:

::

    # Use a LevelDB database for storage.
    db = plyvel.DB('test.ldb', create_if_missing=True)
    store = centidb.Store(PlyvelEngine(db))

The store manages metadata for a set of `Collection` and `Index` objects, along
with any compressors and counters.



treated as an independent set of
`Collections` which access 
The database engine is wrapped by a `Store` object, 


Common Parameters
#################

In addition to those described later, each function accepts the following
optional parameters:

``key``:
  Indicates a function (in the style of ``sorted(..., key=)``) that maps lines
  to ordered values to be used for comparison. Provide ``key`` to extract a
  unique ID or timestamp. Lines are compared lexicographically by default.

``lo``:
  Lowest offset in bytes, useful for skipping headers or to constrain a search
  using a previous search. For line oriented search, one byte prior to this
  offset is included in order to ensure the first line is considered complete.
  Defaults to ``0``.

``hi``:
  Highest offset in bytes. If the file being searched is weird (e.g. a UNIX
  special device), specifies the highest bound to access. By default
  ``getsize()`` is used to probe the file size.



Keys & Indices
##############

When instantiating a Collection you may provide a key function, which is
responsible for producing the unique (primary) key for the record. The key
function can accept either one or two parameters. In the first form, only the
record' value is passed, while in the second form .

is passed three parameters:

    `obj`:
        Which is record value itself. Note this is not the Record instance, but
        the ``Record.data`` (i.e. user data) field.

    `txn`:
        The transaction this modification is a part of. Can be used to
        implement transactional assignment of IDs.

The returned key may be any of the supported primitive values, or a tuple of
primitive values. Note that any non-tuple values returned are automatically
transformed into 1-tuples, and `you should expect this anywhere your code
refers to the record's key`.

For example, to assign a key based on the time in microseconds:

::

    def usec_key(val):
        return int(1e6 * time.time())

Or by UUID:

::

    def uuid_key(val):
        return uuid.uuid4()


Auto-increment
++++++++++++++

When no explicit key function is given, `Collection` defaults to generating
transactionally assigned auto-incrementing integers using `Store.count()`.
Since this doubles the database operations required, auto-incrementing keys
should be used sparingly. Example:

::

    log_msgs = centidb.Collection(store, 'log_msgs')
    log_msgs.put("first")
    log_msgs.put("second")
    log_msgs.put("third")

    assert list(log_msgs.iteritems()) == [
        ((1,), "first"),
        ((2,), "second"),
        ((3,), "third")
    ]

*Note:* as with everywhere, since keys are always tuples, the auto-incrementing
integer was wrapped in a 1-tuple.



Reference
#########

Store Class
+++++++++++

.. autoclass:: centidb.Store
    :members:

Collection Class
++++++++++++++++

.. autoclass:: centidb.Collection
    :members:

Record Class
++++++++++++

.. autoclass:: centidb.Record
    :members:

Index Class
+++++++++++

.. autoclass:: centidb.Index
    :members:


Encodings
#########

.. autoclass:: centidb.Encoder


Predefined Encoders
+++++++++++++++++++

The ``centidb`` module contains the following predefined `Encoder` instances.

    ``KEY_ENCODER``
        Uses `encode_keys()` and `decode_keys()` to serialize tuples. It is
        used internally to represent keys, counters, and `Store` metadata.

    ``PICKLE_ENCODER``
        Uses `cPickle.dumps()` and `cPickle.loads()` with protocol 2 to
        serialize any pickleable object. It is the default encoder if no
        specific `encoder=` argument is given to the `Collection` constructor.

    ``ZLIB_PACKER``
        Uses `zlib.compress()` and `zlib.decompress()` to provide value
        compression. It may be passed as the `packer=` argument to
        `Collection.put()`, or specified as the default using the `packer=`
        argument to the `Collection` constructor.


Thrift Integration
++++++++++++++++++

This uses `Apache Thrift <http://thrift.apache.org/>`_ to serialize Thrift
struct values to a compact binary representation.

Create an `Encoder` factory:

::

    def make_thrift_encoder(klass, factory=None):
        if not factory:
            factory = thrift.protocol.TCompactProtocol.TCompactProtocolFactory()

        def loads(buf):
            transport = thrift.transport.TTransport.TMemoryBuffer(buf)
            proto = factory(transport)
            value = klass()
            value.read(proto)
            return value

        def dumps(value):
            return thrift.TSerialization.serialize(value, factory)

        # Form a name from the Thrift ttypes module and struct name.
        name = 'thrift:%s.%s' % (klass.__module__, klass.__name__)
        return centidb.Encoding(name, loads, dumps)


Create a ``myproject.thrift`` file:

::

    struct Person {
        1: string username,
        2: string city,
        3: i32 age
    }

Now define a collection:

::

    # 'myproject' package is generated by 'thrift --genpy myproject.thrift'
    from myproject.ttypes import Person

    coll = centidb.Collection(store, 'people',
        encoder=make_thrift_encoder(Person))
    coll.add_index('username', lambda person: person.username)
    coll.add_index('age_city', lambda person: (person.age, person.city))

    user = Person(username='David', age=42, city='Trantor')
    coll.put(user)

    assert coll.indices['username'].get('David') == user


Key functions
+++++++++++++

The key encoding is based on SQLite 4's algorithm `as documented here
<http://sqlite.org/src4/doc/trunk/www/key_encoding.wiki>`_, adding support
for UUIDs and `Key` objects, but removing support for floats, using varints for
the integer encoding, and a more scripting-friendly string encoding.

.. autofunction:: centidb.encode_keys
.. autofunction:: centidb.decode_keys
.. autofunction:: centidb.invert


Varint functions
++++++++++++++++

The sortable varint encoding is based on SQLite 4's algorithm `as documented
here <http://sqlite.org/src4/doc/trunk/www/varint.wiki>`_.

.. autofunction:: centidb.encode_int
.. autofunction:: centidb.decode_int


Examples
########

Index Usage
+++++++++++

::

    import itertools
    import centidb
    from pprint import pprint

    import plyvel
    store = centidb.Store(plyvel.DB('test.ldb', create_if_missing=True))
    people = centidb.Collection(store, 'people', key_func=lambda p: p['name'])
    people.add_index('age', lambda p: p['age'])
    people.add_index('name', lambda p: p['age'])
    people.add_index('city_age', lambda p: (p.get('city'), p['age']))

    make_person = lambda name, city, age: dict(locals())

    people.put(make_person('Alfred', 'Nairobi', 46))
    people.put(make_person('Jemima', 'Madrid', 64))
    people.put(make_person('Mildred', 'Paris', 34))
    people.put(make_person('Winnifred', 'Paris', 24))

    # Youngest to oldest:
    pprint(list(people.indices['age'].iteritems()))

    # Oldest to youngest:
    pprint(list(people.indices['age'].itervalues(reverse=True)))

    # Youngest to oldest, by city:
    it = people.indices['city_age'].itervalues()
    for city, items in itertools.groupby(it, lambda p: p['city']):
        print '  ', city
        for person in items:
            print '    ', person

    # Fetch youngest person:
    print people.indices['age'].get()

    # Fetch oldest person:
    print people.indices['age'].get(reverse=True)


Reverse Indices
+++++++++++++++

Built-in support is not yet provided for compound index keys that include
components that are sorted in descending order, however this is easily
emulated:

+-----------+---------------------------------------+
+ *Type*    + *Inversion function*                  |
+-----------+---------------------------------------+
+ Numbers   | ``-i``                                |
+-----------+---------------------------------------+
+ Boolean   + ``not b``                             |
+-----------+---------------------------------------+
+ String    + ``centidb.invert(s)``                 |
+-----------+---------------------------------------+
+ Unicode   + ``centidb.invert(s.encode('utf-8'))`` |
+-----------+---------------------------------------+
+ UUID      + ``centidb.invert(uuid.get_bytes())``  |
+-----------+---------------------------------------+
+ Key       + ``Key(centidb.invert(k))``            |
+-----------+---------------------------------------+

Example:

::

    coll.add_index('name_age_desc',
        lambda person: (person['name'], -person['age']))

Note that if a key contains only a single value, or all the key's components
are in descending order, then transformation is not required as the index
itself can be iterated in reverse:

::

    coll = Collection(store, 'people',
        key_func=lambda person: person['name'])
    coll.add_index('age', lambda person: person['age'])
    coll.add_index('age_height',
        lambda person: (person['age'], person['height']))

    # Not necessary.
    coll.add_index('name_desc',
        lambda person: centidb.inverse(person['name'].encode('utf-8')))

    # Not necessary.
    coll.add_index('age_desc', lambda person: -person['age'])

    # Not necessary.
    coll.add_index('age_desc_height_desc',
        lambda person: (-person['age'], -person['height']))

    # Equivalent to 'name_desc' index:
    it = coll.iteritems(reverse=True)

    # Equivalent to 'age_desc' index:
    it = coll.index['age'].iteritems(reverse=True)

    # Equivalent to 'age_desc_height_desc' index:
    it = coll.index['age_height'].iteritems(reverse=True)


Performance
###########


Notes
#####

Floats
++++++

Float keys are unsupported, partially because I have not needed them, and their
use can roughly be emulated with ``int(f * 1e9)`` or similar. But mainly it is
to defer a choice: should floats order alongside integers? If not, then our
keys don't behave like SQL or Python, causing user surprise. If yes, then
should integers be treated as floats? If yes, then keys will always decode to
float, causing surprise. If no, then a new encoding is needed, wasting ~2 bytes
(terminator, discriminator).

Another option is always treating numbers as float, but converting to int
during decode if they can be represented exactly. This may be less surprising,
since an int will coerce to float during arithmetic, but may cause
once-per-decade bugs: depending on a database key, the expression ``123 /
db_val`` might perform integer or float division.

A final option is adding a `number_factory=` parameter to `decode_keys()`,
which still requires picking a good default.

Non-tuple Keys
++++++++++++++

Keys composed of a single value have much the same trade-offs and problems as
floats: either a heuristic is employed that always treats 1-tuples as single
values, leading to user surprise in some cases, and ugly logic when writing
generic code, or waste a byte for each single-valued key.

In the non-heuristic case, further problems emerge: if the user calls
``get(1)``, should it return the same result as ``get((1,))``? If yes, two
lookups may be required.

If no, then another problem emerges: staticly typed languages. In a language
where we might have a ``Tuple`` type representing the key tuple, every
interface dealing with keys must be duplicated for the single-valued case.
Meanwhile the same problems with lookups and comparison in a dynamic language
also occur.

Another option is to make the key encoding configurable: this would allow
non-tuple keys at a cost to some convenience, but also enable extra uses. For
example, allowing a pure-integer key encoding that could be used to efficiently
represent a `Collection` as an SQL table by leveraging the `OID` type, or to
provide exact emulation of the sort order of other databases (e.g. App Engine).

Metadata Encoding
+++++++++++++++++

Metadata is encoded using the key encoder to allow easy access from another
language, since an implementation absolutely must support the key encoding, it
seemed an obvious choice.

History
+++++++

The first attempt came during 2011 while porting from App Engine and a
Datastore-alike was needed. All alternatives included so much weirdness (Java?
JavaScript? BSON? Auto-magico-sharding? ``PageFaultRetryableSection``?!?) that
I eventually canned the project, rendered incapable of picking something as
*simple as a database* that was *good enough*, overwhelmed by false promises,
fake distinctions and overstated greatness in the endless PR veiled by
marketing site designs, and driven by people for whom the embodiment of
*elegance* is the choice of font on a Powerpoint slide.

Storing data isn't hard: it has effectively been solved **since at least 1972**
when the B-tree appeared, also known as the core of SQLite 3, the core of
MongoDB, and just about 90% of all DBMS wheel reinventions existing in the 40
years since. Yet today when faced with a B-tree adulterated with JavaScript and
a million more dumb concepts, upon rejecting it as **junk** we are instantly
drowned in the torrential cries of a million: *"you just don't get it!"*. I
fear I do get it, all too well, and I hate it.

So this module is borne out of frustration. On a recent project while
experimenting with compression, I again found myself partially implementing
what this module wants to be: a tiny layer that does little but add indices to
a piece of Cold War era technology. No "inventions", no lies, no claims to
beauty, no religious debates about scaleability, just 300ish lines that try to
do one thing right.

And so that remains the primary design goal: **size**. The library should be
*small* and *convenient*. Few baked in assumptions, no overcooked
superstructure of pure whack that won't matter anyway in a year, just indexing
and some helpers to make queries work nicely. If you've read this far, then you
hopefully understand why my receptiveness towards extending this library to be
made "awesome" in some way is all but missing. Patch it at your peril, but
please, bug fixes and obvious omissions only.

Futures
+++++++

1. Support inverted index keys nicely.
2. Avoid key decoding when only used for comparison.
3. Unique index constraint
4. Better documented
5. Smaller
6. Safer
7. Faster
8. C++ library
9. Configurable key scheme
10. Make key/value scheme prefix optional.
11. Make indices work as `Collection` observers, instead of hard-wired.
12. Support "observe-only" `Index` object.
13. Miniature validating+indexing network server module.