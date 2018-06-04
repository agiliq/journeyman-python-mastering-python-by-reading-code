Understanding Python context managers by reading Django source code
------------------------------------------------------------------------


Django comes with a bunch of useful context managers. We will read their
source code to find what context managers can do and how to implement
them including some best parctices.

The three I use most are

-  ``transactions.atomic`` - To get a atomic transaction block
-  ``TestCase.settings`` - To change settings during a test run
-  ``connection.cursor`` - TO get a raw cursor

``connection.cursor`` Is generally implemented in the actual DB backends
such a psycopg2, so we will focus on ``transactions.atomic``,
``TestCase.settings`` and a few other contextmanagers.

What is a context manager?
~~~~~~~~~~~~~~~~~~~~~~~~~~

Context managers are a code patterns for

-  Step 1: Do something
-  Step 2: Do something else
-  Step 3: Final step, **this step must be guaranteed to run**.

For example when you say

.. code:: python


    with transaction.atomic():
        # This code executes inside a transaction.
        do_more_stuff()

What you really want is:

-  create a savepoint
-  ``do_more_stuff()``
-  Commit or rollback the savepoint

Similarly, when you say (Inside a ``django.test.TestCase``)

.. code:: python

    with self.settings(LOGIN_URL='/other/login/'):
        response = self.client.get('/sekrit/')

What you want is

-  Change ``settings`` to LOGIN\_URL='/other/login/'
-  ``response = self.client.get('/sekrit/')``, assert something with on
   response *with the changed setting*.
-  Change ``settings`` back to what existed at start.

A context manager povides a clean api to enforce this three step
workflow.

Some non-Django context managers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The most common context manager is

.. code:: python

    with open('alice-in-wonderland.txt', 'rw') as infile:
        line = infile.readlines()
        do_something_more()

If you did not have ``open`` contextmanager, you would need to do the
below everytime, because you need to ensure ``do_something_more()`` is
called.

.. code:: python

    try:
        infile = open('alice-in-wonderland.txt', 'r')
        line = infile.readlines()
        do_something_more()
    finally:
        infile.close()

Another common use is

.. code:: python

    a_lock = threading.Lock()

    with a_lock:
        do_something_more()

And without a context manager, this would have been.

.. code:: python

    a_lock.acquire()
    try:
        do_something_more()
    finally:
        a_lock.release()

So at a high level, **context managers are syntactic sugar for
``try: ... finally ...``** block. This is important, so I will repeat
**context managers are syntactic sugar for ``try: ... finally ...``**
block

Implementing context managers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Context managers can be implemented as a class with two required methods
and one optional ``__init__``

-  ``__enter__``: what to do when the context starts
-  ``__exit__``: what to do when the context ends
-  ``__init__``: if your context manager requires arguments

Alternatively, you can use ``contextlib.contextmanager`` with yield
statements to get a context manager. We will see an example in the next
section.

A simple Django context manager
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In ``django/tests/backends/mysql/tests.py``, Django implements a very
simple context manager.

.. code:: python

    @contextmanager
    def get_connection():
        new_connection = connection.copy()
        yield new_connection
        new_connection.close()

And then uses it like this:

.. code:: python

    def test_setting_isolation_level(self):
        with get_connection() as new_connection:
            new_connection.settings_dict['OPTIONS']['isolation_level'] = self.other_isolation_level
            self.assertEqual(
                self.get_isolation_level(new_connection),
                self.isolation_values[self.other_isolation_level]
            )

There is some code here which doesn't immediately concern us, let us
just focus on ``with get_connection() as new_connection:``

Using ``@contextmanager``, here is what happened:

-  The part before yield ``new_connection = connection.copy()`` handles
   the context setup.
-  The ``yield new_connection`` part allows using ``new_connection`` as
   ``as new_connection``.
-  The part after yield ``new_connection.close()`` handle context
   teardown.

Lets look at the ``TestCase.settings`` next, which uses the
``__enter__`` - ``__exit__`` protocol.

Implementing Testcase.settings
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Testcase.settings`` is implemented as

.. code:: python

    def settings(self, **kwargs):
        """
        A context manager that temporarily sets a setting and reverts to the
        original value when exiting the context.
        """
        return override_settings(**kwargs)

There is a bit of class hierarchy to jup through which takes us from

``Testcase.settings`` → ``override_settings`` → ``TestContextDecorator``

Skipping the part we don't care about, we get

.. code:: python

    class TestContextDecorator:
        # ...
        def enable(self):
            raise NotImplementedError

        def disable(self):
            raise NotImplementedError

        def __enter__(self):
            return self.enable()

        def __exit__(self, exc_type, exc_value, traceback):
            self.disable()

And then ``override_settings`` implements ``.enable`` and ``.disable``

.. code:: python

    class override_settings(TestContextDecorator):
        # ...
        def enable(self):
            # Keep this code at the beginning to leave the settings unchanged
            # in case it raises an exception because INSTALLED_APPS is invalid.
            if 'INSTALLED_APPS' in self.options:
                try:
                    apps.set_installed_apps(self.options['INSTALLED_APPS'])
                except Exception:
                    apps.unset_installed_apps()
                    raise
            override = UserSettingsHolder(settings._wrapped)
            for key, new_value in self.options.items():
                setattr(override, key, new_value)
            self.wrapped = settings._wrapped
            settings._wrapped = override
            for key, new_value in self.options.items():
                setting_changed.send(sender=settings._wrapped.__class__,
                                     setting=key, value=new_value, enter=True)

        def disable(self):
            if 'INSTALLED_APPS' in self.options:
                apps.unset_installed_apps()
            settings._wrapped = self.wrapped
            del self.wrapped
            for key in self.options:
                new_value = getattr(settings, key, None)
                setting_changed.send(sender=settings._wrapped.__class__,
                                     setting=key, value=new_value, enter=False)

There is a lot of boiler plate here which is interesting, but skipping
the state management we see

.. code:: python

    class override_settings(TestContextDecorator):
        # ...
        def enable(self):
            # ...
            # This gets called by __enter__
            for key, new_value in self.options.items():
                setattr(override, key, new_value)
            self.wrapped = settings._wrapped
            settings._wrapped = override
            for key, new_value in self.options.items():
                setting_changed.send(sender=settings._wrapped.__class__,
                                     setting=key, value=new_value, enter=True)

        def disable(self):
            # ...
            # This gets called by __exit__
            for key in self.options:
                new_value = getattr(settings, key, None)
                setting_changed.send(sender=settings._wrapped.__class__,
                                     setting=key, value=new_value, enter=False)

Implmenting context manager to also be used as a decorator.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When you can say ``with transaction.atomic():``, you can get the same
effect by using it as a decorator.

.. code:: python

    @transaction.atomic
    def do_something():
        # this must run in a transaction
        # ...

Implmenting a context manager to also be used as a decorator is a common
pattern and Django does the same with atomic.
``contextlib.ContextDecorator`` makes this straightforward.

.. code:: python


    # class Atomic is implemented later
    def atomic(using=None, savepoint=True):
        # Bare decorator: @atomic -- although the first argument is called
        # `using`, it's actually the function being decorated.
        if callable(using):
            return Atomic(DEFAULT_DB_ALIAS, savepoint)(using)
        # Decorator: @atomic(...) or context manager: with atomic(...): ...
        else:
            return Atomic(using, savepoint)

    class Atomic(ContextDecorator):
        # There is a lot of complicated corner cases and error handling.
        # See the gory details in django/django/db/transaction.py
        def __init__(self, using, savepoint):
            self.using = using
            self.savepoint = savepoint

        def __enter__(self):
            connection = get_connection(self.using)
            # ...
            # sid = connection.savepoint()
            # connection.savepoint_ids.append(sid)

        def __exit__(self, exc_type, exc_value, traceback):
            # Skip the gory details
            # ...
            sid = connection.savepoint_ids.pop()
            if sid is not None:
                try:
                    connection.savepoint_commit(sid)
                except DatabaseError:
                    connection.savepoint_rollback(sid)

Final thoughts
~~~~~~~~~~~~~~

Context managers provide a simple API for a powerfulo construct. Even
though they are merely syntactic sugar, they make for an itutive API and
in conjunction with the ``contextlib`` module are easy to implement.
