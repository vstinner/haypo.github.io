+++++++++++++++++++++++++++++++++++++++++++++++++
Hide implementation details from the Python C API
+++++++++++++++++++++++++++++++++++++++++++++++++

:date: 2020-12-25 22:00
:tags: optimization, cpython, c-api
:category: cpython
:slug: hide-implementation-details-python-c-api
:authors: Victor Stinner

.. image:: {static}/images/pepsie.jpg
   :alt: My cat attacking the Python C API

This article is the history of Python C API discussions over the last 4 years,
and the creation of C API projects: `pythoncapi website
<https://pythoncapi.readthedocs.io/>`_, `pythoncapi_compat.h header file
<https://github.com/pythoncapi/pythoncapi_compat>`_ and `HPy (new clean C API)
<https://hpy.readthedocs.io/>`_. More and more people are aware of issues
caused by the C API and are working on solutions.

It took me a lot of iterations to find the right approach to evolve the C API
without breaking too many third-party extension modules. My first ideas were
based on two APIs with an opt-in option somehow. At the end, I decided to fix
directly the default API, and helped maintainers of extension modules to update
their projects for incompatible C API changes.

I wrote a ``pythoncapi_compat.h`` header file which adds C API functions of
newer Python to old Python versions up to Python 2.7. I also wrote a
``upgrade_pythoncapi.py`` script to add Python 3.10 support to an extension
module without losing Python 2.7 support: the tool adds ``#include
"pythoncapi_compat.h"``. For example, it replaces ``Py_TYPE(obj) = type``
with ``Py_SET_SIZE(obj, type)``.

The photo: my cat attacking the Python C API.

Year 2016
=========

Between 2016 and 2017, Larry Hastings worked on removing the GIL in a CPython
fork called "The Gilectomy". He pushed the first commit in April 2016: `Removed
the GIL. Don't merge this!
<https://github.com/larryhastings/gilectomy/commit/4a1a4ff49e34b9705608cad968f467af161dcf02>`_
("Few programs work now"). At EuroPython 2016, he gave the talk `Larry Hastings
- The Gilectomy <https://www.youtube.com/watch?v=fgWUwQVoLHo>`_ where he
explains that the current parallelism bottleneck is the CPython reference
counting which doesn't scale with the number of threads.

It was just another hint telling me that "something" should be done to make the
C API more abstract, move away from implementation details like reference
counting. PyPy also has performance issues with the C API for many years.


Year 2017
=========

May
---

In 2017, I discussed with Eric Snow who was working on subinterpreters. He had
to modify public structures, especially the ``PyInterpreterState`` structure.
He created ``Include/internal/`` subdirectory to create a new "internal C API"
which should not be exported. (Later, he moved the ``PyInterpreterState``
structure to the internal C API in Python 3.8.)

I started the discuss C API changes during the Python Language Summit
(PyCon US 2017): `"Python performance" slides (PDF)
<https://github.com/vstinner/conf/raw/master/2017-PyconUS/summit.pdf>`_:

* Split Include in sub-directories
* Move towards a stable ABI by default

See also the LWN article: `Keeping Python competitive
<https://lwn.net/Articles/723752/#723949>`_ by Jake Edge.

July: first PEP draft
---------------------

I proposed the first PEP draft to python-ideas:
`PEP: Hide implementation details in the C API
<https://mail.python.org/archives/list/python-ideas@python.org/thread/6XATDGWK4VBUQPRHCRLKQECTJIPBVNJQ/>`__.

The idea is to add an opt-in option to distutils to build an extension module
with a new C API, remove implementation details from the new C API, and maybe
later switch to the new C API by default.

September
---------

I discussed my C API change ideas at the CPython core dev sprint (at Instagram,
California).  The ideas were liked by most (if not all) core developers who are
fine with a minor performance slowdown (caused by replacing macros with
function calls). I wrote `A New C API for CPython
<https://vstinner.github.io/new-python-c-api.html>`_ blog post about these
discussions.

November
--------

I proposed `Make the stable API-ABI usable
<https://mail.python.org/pipermail/python-dev/2017-November/150607.html>`_ on
the python-dev list. The idea is to add ``PyTuple_GET_ITEM()`` (for example) to
the limited C API but declared as a function call. Later, if enough extension
modules are compatible with the extended limited C API, make it the default.

Year 2018
=========

In July, I created the `pythoncapi website
<https://pythoncapi.readthedocs.io/>`_ to collect issues of the current C
API, list things to avoid in new functions like borrowed references, and start
to design a new better C API.

In September, Antonio Cuni wrote `Inside cpyext: Why emulating CPython C API is
so Hard
<https://morepypy.blogspot.com/2018/09/inside-cpyext-why-emulating-cpython-c.html>`_
article.

Year 2019
=========

In February, I sent `Update on CPython header files reorganization
<https://mail.python.org/archives/list/capi-sig@python.org/thread/WS6ATJWRUQZESGGYP3CCSVPF7OMPMNM6/>`_
to the capi-sig list.

* ``Include/``: limited C API
* ``Include/cpython/``: CPython C API
* ``Include/internal/``: CPython internal C API

In March, I modified the Python debug build to make its ABI compatible with the
release build ABI:
`What’s New In Python 3.8: Debug build uses the same ABI as release build
<https://docs.python.org/dev/whatsnew/3.8.html#debug-build-uses-the-same-abi-as-release-build>`_.

In May, I gave a lightning talk `Status of the stable API and ABI in Python 3.8
<https://github.com/vstinner/conf/blob/master/2019-Pycon/status_stable_api_abi.pdf>`_,
at the Language Summit (during Pycon US 2019):

* Convert macros to static inline functions
* Install the internal C API
* Debug build now ABI compatible with the release build ABI
* Getting rid of global variables

By the way, see my `Split Include/ directory in Python 3.8
<{filename}/split_include_python38.rst>`_ article: I converted many macros in
Python 3.8.

In July, the `HPy project <https://hpy.readthedocs.io/>`_ was created during
EuroPython at Basel. There was an informal meeting which included core
developers of PyPy (Antonio, Armin and Ronan), CPython (Victor Stinner and Mark
Shannon) and Cython (Stefan Behnel).

In December, Antonio, Armin and Ronan had a small internal sprint to kick-off
the development of HPy: `HPy kick-off sprint report
<https://morepypy.blogspot.com/2019/12/hpy-kick-off-sprint-report.html>`_


Year 2020
=========

April
-----

I proposed `PEP: Modify the C API to hide implementation details
<https://mail.python.org/archives/list/python-dev@python.org/thread/HKM774XKU7DPJNLUTYHUB5U6VR6EQMJF/#TKHNENOXP6H34E73XGFOL2KKXSM4Z6T2>`__
on the python-dev list. The main idea is to provide a new optimized Python
runtime which is backward incompatible on purpose, and continue to ship the
regular runtime which is fully backward compatible.

June
----

I wrote `PEP 620 -- Hide implementation details from the C API
<https://www.python.org/dev/peps/pep-0620/>`_ and `proposed the PEP to
python-dev
<https://mail.python.org/archives/list/python-dev@python.org/thread/HKM774XKU7DPJNLUTYHUB5U6VR6EQMJF/>`_.
This PEP is my 3rd attempt to fix the C API: I rewrote it from scratch. Python
now distributes a new ``pythoncapi_compat.h`` header and a process is defined
to reduce the number of broken C extensions when introducing C API incompatible
changes listed in this PEP.

I created the `pythoncapi_compat project
<https://github.com/pythoncapi/pythoncapi_compat>`_: header file providing new
C API functions to old Python versions using static inline functions.

December
--------

I wrote a new ``upgrade_pythoncapi.py`` script to add Python 3.10
support to an extension module without losing support with Python 2.7.  I sent
`New script: add Python 3.10 support to your C extensions without losing Python
3.6 support
<https://mail.python.org/archives/list/capi-sig@python.org/thread/LFLXFMKMZ77UCDUFD5EQCONSAFFWJWOZ/>`_
to the capi-sig list.

The pythoncapi_compat project got its first users (bitarray, immutables,
python-zstandard)! It proves that the project is useful and needed.

I collaborated with the HPy project to create a manifesto explaining how the C
API prevents to optimize CPython and makes the CPython C API inefficient on
PyPy. It is still a draft.
