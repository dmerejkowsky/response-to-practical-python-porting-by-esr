Response to ERS' "Pratical Python porting for system programmers"
=================================================================

This text is a response to Eris S. Raymond and
Peter A. Donis HOWTO on Python2 to Python3 porting
for system programmers.

Go read the original text
`here <http://www.catb.org/esr/faqs/practical-python-porting/>`_ first.

Introduction
------------

I'm going to talk here about my own porting experience on the project
I've been working on for several years now.

It's a build framework written in Python called ``qiBuild``.

See the `documentation <http://doc.aldebaran.com/qibuild>`_, the
`github repo <https://github.com/aldebaran/qibuild>`_ and the
`demo on asciinema <https://asciinema.org/a/35360>`_ for more details.

The gist of it is that this is a program that reads XML config files,
and runs some commands like `git` or `cmake` to fetch sources, configure
and build some complex `C++` code, so the scope is a bit different
than the type of programs (written by "system programmer") that ESR is
talking about in his HOWTO.

Main differences are:

* We don't care about performance (the bottlenecks are the network when
  we do git stuff, and forking other executables when we build stuff)

* We don't really care if bytes in the range 0x80..0xFF are modified,
  because we rarely parse binary data and, as stated above, we don't
  mind the overhead of encoding and decoding strings to bytes.

* The tests are written in Python using `pytest <http://pytest.org/latest/>`_
  and have a lot of dependencies on external packages. (``reposurgeon`` and
  ``src`` are not using tests written in Python, and have no dependencies
  outside the stdlib)

There are some similarities, though:

* The software has a large suite of tests (85% line coverage for `qiBuild`)

* The main goal is the same : we want the code to work both under Python3 and
  Python2.7


Where I agree
-------------

The "Why is this difficult" section is a good introduction to the
real problems that occurs when Porting to Python3.

The "What doesn't work" also contains solid advice.

"Make you change testable": I can't stress this enough. Without a rigorous
test suite, that checks behavior on input containing non-ASCII characters,
you are going to be in a lot of trouble when trying to port to Python3,
and will be faced with mysterious ``UnicodeDecodeError`` or
``can't concat bytes to string`` errors.

"Fix up string/unicode mixing"::

> The art here is in doing as little work as possible. Your encode() and
> decode() calls should intercept your binary I/O close to where it happens, so
> the bulk of your code is just seeing Unicode strings.
> This is also the stage at which you may need to tag some literals with a
> prefix b for byte-buffer. Beware, if you have a lot of these it may mean you
> have not put encode/decode calls near enough to the natural choke points where
> your binary I/O is happening.

All good advice. In ``qiBuild`` for instance we have a method to read output
of ``git`` commands (``Git.call()``). Data is read from the ``subprocess.Popen``
object as bytes-buffer and is immediately encoded as UTF-8 string
(Yes, UTF-8 and not Latin-1, more on this later)


Missing points
--------------

Here are some points that are not at all covered by ESR's HOWTO, but that
I still find useful:

Don't target Python between 3.0 and 3.2
++++++++++++++++++++++++++++++++++++++++

You won't be able to use ``u"foo"`` to prefix your Unicode string litterals,
(among other things) which really is a PITA.

The only case you may want it is for old distributions, such as ``Ubuntu
12.04``, but users of these systems can still use ``Python2.7`` since
you're writing Python2 compatible code.

Of course, when dropping Python2 support you should also drop support for
these old distributions too.

Fixing print
+++++++++++++

I find it odd that ESR's howto does not mention it, it's *the* most
known problem when switching to Python3

If you use ``2to3``, code will be converted from:

.. code-block::

    print foo, bar

to

.. code-block::

    print(foo, bar)

The problem is that for Python2, this statement means "Print the tuple
(foo, bar)", so the result is not what you expect.

The fix is simple, just add

.. code-block::

    from __future__ import print_function

before any of your imports

Fixing division
+++++++++++++++

ESR does not talk about the case when you really want a float
result, even when both arguments are ints.

There's a way to fix this too.

Use:

.. code-block:: python

    from __future__ import division

This makes it possible to avoid using things like:

.. code-block:: python

    i_really_need_a_float = float(a) / b

Of course, if you really need truncating divison, use `//`

Fixing exceptions testing
+++++++++++++++++++++++++++

In ``qiBuild`` I use a lot of exceptions, and thus a lot of tests
are checking exceptions for their message.

When you have a exception derived from the basic ``Exception`` class,
you should make sure when porting to Python3, to not use the
``message`` member, but the ``args`` member:

.. code-block:: python

    # Fails on Python3: Exception has no attribute named 'message'
    with pytest.raises(MyException) as e:
        test_something_that_should_throw()
    assert "something" in e.message


.. code-block:: python

    # Works both for Python3 and Python2

    with pytest.raises(MyException) as e:
        test_something_that_should_throw()
    assert "something" in e.args[0]

A note on continuous integration
+++++++++++++++++++++++++++++++++

On both ``reposurgeon`` and ``src``, the port to Python3 was done while no
other development was done. On ``qiBuild``, the development continued without
waiting for the Python3 port to be over and merged, so the port had
to be done on an other branch. (I called it 'six')

So, how to cope with that?

Well, use continuous integration. In my case I'm using Jenkins.

Whenever a commit is merged on the development branch, the following
happens:

* The 'six' branch gets rebased

* The test suite is ran both for Python2 and Python3

* The branch gets "pushed forced" to the main repository.

If any of this steps go wrong (for instance, the rebase failed because of
conflicts, or one of the test suite failed), a mail is sent and
appropriate action can be taken.

This means the 'six' branch continues to be "alive" and can be trivially and
safely merged to the development branch when ready.


Where I don't agree
-------------------

Come on, I know this is the part you've all been waiting for :)

A little disclaimer first.

These are my own opinions, and your mileage may vary. I'm not saying
that ESR is wrong, I'm just offering an other point of view on a topic
I care about, based on my own experience on a somewhat similar project.

About the steps
++++++++++++++++

Here are the steps ESR recommends:

#. Run ``2to3`` and apply the patch it generates
#. Partially revert it to make sure it still runs under Python2
#. Change the shebangs to be ``#!/usr/bin/env python3``
#. Fix Python3 issues
#. Change the shebangs again to be ``#!/usr/bin/env python``
#. Tweak the test suite to run twice, once for Python2 and once for Python3

The steps I've followed are a bit different:

#. Run ``2to3`` and apply the patch it generates (no changes here)
#. Make the whole test suite pass on Python3
#. Then make the whole test suite pass on Python2. But this time,
   instead of manually writing compatibly code, I used the excellent
   `six <http://pythonhosted.org/six/>`_ library.
   (More on ``six`` later)
#. When Python2 test suite passes again, check with Python3
#. Last step is the same: make sure the test suite runs twice, once for
   Python2 and once for Python3. I recommend using
   `tox <https://testrun.org/tox/latest/>`_ for this, especially if you are
   using Jenkins to run your test suite.

Note that if we wish to drop Python2 compatibily, all we have to do is revert
the patch that uses ``six``

Also, there's no need to manually amend the patch generated by ``2to3``,
which means it's easy to redo the port once the changes are rebased
(see above)

About the porting itself
+++++++++++++++++++++++++

Why ``six`` ?
~~~~~~~~~~~~~~


``reposurgeon`` and ``src`` do not use ``six`` to help Python3 porting,
probably because the author did not want to depend on anything other than
the stdlib.

In ``qiBuild`` we already depend on third-party libraries, so adding an
other one was no being deal.

Also, ``six`` is the choice for a lot of projects that wish to achieve
Python2/Python3 compatibility with the same code base (Sphinx and Django, to
only name a few)

I also thinks that using ``six`` leads to cleaner code.

*  It takes care of libraries whose name changed, so you can write

   .. code-block:: python

      from six.moves import input

   and then use ``input`` everywhere, instead of

   .. code-block:: python

       input = raw_input
       except NameError:
           my_input = input

   which looks like a hack to me.

*  Same thing for import changes:

   .. code-block:: python

     from six.move import configparser

   Instead of:

   .. code-block:: python

       try:
           import configparser
       except ImportError:
           import ConfigParser as configparser

* Lastly, it's the best way I know to handle code that use
  metaclasses while keeping a syntax compatible with Python2 and Python3

.. note:: `pies <https://github.com/timothycrosley/pies>`_ is an alternative to
          ``six`` you may want to consider. I did not use it so I can't really
          comment on it. It seems to be less popular than six, though.

.. TODO: talk about python-future

Use ``UTF-8`` everywhere
~~~~~~~~~~~~~~~~~~~~~~~~~

I chose to always encode in UTF-8 instead of Latin-1.

Rationale:

* UTF-8 has become the 'standard' when it comes to encoding, and can handle
  things than Latin-1 can't.

* We do a lot of XML parsing and writing, and UTF-8 is the default encoding
  for XML

* As stated above, we don't care about the high-byte-preserving stuff since
  we don't write binary data.


Re-assigning ``sys.stdout`` and ``sys.stderr``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I also don't recommend the trick that re-assigns ``sys.stdout`` and
``sys.stdin`` to use ``io.TextWrapper`` instead. Instead, make sure that
your string is UTF-8 encoded before sending it to ``sys.stdout`` or
``sys.stderr``.

If you have to mock ``sys.stdout`` in your tests, do something like:

.. TODO: add examples fro qibuild's six branch


Messing with shebangs
~~~~~~~~~~~~~~~~~~~~~

There is a better way, that may seem overkill for single-file projects like
``reposurgeon`` or ``src``, but is quite handy for a project like ``qiBuild``
which has a bunch of command-line scripts (``qibuild``, ``qisrc``, and so on)

*  Write a setup.py and declare an entry point (Usually a ``main`` method from
   one of your modules):

  .. code-block:: python

    # setup.py

    from setuptools import setup
    # Yes, you need setuptools and not distutils

    setup(
        name = "foo",
        py_modules=["foo"],
        entry_points = {
          "console_scripts" : [
              "foo = "foo:main",
          ]
        }
    )

*  Create two virtualenvs, one for each version of Python

  .. code-block:: bash

      mkdir -p ~/.venvs
      virtualenv-2 ~/.venvs/foo-py2
      virtualenv-3 ~/.venvs/foo-py3

*  Then in both env, run ``pip install --editable .`` from the sources
   of your porject:


    .. code-block:: bash

        source ~/.venvs/foo-py2/bin/activate
        pip install --editable .
        deactivate # exit the virtualenv for Python2
        source ~/.venvs/foo-py3/bin/activate
        pip install --editable .


Done. ``setuptools`` will generate a ``foo`` script with the correct
shebang in both virtualenvs that gets inserted into your ``PATH``
when you switch virtualenvs when sourcing the ``activate`` script.

For extra convenience you can use `virtualenvwrapper
<https://virtualenvwrapper.readthedocs.org/en/latest/>`_ to quickly
switch from one virtualenv to an other.

Messing with xrange, imap, izp
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. TODO (just use the Python3 version, we don't care about the loss of
   perfs in Python2)

Conclusion
-----------

Thanks to ESR for giving me the idea of writing my own porting guide,
it was a fun exercise.

I've left a comment in his blog post, discussion can continues on his blog.

If you are curious, the ``six`` branch is available on
`my personal fork on github <https://github.com/dmerejkowsky/qibuild/commits/six>`_,
but please don't use it as history on this branch is frequently rewritten.

Also, note that there is just one big commit where all the porting happens.

Initially there was one per step, but it's more convenient to
have them squashed when rebasing.
