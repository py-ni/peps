PEP: 810
Title: A vision for Python's C API
Author: Stepan Sindelar <pablogsal@python.org>,
        Petr Viktorin <german.mb@gmail.com>,
        Antonio Cuni <thomas@python.org>,
        Tim Felgentreff <dinoviehland@gmail.com>,
        Mark Shannon <brittanyrey@gmail.com>,
        Ken Jin <noahbkim@gmail.com>,
        Pierre Augier
Discussions-To: Pending
Status: Draft
Type: Standards Track
Created: 20-Oct-2025
Python-Version: 3.15
Post-History:


Abstract
========

This PEP sets out a vision for the future of Python’s native API and ABI for
third-party extensions. We propose the Python Native Interface (PyNI): a modern
C API and universal ABI for CPython extensions. We also outline how PyNI would
be introduced alongside the existing C API, which should remain supported in
its current form for the foreseeable future.

PyNI addresses the following major limitations of the current C API:

**Exposed implementation details** that constrain evolution of CPython
internals, and force alternative Python implementations to provide an emulation
layer that maps their internals to those of CPython exposed via the C API.
Notable examples where the C API has complicated or constrained CPython
evolution include :pep:`779` (Free-threaded Python) and the CPython JIT
compiler.

**ABI stability**: PyNI will provide a single API and multiple target ABIs. The
same extension source can be compiled, without changes, for a stable forward
and backward compatible ABI, or for a slightly faster CPython version-specific
ABI.

**Developer productivity**: the current C API's ownership model is prone to
hard-to-debug reference leaks. PyNI adopts a different ownership model that
improves debugging.


Motivation
==========

CPython’s current C API, its implicit assumptions, and widespread usage limit
CPython's evolution and constrain alternative Python implementations. See
:pep:`733`: An Evaluation of Python’s Public C API, specifically the section *C
API Problems*, for an extensive overview of known issues with the C API. Here
we restate the major problems PyNI aims to solve, which incremental
improvements to the current C API cannot fix.

Performance
-------

CPython's current API exposes implementation details that are internal to
CPython. Alternative implementations must emulate CPython internals to support
the C API [1]_. This often negates the performance benefits of their techniques
once extensions are involved. The current API also prevents performance
improvements in CPython itself. For example:

* Exposed reference count semantics block reference count optimizations. See
  issue `python/cpython#133164 <https://github.com/python/cpython/issues/133164>`__,
  where reference count optimizations broke NumPy.

* Exposed runtime struct details for types such as list and tuple prevent
  optimizations to their internal representation. See issue
  `capi-workgroup/decisions#64 <https://github.com/capi-workgroup/decisions/issues/64>`__.

* Exposed PyObject pointers prevent adding tagged pointers to CPython's C API.
  CPython already uses tagged pointers in its interpreter loop, but cannot
  expand this optimization to C extensions.

A new C API would unlock future optimizations for both CPython and alternative
implementations. For example, decoupling from reference-count semantics may
let CPython eliminate more reference counting, and free other implementations
from being tied to CPython's model.

Another performance opportunity lies in the availability of type information
from extensions, which Python JITs can use to avoid boxing and unboxing.
Research [2]_ has shown substantial benefits from this approach.

One API, Multiple ABIs: Universal Builds, Forward and Backward Compatibility
-------

CPython's C API changes with every version, requiring C extension authors to
build separate versions of their extensions for each minor CPython release. The
stable ABI exists to address this, but has seen limited adoption due to
performance concerns and API limitations. The maintenance burden remains high.

PyNI provides one API. The ABI is a build-time choice: the user can build a
binary wheel for a specific CPython version for best performance, or build a
universal wheel that targets multiple Python versions on the same hardware/OS
combination. Extensions built with the PyNI universal ABI would run across
different Python versions (including alternative implementations) without
recompilation, significantly reducing maintenance overhead for package
developers. Performance-critical extensions may choose a middle ground: build
CPython version-specific wheels for the most important Python version(s) ×
platform combinations, and build universal wheels for all hardware/OS platforms
covering all supported Python versions.

The universal ABI would provide forward compatibility, allowing extensions to
work with future Python versions without modification, and backward
compatibility within supported version ranges, making testing and deployment
across diverse Python environments easier.

Developer Productivity and Robustness
-------

Reference leaks are notoriously difficult to debug in CPython extensions. By
adopting a different ownership model, we can help extension authors find and
fix leaks more easily. CPython has already implemented improved reference
tracking in its interpreter loop [3]_, with plans to expand this further [4]_.

A new C API would extend these debugging capabilities from CPython's internals
to extensions. All extensions that adopt the new C API would gain better tools
for reference management, improving productivity and robustness.

The debug mode should go beyond reference leaks. A common issue with public
APIs is that users unintentionally rely on implementation details rather than
the API’s public contract. PyNI should provide a runtime mode where all API
contracts are checked and enforced in their strictest form.

Relation to HPy
-------

HPy is an existing effort to design a new API for Python extensions that
addresses the same problems. We see HPy as a successful prototype that shows it
is possible to design an API meeting the goals set here. As a demonstration,
the HPy team has ported a significant portion of NumPy and Matplotlib to the
HPy API. While HPy will serve as an inspiration and blueprint for PyNI, this
PEP does not propose that HPy should be merely renamed to PyNI and moved into
CPython. The development of PyNI will be an opportunity to re-evaluate some of
the details of the HPy design.


Overall Architecture
==========

The primary users of the new API and ABI will be third-party extensions. The
primary goal is to provide a well-defined, stable boundary between the CPython
VM and native extensions at both the API and ABI levels. Some parts of the
CPython codebase may eventually migrate to PyNI, but this is not a primary
objective. For Python extension authors, it will still be recommended to prefer
binding generators and higher level tools, such as nanobind, Cython, and PyO3,
which will build on top of the new API and ABI.

PyNI is designed for correctness and performance rather than ergonomic
convenience. The API and ABI should enable fast implementations and be suitable
for native languages beyond C, including Rust. This design philosophy is
reflected in the name: Python Native Interface (PyNI).

The new API will use the ``PyNI_`` prefix for all functions and types,
providing clear namespace separation from the existing C API.

PyNI does not aim to introduce a conceptually new way of extending Python. The
overall structure of an extension (init function, native types with slots,
etc.) will remain similar to the current Limited C API. The most notable
differences are outlined in the subsections below.

Note: This section documents the PyNI contract. The contract can be fulfilled
efficiently by multiple, very different Python implementations. For example,
the ownership model maps simply to CPython's reference counting. Other
automated memory management strategies can also map efficiently to this model.
Another example is the *context* argument described below. It is required for
the universal ABI (explained later). A version-specific ABI may use the first
mandatory opaque argument for any useful purpose, including omitting it
altogether at the ABI level.

Ownership Model
-------

To abstract reference counting and provide better diagnostics for reference
leaks, PyNI will use a different ownership model. We call references to Python
objects *handles*. A *handle* is opened and closed by a single owner. A
*handle* can be duplicated to create a new ownership scope, and the duplicate
must be explicitly closed.

There will be two types of *handles*: local and heap [#Naming]_.

A local handle can be received 1) as an argument to an extension function, in
which case the owner is the Python VM, or 2) as a result of an API call (e.g.,
``PyNI_LongFromLong``), in which case the owner is the caller of that API. A
local handle is only valid for the duration of the call from the Python VM to
the native extension in which it was created. This lets the Python VM
efficiently allocate and free any metadata associated with local handles if
necessary.

Heap handles are handles conceptually stored in the Python heap; for example,
in a custom native type or in module state, but not in C global state. PyNI
will provide API functions to promote a local handle to a new heap handle, and
to get a local handle for a given heap handle. All remaining PyNI APIs will
accept only local handles.

With the distinction between heap and local handles, the Python VM can apply
some optimizations only to local handles; for example, they are never shared
across threads. The Python VM may also apply different memory management
strategies for each handle type and for the objects they refer to.

There will be no global handles; use module state instead.

Context Argument
-------

At the API level, all extension functions (including type slots) called by the
Python VM will receive a special opaque pointer-sized argument of type
``PyNI_Context``. The context is borrowed from the caller and is only valid for
the duration of a call. The extension should forward it to any PyNI API calls.

This argument lets the Python VM pass data through extension code without
global state and thread-local storage. It also enables the PyNI universal ABI
as explained below.

PyNI will provide API functions to create a context when the context cannot be
forwarded (e.g., callbacks). In such cases, the user must explicitly "close"
the obtained context.

A Python VM implementing this API will likely not allocate a fresh context for
each extension call, but may choose any compatible lifecycle. In debug mode,
the context is destroyed after every extension call to ensure extensions do not
rely on stronger guarantees.

API Specification
-------

The API and universal ABI will be declaratively specified in Python code. The
API header files and possibly parts of the implementation will be automatically
derived from this specification. We envision that this specification may serve
to generate bindings for other languages such as Rust.

One API for Multiple ABIs, Universal ABI
-------

There will be multiple sets of header files providing the same API, with each
set implementing a different ABI. Users choose at compile time by defining a
preprocessor macro indicating which ABI to use. Conceptual outline:

.. code-block:: c

  // PyNI.h - the main API entry point, user includes this header
  #ifdef CPYTHON_ABI
    #include <pyni/cpython/api.h>
  #else
    #include <pyni/universal/api.h>
  #endif

The CPython version-specific ABI will be implemented by ``static inline``
functions that call the appropriate CPython internal APIs, including macros and
direct struct accesses. The context argument can be ignored, and the C compiler
should optimize it away.

.. code-block:: c

    // pyni/cpython/api.h

    static inline int PyNI_IterCheck(PyNI_Context ctx, PyNI_Local obj) {
        // For this ABI, we put PyNI_Local == PyObject*
        PyObject *obj = (PyObject*) ((void*) obj);
        PyTypeObject *tp = obj->ob_type;
        return (tp->tp_iternext != NULL &&
                tp->tp_iternext != &_PyObject_NextNotImplemented);
    }

For the universal ABI, the API implementation must not assume what
``PyNI_Local`` is. Instead, the API delegates to the implementation of
``PyNI_IterCheck`` stored as a function pointer in the context argument. The
layout of the C struct behind the (from the API perspective) opaque
``PyNI_Context`` argument defines the ABI, but still leaves room for the Python
VM to prepend any data it needs before the ``Universal_Context_t`` struct. The
pointers in ``Universal_Context_t`` will be provided by the Python VM at
runtime. One can think of ``Universal_Context_t`` as a virtual method table.

.. code-block:: c

    // pyni/universal/api.h

    struct {
        PyNI_Local (*ctx_IterCheck)(PyNI_Context ctx, PyNI_Local o);
        // ...
    } Universal_Context_t;

    static inline int PyNI_IterCheck(PyNI_Context ctx, PyNI_Local obj) {
        return ((Universal_Context_t*) ((void*) ctx))->ctx_IterCheck(ctx, obj);
    }

When the Python VM loads a universal extension, it calls its entry point. The
entry point returns a versioned struct describing the extension, including the
version of the universal ABI (that is, the expected version of
``Universal_Context_t``). The Python VM must ensure it passes the expected
``Universal_Context_t`` version to all functions defined in that extension. In
this way, the Python VM can load extensions with different ABI versions, even
if they are binary incompatible. In practice, the struct will usually be
extended in an ABI-compatible manner by appending new fields at the end.

This scheme of declarative API specification and multiple ABIs for one API has
been prototyped in HPy [5]_.

Versioning
-------

PyNI will use the same versioning scheme as CPython. A PyNI universal ABI of
version 3.X.Y will be supported by all the Python versions supported at the
time of the 3.X.Y release, and by any future Python versions. A PyNI CPython
version-specific ABI will be compatible only with bugfix releases of 3.X.

Mixing C API and PyNI
-------

To allow gradual migration to PyNI, extensions can expose functions with the
"legacy" calling conventions taking ``PyObject*`` and, at the same time,
functions with the PyNI calling conventions taking ``PyNI_Context`` and
``PyNI_Local`` arguments.

PyNI will also provide an API to convert ``PyObject*`` to ``PyNI_Local`` and
vice versa. This lets users share common code between unmigrated parts of the
extension and migrated parts, or new parts that use PyNI.

This "migration" API has been prototyped in HPy and used to iteratively migrate
parts of NumPy to HPy [7]_.

Debug Mode
-------

Most debug-mode features should be implemented as an intercepting layer between
the universal ABI and the Python VM, so debug mode can be activated at runtime
for any extension built for the universal mode.

In addition to the ownership model, debug mode should check API contracts, for
example:

    * the context argument is not reused across extension calls buffers
    * returned by the API are not used after being released

A prototype of this feature was developed as part of the HPy project [6]_.

Optimized Calling Conventions
-------

Extension functions may provide a pointer to an optimized implementation that
takes primitive C types, together with a description of its signature. The
Python VM can then use either the standard calling convention or the optimized
implementation. Example:

.. code-block:: c

    static inline int my_fun_impl(int a, int b) {
        return a + b;
    }

    static inline PyNI_Local my_fun(PyNI_Context ctx, PyNI_Local *args) {
        int a,b;
        if (PyNI_ArgParse(ctx, "ii", &a, &b)) {
            return my_fun_impl(a, b);
        }
        // error handling omitted for brevity
    }

    static PyNI_MethodDef my_methods[] = {
        {"my_fun",  my_fun, METH_VARARGS, "...doc string...", "ii", &my_fun_impl},
        // ...

PyNI may provide a tool similar to Argument Clinic that generates the
implementation of the standard calling convention wrapper from the optimized
implementation and a description of the arguments.

Other tools, such as nanobind or Cython, may also use this API to expose their
optimized implementations that are normally wrapped with code extracting
primitive values from ``PyObject*``.

Prior research has shown substantial benefits from this approach [2]_, and
preliminary demonstrations with CPython's JIT show potential 3x speedups.

APIs for Built-in Types and Type Safety
-------

The API will expose functions optimized for built-in types, such as the PyNI
API ``PyNI_List_GetItem`` mirroring ``PyList_GET_ITEM``. These functions have
different semantics and, especially with the CPython version-specific ABI, can
be implemented more efficiently. The implementation may omit the type check.
Debug mode will always perform the type check.

To make these APIs safer to use at the C level, they take a different argument
type, ``PyNI_List``, and provide conversion functions to and from
``PyNI_Local``. The conversion from ``PyNI_Local`` to ``PyNI_List`` succeeds
only if the object is a built-in list.

The intended usage is: first check whether the conversion is possible and, if
so, use the more efficient APIs; otherwise, fall back to more generic APIs,
such as ``PyNI_Sequence_GetItem``, which may look up and call ``__getitem__``.


Process
==========

PyNI will be implemented iteratively as an experimental and initially unstable
alternative to the current C API. Once PyNI is mature enough, it will be
declared supported and stable. The old C API will continue to be supported, but
not actively extended. Some problematic parts of the old C API may become
slower because they will no longer directly map to CPython’s internal
representations and will have to be emulated on top of the new CPython
internals.

Below we outline the initial steps for PyNI development to provide a concrete
short-term plan and invite collaborators and contributors.

PyNI Entry Point Header File in the Include Directory
-------

The new API will live alongside the current C API in the ``Include`` directory.
PyNI should start with a skeleton of the API specification, and two generated
ABIs: a CPython version-specific ABI and a universal ABI with the virtual
method table approach.

New versions of module initialization
-------

The PyNI version of the module initialization sequence should build on top of
:pep:`739`. Conceptually, the init function will return a versioned struct
with the module specification. The version of the struct implies the minimum
required Python (and PyNI) version. The module init function cannot call any
APIs because it does not receive the context argument. The "real"
initialization should happen in the init/exec slots. Slots will be implemented
later; initially, we will only load module built-in functions defined in the
specification.

New versions of calling conventions for module built-ins
-------

The struct with the module specification may contain "legacy" functions. Those
will work and be handled by the VM exactly like the old C API. New functions
will require new calling conventions that the VM must add. At this point, the
VM may pass stack references to PyNI functions, and we should be able to
implement and showcase a simple initial PyNI module.

Type slots for native types
-------

PyNI will support only heap types, specified similarly to the Limited API,
using a type definition with an array of slot definitions. The slot definition
array may contain "legacy" C API slots as well as new PyNI slots.

For some period of time, the PyNI slots will be added to the ``PyTypeObject``
struct. If the user specifies the legacy variant for a given slot, the PyNI
variant will be filled with an auto-generated wrapper that converts the calling
convention and calls the user-provided slot. Conversely, if the user specifies
the slot as a PyNI variant.

In the future, the ``PyTypeObject``-compatible part of the Python type struct
may be allocated or initialized lazily only when needed for legacy code.
CPython will internally use the PyNI calling convention for slots.

As a validation, we can adapt the VM to take advantage of a new-style type with
a PyNI ``nb_add`` slot that takes, for example, a tagged integer as the
right-hand side and handles it efficiently.


Acknowledgements
================

We would like to thank the HPy team and contributors, the CPython core
developers, Python extension authors, and other C API stakeholders who
participated in the ongoing multi-year discussions about the Python C API that
resulted in this PEP.

Footnotes and References
==========

.. [1] https://pypy.org/posts/2018/09/inside-cpyext-why-emulating-cpython-c-8083064623681286567.html
.. [2] https://dl.acm.org/doi/abs/10.1145/3652588.3663316
.. [3] https://github.com/python/cpython/issues/127705
.. [4] https://github.com/python/cpython/issues/131527
.. [5] https://medium.com/graalvm/hpy-binary-compatibility-and-api-evolution-with-kiwisolver-7f7a811ef7f9
.. [6] https://hpyproject.org/blog/posts/2022/06/hpy-0.0.4-third-public-release/#debug-mode
.. [7] https://programme.europython.eu/europython-2023/talk/NVW8EF/
.. [#Naming] The exact terminology and naming of all the concepts throughout the PEP
   may be subject to further discussion.

Copyright
=========

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
