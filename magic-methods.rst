Understanding python magic methods by reading Django queryset source code.
-------------------------------------------------------------------------------------

Django querysets are amazing. We use them everyday, but rarely think
about the wonderful API they give us. Just some of the amazing
properties which queysets have

-  You can get a slice ``queryset[i:j]`` out of them, only the needed
   objects are pulled from DB.
-  You can lookup a specifc object ``queryset[i]``, only the required
   object is pulled from DB.
-  You can iterate over them, ``for user in users_queryset``, as if they
   were a list.
-  You can ``AND`` or ``OR`` them and they apply the criteria at the SQL
   level.
-  You can use them like a boolean,
   ``if users_queryset: users_queryset.update(first_name="Batman")``
-  You can pickle and unpickle them, even when the individual istances
   may not be.
-  You can get a useful representation of the queryset in python cli, or
   ipython. Even if the queryset consists of 1000s of records, only
   first 20 records will be printed and shown.

Querysets get all of these properties by implemnting the Python magic
methods, aka the dunder methods. So why do you need these magic, dunder
methods? **Because they make the api much cleaned to use.**

It is more intutive to say,
``if users_queryset: users_queryset.do_something()`` than
``if users_queryset.as_boolean: users_queryset.do_something()``. It is
more intutive to say ``queryset_1 & queryset_2`` rather than
``queryse_1.do_and(queryset_2)``

Magic methods are metods implemented by classes which have a special
meaning to the Python interpretor. They always start with a ``__`` and
are sometimes called **dunder** method. (Dunder == double underscore).

Query and related classes implement the following methods to get the
properies we listed above.

-  ``__getitem__``: For ``queryset[i:j]`` and ``queryset[i]``
-  ``__iter__`` for ``for user in users_queryset``
-  ``__and__`` and ``__or__`` for ``queryset_1 & queryset_2`` and
   ``queryset_1 | queryset_2``
-  ``__bool__`` to use them like a boolean
-  ``__getstate__`` and ``__setstate__`` to pickle and unpickle them
-  ``__repr__`` to get a useful representation and to limit the DB hit

We will look at how Django 2.0 does it.

Implementing ``__getitem__``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The code looks like this:

.. code:: python

    def __getitem__(self, k):
        """Retrieve an item or slice from the set of results."""
        if not isinstance(k, (int, slice)):
            raise TypeError
        assert ((not isinstance(k, slice) and (k >= 0)) or
                (isinstance(k, slice) and (k.start is None or k.start >= 0) and
                 (k.stop is None or k.stop >= 0))), \
            "Negative indexing is not supported."

        if self._result_cache is not None:
            return self._result_cache[k]

        if isinstance(k, slice):
            qs = self._chain()
            if k.start is not None:
                start = int(k.start)
            else:
                start = None
            if k.stop is not None:
                stop = int(k.stop)
            else:
                stop = None
            qs.query.set_limits(start, stop)
            return list(qs)[::k.step] if k.step else qs

There is a lot going on here, but each ``if`` block is straightforward.

-  In the first of block, we ensure slice has reaonable value.
-  In second block, if ``_result_cache`` is filled, aka the queryset has
   been evaluated, we return the slice from the cache and skip hitting
   the db again.
-  If the ``_result_cache`` is not filled, we
   ``qs.query.set_limits(start, stop)`` which sets the limit and offset
   in sql.

Implementing ``__iter__``
~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: python

    def __iter__(self):
        # ...
        self._fetch_all()
        return iter(self._result_cache)

Pretty strightforward, we populate the data then use builtin ``iter`` to
return an iterator.

It is also instructive to look at ``FlatValuesListIterable.__iter__``
which uses ``yield`` to implment ``__iter__``.

.. code:: python

    class FlatValuesListIterable(BaseIterable):
        """
        Iterable returned by QuerySet.values_list(flat=True) that yields single
        values.
        """

        def __iter__(self):
            queryset = self.queryset
            compiler = queryset.query.get_compiler(queryset.db)
            for row in compiler.results_iter(chunked_fetch=self.chunked_fetch, chunk_size=self.chunk_size):
                yield row[0]

Implementing ``__and__`` and ``__or__``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The code looks like this:

.. code:: python

    def __and__(self, other):
        self._merge_sanity_check(other)
        if isinstance(other, EmptyQuerySet):
            return other
        if isinstance(self, EmptyQuerySet):
            return self
        combined = self._chain()
        combined._merge_known_related_objects(other)
        combined.query.combine(other.query, sql.AND)
        return combined

We d some sanity checks on the querysets, return early if one of the
querysets is empty then apply SQL or using
``combined.query.combine(other.query, sql.AND)``. The ``__or__`` is
essentially same except the SQL is changed using
``combined.query.combine(other.query, sql.OR)``

Implementing ``__bool__``
~~~~~~~~~~~~~~~~~~~~~~~~~

The code looks like this:

.. code:: python


    def __bool__(self):
        self._fetch_all()
        return bool(self._result_cache)

Pretty straightforward, ``_fetch_all()`` ensures that the queryset is
evaluated, and ``_result_cache`` is filled. We then return the boolean
equivalent of ``_result_cache``, which means if there are any records,
you will get a ``True``.

Implementing ``__getstate__`` and ``__setstate__``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``__getstate__`` and ``__setstate__`` look like this:

.. code:: python

    def __getstate__(self):
        # Force the cache to be fully populated.
        self._fetch_all()
        return {**self.__dict__, DJANGO_VERSION_PICKLE_KEY: get_version()}

    def __setstate__(self, state):
        msg = None
        pickled_version = state.get(DJANGO_VERSION_PICKLE_KEY)
        if pickled_version:
            current_version = get_version()
            if current_version != pickled_version:
                msg = (
                    "Pickled queryset instance's Django version %s does not "
                    "match the current version %s." % (pickled_version, current_version)
                )
        else:
            msg = "Pickled queryset instance's Django version is not specified."

        if msg:
            warnings.warn(msg, RuntimeWarning, stacklevel=2)

        self.__dict__.update(state)

While pickling, we ensure data is populated, then use ``self.__dict__``
to get queryset representation, and return it along with Django version.
While unpickling, ``__setstate__`` ensures that a warning is raised when
pickled querysets are used across Django versions.

On a related note,
``{**self.__dict__, DJANGO_VERSION_PICKLE_KEY: get_version()}``, shows
why you should move to Python 3. This syntax for merging dictionaries
doesn't work in Python2.

Implementing ``__repr__``
~~~~~~~~~~~~~~~~~~~~~~~~~

The code for ``__repr__``, look like this

.. code:: python


    def __repr__(self):
        data = list(self[:REPR_OUTPUT_SIZE + 1])
        if len(data) > REPR_OUTPUT_SIZE:
            data[-1] = "...(remaining elements truncated)..."
        return '<%s %r>' % (self.__class__.__name__, data)

This is straightforward, but has a few nice tricks worth looking at.

``self[:REPR_OUTPUT_SIZE + 1]`` does slicing, which because we
implemented ``__getitem__``, does ``... limit ... offset ...`` query.

``REPR_OUTPUT_SIZE`` ensures that we don't pull in the wholeyset to
display data, but pulls up ``REPR_OUTPUT_SIZE + 1`` records. On next
line ``len(data) > REPR_OUTPUT_SIZE`` allows us the check if there were
more records without hitting the DB.

Final thoughts
~~~~~~~~~~~~~~

Magic, dunder methods provide a clean straightforward way to provide a
clean api to your classes. Unlike their name, they don't have any hidden
magic and should be used where it makes sense.
