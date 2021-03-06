How to hack on HTTPolice
========================

Development environment
~~~~~~~~~~~~~~~~~~~~~~~
Set up::

  $ virtualenv /path/to/env
  $ source /path/to/env/bin/activate
  $ pip install -e .
  $ pip install -r tools/requirements.txt
  $ pip install ...    # any extra tools you like to have

Run tests::

  $ pytest

Run Pylint (currently doesn't work under Python 3.6)::

  $ tools/pylint_all.sh -j 2
  $ # ... or selectively:
  $ pylint httpolice/response.py

The delivery pipeline (Travis CI) enforces various other checks;
if you want to run them locally before pushing to GitHub, see ``.travis.yml``.

Use isort if you like -- there's an ``.isort.cfg`` with the right options --
but this is not enforced automatically for now.

Versions of development tools (pytest, Pylint...)
are pinned down to help make builds/QA reproducible.
From time to time, they are manually upgraded (see the "Maintenance" section).
Eventually I will use pip-sync for this,
but right now it is unusable due to `pip-tools issue #206`__.

__ https://github.com/nvie/pip-tools/issues/206


Codebase overview and caveats
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Rules and magic
---------------
At the core of HTTPolice are four long functions:

- ``httpolice.message.check_message``
- ``httpolice.request.check_request``
- ``httpolice.response.check_response_itself``
- ``httpolice.response.check_response_in_context``

Code in these functions looks a bit like rules in an expert system.
Most of these rules are ad-hoc (corresponding directly to text in the RFCs),
self-explanatory (by reference to the notice IDs they produce),
and independent of each other.
There is no need to split them into smaller functions.

The important thing about these rules is that they utilize a rich domain model
defined mostly in ``httpolice.structure`` and ``httpolice.headers``.
This provides magic, such as in ``Parametrized`` and ``HeaderView``,
that makes the rules look very natural and easy to follow.
However, **you have to know and keep in mind** how this magic actually works,
otherwise it's easy to introduce bugs.


``None`` and ``Unavailable``
----------------------------
Many things in the domain model can have the special values
``None`` and/or ``httpolice.structure.Unavailable``
in addition to the normal range of their types.
Usually:

- ``None`` means "we have no information about this thing";
- ``Unavailable`` means "we know that this thing is present
  (not missing), but we don't know its value".

For example:

- ``httpolice.Message.body`` can be:

  - a non-empty bytestring;
  - an empty bytestring, meaning that the body was absent or empty
    (RFC 7230 distinguishes between absent and empty, but we don't);
  - ``Unavailable``, meaning that the body was present but unknown
    (for instance, HAR files never contain the raw payload body,
    only the ``Message.decoded_body``);
  - ``None``, meaning that we have no idea if there was or wasn't a body.

- ``httpolice.known.method.is_idempotent`` can return:

  - ``True``, which means "definitely is";
  - ``False``, which means "definitely is not";
  - ``None``, which means "don't know"

  (this is why you see comparisons to ``False`` all over the place).


Code coverage
-------------
The delivery pipeline (Travis CI) requires "100%" statement coverage in tests.
This is *not* meant to enforce that every line has been tested,
but rather to ensure that no line has been *forgotten*.
Feel free to add ``pragma: no cover`` to code
that would be hard to cover with a natural, functional test.

Use ``pytest --no-cov`` to skip coverage checks when iterating on some code.


Typical workflows
~~~~~~~~~~~~~~~~~

Handling a new header
---------------------
Let's say RFC 9999 defines a new header called ``Foo-Bar``,
and you want HTTPolice to understand it.

#. Read RFC 9999.
#. Check for any updates and errata to RFC 9999
   (these are shown at the top of the page
   if you're using the HTML viewer at `tools.ietf.org`__).
   Note that not all errata are relevant: some may have been "rejected".
#. Rewrite the ABNF rules for ``Foo-Bar`` from RFC 9999
   into a new module ``httpolice.syntax.rfc9999``,
   using the parser combinators from ``httpolice.parse``.
   Consult other modules in ``httpolice.syntax`` to get the hang of it.
#. Add information about ``Foo-Bar`` into ``httpolice.known.header``.
   Consult the comments in that module regarding the fields you can fill in.
#. Some complex headers may need special-casing in ``httpolice.header``.
   See ``CacheControlView`` for an example.

__ https://tools.ietf.org/

All the basic checks for this header (no. 1000, 1063, etc.) should now work.


Adding a notice
---------------
#. Write your notice at the end of ``httpolice/notices.xml``.
   Let's say the last notice in HTTPolice has an ID of 1678,
   so your new notice becomes 1679.
#. In ``test/combined_data/``, copy ``simple_ok`` to ``1679_1``.
   For some notices, it's convenient to start with another file (like ``put``)
   or use HAR files instead (``test/har_data/``).
#. Change the ``1679_1`` file in such a way that it should trigger notice 1679.
#. Write "1679" at the top of that file
   to indicate the expected outcome of this test case.
   In HAR files, use the ``_expected`` key instead.
   You can also write comments there. Consult existing files.
#. If necessary, add more test cases: ``1679_2``, and so on.
#. Run your tests and make sure they fail as expected::

     $ pytest --no-cov -k1679

#. Write the actual checks logic.
   Usually it goes into one of the four big functions described above,
   but sometimes a better place is in ``httpolice.syntax`` (see e.g. no. 1015)
   or somewhere else.
#. Run the tests again and make sure they pass.
#. Check the report for your test cases
   to make sure the explanation looks good::

     $ httpolice -i combined -o html test/combined_data/1679* >/tmp/report.html
     $ open /tmp/report.html


Releasing a new version
-----------------------

#. Make sure that you're on master, it's clean and synced with GitHub,
   and that Travis is green.

#. Sync with IANA registries by running (under Python 2)::

     $ tools/iana.py

   and propagating any changes to ``httpolice.known``.

#. If necessary, update the version number in ``httpolice/__metadata__.py``
   (e.g. 0.12.0.dev4 → 0.12.0).

#. If releasing a "stable" version,
   replace the "Unreleased" heading in ``CHANGELOG.rst``
   with "<version> - <release date>", e.g. "0.12.0 - 2016-08-14".

#. Commit as necessary, for example::

     $ git commit -am 'Version 0.12.0'

#. Apply a Git tag equal to the version number, for example::

     $ git tag -a 0.12.0 -m 'Version 0.12.0'

#. Push master and tags::

     $ git push --tags origin master

#. Watch as Travis builds and uploads stuff to PyPI.

#. If releasing a "stable" version,
   check that Read the Docs has built it and updated the "stable" pointer.
   (You may need to refresh the page to see it.)
   If it hasn't, log in to readthedocs.org and force versions/builds manually.

#. Bump the version number in ``httpolice/__metadata__.py``
   (e.g. 0.12.0 → 0.13.0.dev1).

#. Commit and push::

     $ git commit -am 'Bump version to 0.13.0.dev1'
     $ git push


Maintenance
~~~~~~~~~~~

- Watch for new versions of related software
  and make sure they are compatible with HTTPolice:

  - main dependencies (``install_requires``);
  - sources of input data (tcpflow, major Web browsers, Fiddler).

- Update development dependencies:

  #. Review and update ``tools/requirements.in``.

  #. Pin down new versions::

       $ rm tools/requirements.txt
       $ pip-compile tools/requirements.in
       $ pip install -r tools/requirements.txt

  #. Update other pinned versions:

     - PyPy (in ``.travis.yml``);
     - Nu Html Checker (in ``tools/vnu.sh``);
     - JSHint (in ``tools/jshint.sh``).

  #. Check that everything is OK with the new versions.
     Maybe some Pylint overrides are no longer necessary, etc.

- Look at Travis build logs and make sure nothing strange is going on there.
- Check links in notices::

    $ linkchecker --check-extern extra_doc/_build/notices.html
