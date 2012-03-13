---
layout: post
title: "Cross-Python development with auto2to3"
date: 2012-03-13 09:00
comments: true
categories: Python
---
`2to3` can be a useful tool for developing Python libraries that work
with both Python 2 and 3, but it's fallen out of fashion because it
intrudes into the development process in awkward ways and generally
slows things down.  In spite of its weaknesses, I believe `2to3` is
a good approach for many libraries to support multiple versions of
Python, so in this post I'll share the approach and tools I used
to add Python 3 support to [Tornado][tornado]

## `2to3` vs single-source

The major alternative to `2to3` is to use a single source tree written
to the common subset of all supported versions of Python.  This is a
viable approach even for large projects, as shown by e.g. 
[Vinay Sajip's work with Django][django3k].  However, the resulting code
requires many [compatibility shims][singlesourceguide], and looks
somewhat unnatural from the perspective of both Python 2 and 3.  This
is especially true if compatibility with Python 2.5 is desired -- many
features that ease the transition to Python 3 were introduced in
version 2.6.  Personally, I prefer the workflow afforded by `2to3`, in
which the source code remains more-or-less normal Python 2, but it
works on Python 3 as well.

<!-- more -->

## Getting started

You'll need a good unit/regression test suite to ensure that things are
working as expected in Python 3.  In addition, your package should be
installable with a standard `setup.py` command.  Finally, of course,
check your third-party dependencies to ensure they are compatible with
Python 3.

You'll need the following tools:

* At least one build of Python 3.  Since the early versions of Python 3
  saw relatively little adoption, it's generally safe to skip 3.0 and 3.1
  and go straight to 3.2.  On Ubuntu you can install multiple Python packages
  from the [deadsnakes ppa][deadsnakes]; on a Mac try Homebrew or Macports.
* `2to3`: Included in recent versions of Python (as far back as 2.6,
  although we'll be using the version in 3.2)
* `virtualenv` and `pip`:  The de facto standards for managing multiple
  Python environments.
* `distribute`: The Python 3 successor of `setuptools`, necessary for
  running `2to3` automatically at install time.  `virtualenv` will
  install this automatically, so there's no need to download it separately.
* [tox][]: A virtualenv manager and test runner.
* [auto2to3][]: My own contribution to the Python 3 toolchain, `auto2to3`
  makes it easier and faster to work with `2to3`.  Note that `auto2to3`
  must be installed in a Python 3 environment; the other tools on this
  list will generally be installed under Python 2.

Setup procedure:

    virtualenv -p python2.7 ~/envs/py27
    virtualenv -p python3.2 ~/envs/py32
    ~/envs/py27/bin/pip install tox
    ~/envs/py32/bin/pip install auto2to3

## Iterating with `auto2to3`

`auto2to3` is an import hook that automatically runs `2to3` on demand.
The converted file is cached on disk so subsequent runs are faster.
Its command-line interface is similar to that of the Python
interpreter; it accepts both filenames and module names (with `-m`) to
specify the program to run.

For example, to run the Tornado test suite under both Python 2 and 3, do:

    ~/envs/py27/bin/python -m tornado.test.runtests
    ~/envs/py32/bin/python -m auto2to3 -m tornado.test.runtests

At this point, if you're lucky, you'll see a long list of failures (if you're
unlucky, some failure happened early enough that it prevented the rest of the
test suite from running).  Errors you're likely to encounter include:

* **Bytes and unicode don't mix...** The big change in Python 3 is that
  there is no longer an implicit conversion between byte strings and
  unicode strings.  Attempts to use one when the other is expected will
  usually result in `TypeErrors`.  Bugs of this type are often (but
  not necessarily) bugs in Python 2 as well, but would only manifest when
  non-ascii characters are used.
* **...Except when they do.**  There's an implicit conversion between bytes
  and `str`, because `str()` can convert any type to string.  This is
  an indirect conversion via `repr()`, and is unlikely to be what you want:

        $ python2.7 -c 'print str(b"foo")'
        foo
        $ python3.2 -c 'print(str(b"foo"))'
        b'foo'

* **`2to3` gets some things wrong.** For example, it assumes that
  calls to `.keys()` refer to the `dict` method (which changed to an
  iterator in Python 3) and wraps them in `list()`.  More esoteric issues
  include some problems with the three-argument form of the `raise`
  statement.  When you see these kinds of errors you may need to rework
  the code to run correctly both with and without `2to3`, or in some cases
  disable the relevant `2to3` fixer.

As an example, [this commit][tornado_py3_merge] (with 316 lines
changed) is the one that merged most of the Python&nbsp;3-related
changes to the Tornado master branch.

## Wrapping it up

Once your tests are passing with `2to3`, it's time to prepare the package for
distribution.  In `setup.py`, do something like this:

    import sys
    import distutils.core

    kwargs = {}
    if sys.version_info[0] >= 3:
        import setuptools  # setuptools (aka distribute) is required for use_2to3
        kwargs["use_2to3"] = True

    distutils.core.setup(
        ...
        **kwargs)

This will run `2to3` automatically at installation time, so everything should
just work for people installing your package under Python 3.

Once the initial port to Python 3 is done, it's often more convenient
to treat Python 3 just like any other Python version.  This can be
done by running tests under `tox` before each commit.  It's slower
than `auto2to3` since it doesn't cache the converted output, but it's
a more realistic simulation of real-world installations.  Create a
file `tox.ini` (in the same directory as `setup.py`) that looks
something like this:

    [tox]
    envlist = py27, py32

    [testenv]
    # Change these variables as needed.
    commands = python -m tornado.test.runtests
    deps = pycurl

    # python will import relative to the current working directory by default,
    # so cd into the tox working directory to avoid picking up the working
    # copy of the files (especially important for 2to3).
    changedir = {toxworkdir}

Now run `~/envs/py27/bin/tox` and it will run your tests under both Python
2.7 and 3.2.

Once you've tested everything enough to be confident in a new release,
just upload a new build to PyPI and it will be installable via
`pip` in both 2 and 3.

## The future

Once the initial work of porting is done, I've found it to be fairly
simple to maintain compatibility as development continues - I work
mainly in Python 2.7, and test the other versions via `tox`.  The most
common error that is caught in my Python 3 test runs is forgetting to
mark the ascii string literals I use in unit tests as byte strings,
which is easily remedied.

`2to3` is of course a transitional tool, and at some point you'll want to
switch to working mainly in Python 3.  It should be possible to adapt
this workflow to go in the other direction with `3to2`, but I haven't
pursued this approach yet mainly due to the lack of standardized support
for running `3to2` automatically in `setuptools`/`distribute`.


[tornado]: http://www.tornadoweb.org
[django3k]: https://groups.google.com/forum/#!topic/django-developers/XjrX3FIPT-U
[singlesourceguide]: https://code.djangoproject.com/wiki/PortingNotesFor2To3
[deadsnakes]: https://launchpad.net/~fkrull/+archive/deadsnakes
[tox]: http://tox.readthedocs.org/en/latest/index.html
[auto2to3]: https://github.com/bdarnell/auto2to3
[tornado_py3_merge]: https://github.com/facebook/tornado/commit/3b3534c2436326328620fbea2626a30ea5e79f13
