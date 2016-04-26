:tocdepth: 1

All LSST code currently uses `SWIG <http://www.SWIG.org>`_ to generate Python wrappers around C++ code. This document investigates using `pybind11 <http://pybind11.readthedocs.org/en/latest/index.html>`_ as an alternative.
To start the investigation Jim Bosch has written a `C++/Python Bindings Challenge <https://github.com/lsst-dm/python-cpp-challenge>`_.
It consists of "a small suite of C++ classes designed to highlight any of the most common challenges involved in providing Python bindings ot a C++ library, as well as a set of Python unit tests that attempt to measure the quality of the resulting bindings".
Concretely, this document then describes the (partial) pybind11 solution to this challenge.

About the scope of this document
================================

Although this document contains some examples of how to wrap C++ with pybind11 and might help to get you started it is not meant to be a full tutorial by any means. The examples are just there to get some context when describing the various things that I encountered. If you are looking for comprehensive information about pybind11 please have a look at some of the documents linked below instead.

About pybind11
==============

pybind11 is a header-only library to create Python bindings for C++ code.
Compared to other tools it is extremely lightweight and requires only a modern C++ compiler
with support for C++11.
In concept is quite similar to `Boost Python <www.boost.org/libs/python/doc>`_, in that it
minimizes the boilerplate involved with writing a Python extension module by inferring type
information using compile-time introspection.
In contrast to Boost Python it doesn't need to be compiled and is actively developed
(albeit by a very small group of developers).

The software is available under the terms of a BSD license.

For more information about pybind11 and supported features see the `documentation <http://pybind11.readthedocs.org/en/latest/index.html>`_

A quick example
===============

Given a simple C++ class:

.. code-block:: cpp

    #include <string>
    
    namespace basics {
    
    class Doodad {
    public:
        Doodad(std::string const & name_, int value_) : name{name_}, value{value_} {};
            
        std::string name;
        int value;
    };
        
    } // namespace basics

a basic pybind11 wrapper for this would look like this.

.. code-block:: cpp

    #include <pybind11/pybind11.h>
    
    #include "basics.h"
    
    namespace py = pybind11;
    
    PYBIND11_PLUGIN(basics) {
        py::module m("basics", "pybind11 basics module");
    
        py::class_<basics::Doodad>(m, "Doodad")
            .def(py::init<const std::string &, int>())
            .def_readwrite("name", &basics::Doodad::name)
            .def_readwrite("value", &basics::Doodad::value);
    
        return m.ptr();
    }

This can then be compiled into a standard (C)Python extension module with the following ``setup.py`` file.
Note that, since pybind11 is just C++11 (or later) any compiler and build system can be used.

.. code-block:: python

    import os, sys
    
    from distutils.core import setup, Extension
    from distutils import sysconfig
    
    cpp_args = ['-std=c++11', '-stdlib=libc++', '-mmacosx-version-min=10.7']
    
    ext_modules = [
        Extension(
    	'basics',
            ['basics.cpp'],
            include_dirs=['pybind11/include'],
    	language='c++',
    	extra_compile_args = cpp_args,
        ),
    ]
    
    setup(
        name='example',
        version='0.0.1',
        author='Pim Schellart',
        author_email='P.Schellart@princeton.edu',
        description='Example',
        ext_modules=ext_modules,
    )

Which can be built with:

.. code-block:: bash

    python setup.py build_ext --inplace

Quick example step-by-step
--------------------------

Now let's examine the wrapper code step by step.

As discussed before pybind11 is a header-only library. Moreover, most of the functionality is contained
in a single header.

.. code-block:: cpp

    #include <pybind11/pybind11.h>
    
Next we include the header file of the to-be-wrapped class (which in this case also includes the class definition,
but that is of course not necessary one can simply link).

.. code-block:: cpp

    #include "basics.h"
    
All of pybind11 lives in its own namespace.

.. code-block:: cpp

    namespace py = pybind11;
    
The ``PYBIND11_PLUGIN()`` macro creates a function that will be called when an import statement is issued from within Python.

.. code-block:: cpp

    PYBIND11_PLUGIN(basics) {
        ...

Then a module is created, with a docstring.

.. code-block:: cpp

        py::module m("basics", "pybind11 basics module");
    

Followed by a new extension type (the wrapper for ``Doodad``).

.. code-block:: cpp

        py::class_<basics::Doodad>(m, "Doodad")

The ``.def`` methods create binding code that expose C++ functions to Python.
The special ``init`` template is for constructors.

.. code-block:: cpp

            .def(py::init<const std::string &, int>())

Properties are directly exposed with ease.

.. code-block:: cpp

            .def_readwrite("name", &basics::Doodad::name)
            .def_readwrite("value", &basics::Doodad::value);
    
Finally a pointer to the module object is returned to the Python interpreter.

.. code-block:: cpp

        ...
        return m.ptr();
    }

Solving the C++/Python bindings challenge with pybind11
=======================================================

The previous section gave a quick overview of wrapping a C++ class with pybind11. This section describes some of the issues encountered while wrapping the C++/Python bindings challenge code. This code was designed to be more representative of the type of code encountered when porting larger swaths of LSST library code.

It contains four C++ source files which are to be compiled into three different Python modules (with interdependencies).

* ``basics`` contains a class ``Doodad``, a class ``Secret`` and a struct ``WhatsIt``. The class ``WhatsIt`` should be visible to Python only as a tuple and ``Secret`` can only be constructed by ``Doodad``, it is to be passed around in Python as an opaque object. ``Doodad`` is the main class to be wrapped.

* ``extensions`` contains a templated class ``Thingamajig`` that inherits from ``Doodad``.

* ``containers`` defines a ``DoodadSet`` class (backed by a ``std::vector``) that contains ``Doodads`` and ``Thingamajigs``.

* ``conversions`` contains various tests for SWIG compatibility (more on this later).

The current solution does not (yet) wrap ``extensions`` due to time constraints. For the other modules this is the status of the unit tests:

* ``basics``, passes all unit tests except for const correctness;
* ``containers``, passes all unit tests except for inheritance (because ``extensions`` is not implemented);
* ``converters``, SWIG -> pybind11 and pybind11 -> SWIG work, but do not preserve const correctness.

Diving in
---------

Because the final pybind11 solution requires so little code (in contrast to Cython) the full solutions to
individual components is presented, each followed by a detailed breakdown of the issues encountered.

Basics
^^^^^^

The full wrapper of the ``basics`` module (excluding comments) is.

.. code-block:: cpp

    #include <pybind11/pybind11.h>
    
    #include "basics.hpp"
    
    namespace py = pybind11;
    
    PYBIND11_DECLARE_HOLDER_TYPE(T, std::shared_ptr<T>);
    
    PYBIND11_PLUGIN(basics) {
        py::module m("basics", "wrapped C++ basics module");
    
        m.def("compare", &basics::compare);
        m.def("adjacent", &basics::adjacent);
    
        py::class_<basics::Secret>(m, "Secret");
    
        py::class_<basics::Doodad, std::shared_ptr<basics::Doodad>>(m, "Doodad")
            .def(py::init<const std::string &, int>(), py::arg("name"), py::arg("value") = 1)
            .def("__init__",
                [](basics::Doodad &instance, std::pair<std::string, int> p) {
                    new (&instance) basics::Doodad(basics::WhatsIt{p.first, p.second});
                }
            )
            .def_readwrite("name", &basics::Doodad::name)
            .def_readwrite("value", &basics::Doodad::value)
            .def_static("get_const", &basics::Doodad::get_const)
    
            .def("clone", [](const basics::Doodad &d) { return std::shared_ptr<basics::Doodad>(d.clone()); })
            .def("get_secret", &basics::Doodad::get_secret, py::return_value_policy::reference_internal)
            .def("write", [](const basics::Doodad &d) { auto tmp = d.write(); return make_pair(tmp.a, tmp.b); })
            .def("read", [](basics::Doodad &d, std::pair<std::string, int> p) { d.read(basics::WhatsIt{p.first, p.second}) ; });
    
        return m.ptr();
    }

Let's break that down into smaller chunks.

Holder types
""""""""""""

By default pybind11 uses a ``std::unique_ptr`` to manage references to an object in Python. The object is deallocated when the Python's reference count for it reaches zero. If objects are shared the holder type can be changed to a ``std::shared_ptr`` with the following.

.. code-block:: cpp

        py::class_<basics::Doodad, std::shared_ptr<basics::Doodad>>(m, "Doodad")

To enable transparent conversion of functions that convert between ``std::shared_ptr<Doodad>`` and Python ``Doodad`` the following macro is defined.

.. code-block:: cpp

    PYBIND11_DECLARE_HOLDER_TYPE(T, std::shared_ptr<T>);

In the case of ``Doodad`` this was required because many functions return or accept ``shared_ptr<Doodad>`` and, only one type of shared pointer can be associated with a given class using pybind11.

Functions
"""""""""

Two helper functions require wrapping at module level. This is easy with pybind11.

.. code-block:: cpp

        m.def("compare", &basics::compare);
        m.def("adjacent", &basics::adjacent);

In principle methods work exactly the same, except the ``.def`` applies to the module instead.

However, in this challenge there is also a set of methods that take / return a type not exposed to Python (``WhatsIt``).

.. code-block:: cpp

    void read(WhatsIt const & it);

    WhatsIt write() const;

The wrapper should transform these from / to a Python ``tuple``. Adding methods that perform some additional pre / post processing is trivially supported by pybind11 by binding to a lambda function.

.. code-block:: cpp

            .def("write", [](const basics::Doodad &d) { auto tmp = d.write(); return make_pair(tmp.a, tmp.b); })
            .def("read", [](basics::Doodad &d, std::pair<std::string, int> p) { d.read(basics::WhatsIt{p.first, p.second}) ; });

The first argument to the lambda is always a reference to the instance (i.e. ``self`` in Python).
Furthermore pybind11 automatically maps a two element Python ``tuple`` to a ``std::pair`` which means that we can stick to nice and standard C++ on this side of the fence.

Ideally, we'd like to provide custom converters for a type like ``WhatsIt`` that are automatically used whenever it is encountered in a function signature (like Swig's typemaps).  This is unquestionably possible with pybind11 (we can see the feature being used in how it handles STL types), but we have not fully investigated whether this feature has a documented public interface.

Constructors
""""""""""""

Two constructors are provided (and no default constructor). One takes default, and named, arguments.

.. code-block:: cpp

    explicit Doodad(std::string const & name_, int value_=1);

This is easily wrapped with.

.. code-block:: cpp

    .def(py::init<const std::string &, int>(), py::arg("name"), py::arg("value") = 1)

The second takes a reference to a ``WhatsIt`` object.

.. code-block:: cpp

    explicit Doodad(WhatsIt const & it);

Again, the mapping from a ``tuple`` to a ``WhatsIt`` is done by binding a lambda function. This time to the constructor ``__init__`` overload (pybind11 allows for function overloads which are tried in the declared order).

.. code-block:: cpp

        .def("__init__",
                [](basics::Doodad &instance, std::pair<std::string, int> p) {
                    new (&instance) basics::Doodad(basics::WhatsIt{p.first, p.second});
                }
            )

So in Python the following are now equivalent:

.. code-block:: python

    challenge.basics.Doodad("bla", 1)
    challenge.basics.Doodad("bla")
    tmp = ("bla", 1)
    challenge.basics.Doodad(tmp)

Const problems
""""""""""""""

The aforementioned ``get_const()`` static method returns a ``shared_ptr<const Doodad>`` (that can't be modified).
This is exposed to Python with.

.. code-block:: cpp

            .def_static("get_const", &basics::Doodad::get_const)

Which just works because we have changed the holder type for ``Doodad`` to ``shared_ptr<Doodad>`` as discussed above. Great!

But wait, it turns out that the objects returned by this method are modifiable. Bummer :(
Alas pybind11 strips out const modifiers. This issue was `bugreported <https://github.com/pybind/pybind11/issues/156>`_ but the developer indicated this will not be fixed because the concept of const is quite alien to Python and it would significantly complicate the library to support it.

Therefore this solution to the C++/Python bindings challenge does not support constness. Note that the Cython solution also had to have an ugly hack (different types to represent const and non-const objects).

Opaque types
""""""""""""

The challenge also involves wrapping the ``Secret`` type as an opaque type in Python.
Again this is easy with pybind11.

.. code-block:: cpp

        py::class_<basics::Secret>(m, "Secret");
    
On the C++ side ``Secret`` is a friend class of ``Doodad`` and has no publicly accessible constructors. Instead a reference to a ``Secret`` instance is returned by the ``get_secret()`` method of ``Doodad``. Importantly, the object is *owned* by ``Doodad`` and it remains responsible for destroying the ``Secret`` instance.

This can be communicated to pybind11 by specifying a `return value policy <http://pybind11.readthedocs.org/en/latest/advanced.html#return-value-policies>`_.

.. code-block:: cpp

        .def("get_secret", &basics::Doodad::get_secret, py::return_value_policy::reference_internal)

Note that while this works perfectly, I personally think objects that cannot outlive their creator are not very Pythonic and are best avoided.

Clone and unique_ptr
""""""""""""""""""""

For the C++ method ``clone``

.. code-block:: cpp

    virtual std::unique_ptr<Doodad> clone() const;

pybind11 allows this to be wrapped directly as you'd expect, even though the holder type is ``shared_ptr``, not ``unique_ptr``:

.. code-block:: cpp

        .def("clone", &basics::Doodad::clone)

This simply constructs a ``shared_ptr`` from a ``unique_ptr``, so the opposite would not work.

Containers
^^^^^^^^^^

The full wrapping code for the ``containers`` module is.

.. code-block:: cpp

    #include <pybind11/pybind11.h>
    #include <pybind11/stl.h>
    
    #include "basics.hpp"
    #include "containers.hpp"
    
    namespace py = pybind11;
    
    PYBIND11_DECLARE_HOLDER_TYPE(T, std::shared_ptr<T>);
    
    namespace containers {
    
    class DoodadSetIterator {
    public:
        DoodadSetIterator(DoodadSet::iterator b, DoodadSet::iterator e) : it{b}, it_end{e} {};
    
        std::shared_ptr<basics::Doodad> next() {
            if (it == it_end) {
                throw py::stop_iteration();
            } else {
                return *it++;
            }
        };
    
    private:
        DoodadSet::iterator it;
        DoodadSet::iterator it_end;
    };
    
    } // namespace containers
    
    PYBIND11_PLUGIN(containers) {
        py::module m("containers", "wrapped C++ containers module");
    
        py::class_<containers::DoodadSet> c(m, "DoodadSet");
    
        c.def(py::init<>())
            .def("__len__", &containers::DoodadSet::size)
            .def("add", (void (containers::DoodadSet::*)(std::shared_ptr<basics::Doodad>)) &containers::DoodadSet::add)
            .def("add", [](containers::DoodadSet &ds, std::pair<std::string, int> p) { ds.add(basics::WhatsIt{p.first, p.second}) ; })
            .def("__iter__", [](containers::DoodadSet &ds) { return containers::DoodadSetIterator{ds.begin(), ds.end()}; }, py::keep_alive<0,1>())
            .def("as_dict", &containers::DoodadSet::as_map)
            .def("as_list", &containers::DoodadSet::as_vector)
            .def("assign", &containers::DoodadSet::assign);
    
        c.attr("Item") = py::module::import("challenge.basics").attr("Doodad");
    
        py::class_<containers::DoodadSetIterator>(m, "DoodadSetIterator")
            .def("__iter__", [](containers::DoodadSetIterator &it) -> containers::DoodadSetIterator& { return it; })
            .def("__next__", &containers::DoodadSetIterator::next);
    
        return m.ptr();
    }

Now let's look at a few interesting things.

Overload resolution
"""""""""""""""""""

The ``add`` method is overloaded twice with different argument types.

.. code-block:: cpp

    void add(std::shared_ptr<Item> item);
    void add(basics::WhatsIt const & it);

In order for the compiler to know which one to select,
for the non-lambda binding of ``add``,
it is explicitly cast to a function pointer.

.. code-block:: cpp

        .def("add", (void (containers::DoodadSet::*)(std::shared_ptr<basics::Doodad>)) &containers::DoodadSet::add)

When called from Python the two bindings of ``add`` are tried in order.

Class attributes
""""""""""""""""

The ``.attr`` method can be used at module level to set constants and at class level to set class attributes.
In this case a class attribute is set for the ``typedef basics::Doodad Item``.
Because ``Doodad`` is part of the ``challenge.basics`` module in Python we instantiate a new module object with an import.
When reffering to something in the same module this is not needed and the current module object ``m`` can be used instead.

.. code-block:: cpp

        c.attr("Item") = py::module::import("challenge.basics").attr("Doodad");

STL types
"""""""""

Like Cython, pybind11 has built in support mapping the basic ``stl`` types (e.g. ``vector``, ``map``, ``tuple``, ``pair`` etc.) to standard Python types (e.g. ``list``, ``dict`` and ``tuple``).
Support for ``tuple`` and ``pair`` is included in the default header, for the rest an extra header needs to be included.

.. code-block:: cpp

    #include <pybind11/stl.h>

We thus can simply expose methods that use such types:

.. code-block:: cpp

        std::vector<std::shared_ptr<Doodad>> as_vector() const;
        std::map<std::string,std::shared_ptr<Doodad>> as_map() const;
        void assign(std::vector<std::shared_ptr<Doodad>> const & items);

directly as.

.. code-block:: cpp

        .def("as_dict", &containers::DoodadSet::as_map)
        .def("as_list", &containers::DoodadSet::as_vector)
        .def("assign", &containers::DoodadSet::assign);

Iterators
"""""""""

In order to support standard iteration over this ``DoodadSet`` container (e.g. with ``for``)
the ``DoodadSet`` wrapper defines the ``__iter__`` special function.

.. code-block:: cpp

        .def("__iter__", [](containers::DoodadSet &ds) { return containers::DoodadSetIterator{ds.begin(), ds.end()}; }, py::keep_alive<0,1>())

This function returns a ``DoodadSetIterator`` instance which is also created as an extension type.
In addition it uses the ``keep_alive`` call policy to make sure that the ``DoodadSet`` container
cannot be destroyed while an iterator pointing to it still exists.

.. code-block:: cpp

        py::class_<containers::DoodadSetIterator>(m, "DoodadSetIterator")
            .def("__iter__", [](DoodadSetIterator &it) -> DoodadSetIterator& { return it; })
            .def("__next__", &DoodadSetIterator::next);

The ``DoodadSetIterator`` is implemented as.

.. code-block:: cpp

    class DoodadSetIterator {
    public:
        DoodadSetIterator(DoodadSet::iterator b, DoodadSet::iterator e) : it{b}, it_end{e} {};
    
        std::shared_ptr<basics::Doodad> next() {
            if (it == it_end) {
                throw py::stop_iteration();
            } else {
                return *it++;
            }
        };
    
    private:
        DoodadSet::iterator it;
        DoodadSet::iterator it_end;
    };
 
SWIG interoperability
^^^^^^^^^^^^^^^^^^^^^

The final thing that is needed is to pass all unit test for conversions to and from SWIG.

The SWIG wrapped extension module ``converters`` contains functions like.

.. code-block:: cpp

    std::shared_ptr<basics::Doodad> make_sptr(std::string const & name, int value) {
        return std::shared_ptr<basics::Doodad>(new basics::Doodad(name, value));
    }

These then use typemaps declared in ``basics_typemaps.i`` such as.

.. code-block:: cpp

    %typemap(out) std::shared_ptr<basics::Doodad> {
    }

    %typemap(in) std::shared_ptr<basics::Doodad> {
    }

These typemaps are used by SWIG whenever a ``shared_ptr<Doodad>`` is encountered as input or output.

To get these typemaps to work with the pybind11 wrapped ``Doodad`` we first need to include the pybind header file in the SWIG module (``containers.i``) and declare the holder type.

.. code-block:: cpp

        %{
        #include "basics.hpp"
        #include <pybind11/pybind11.h>
        
        /* Needed for casting to work with shared_ptr<Doodad> */
        PYBIND11_DECLARE_HOLDER_TYPE(T, std::shared_ptr<T>);
        %}

During runtime, pybind11 keeps track of all the wrapped types that are provided by the various loaded modules in a global (but hidden) data structure stored in ``__pybind11_``. When a pybind11 module is imported it retrieves this structure and adds all the extension types it provides to maps within it.
Because the SWIG module does not itself add any types it performs no pybind11 initialization, and therefore it does not know about the already imported types. This can be fixed by adding a bit of code to the SWIG initializer.

.. code-block:: cpp

        %init %{
            /* Get registered types from other pybind11 modules */
            pybind11::detail::get_internals();
        %}

Now we can use the ``pybind11::cast`` function to get a ``shared_ptr<Doodad>`` from a pybind11 ``Doodad`` Python object and pass it to a SWIG wrapped function with this typemap.

.. code-block:: cpp

        %typemap(in) std::shared_ptr<basics::Doodad> {
            /* First make a pybind11 object handler around the PyObject *
             * Then, cast it to a shared_ptr<Doodad> using the pybind11 caster. */
            pybind11::object p{$input, true};
            try {
                std::shared_ptr<basics::Doodad> ptr(p.cast<std::shared_ptr<basics::Doodad>>());
                $1 = ptr;
            } catch(...) {
                return nullptr;
            }
        }

Because it is typically not needed by users the cast function in the other direction (from a ``shared_ptr<Doodad>`` to a ``Doodad`` Python type) is not part of the API. But of course it is used internally in pybind11 so we can simply extract it from there.

.. code-block:: cpp

        %typemap(out) std::shared_ptr<basics::Doodad> {
            /* Use a pybind11 typecaster to create a PyObject from a shared_ptr<Doodad> */
            pybind11::detail::type_caster<std::shared_ptr<basics::Doodad>> caster;
            pybind11::handle out = caster.cast($1, pybind11::return_value_policy::take_ownership, pybind11::handle());
            $result = out.ptr();
        }

The ``type_caster`` is a template at the heart of pybind11 that maps between types based on the list of registered types (see above) and the deduced types of the template.

Note that we set the return value policy to ``take_ownership``. This causes the pybind11 extension type to reference the existing object and take ownership. Python will call the destructor and delete operator when the reference count reaches zero.

With these two typemaps, a pybind11 wrapped ``Doodad`` can now be passed to and returned from a SWIG wrapped function.

Build issues
^^^^^^^^^^^^

The whole solution is build with.

.. code-block:: python

    import os, sys
    
    from distutils.core import setup, Extension
    from distutils import sysconfig
    
    cpp_args = ['-std=c++11', '-stdlib=libc++', '-mmacosx-version-min=10.7']
    
    if sys.platform == 'darwin':
        vars = sysconfig.get_config_vars()
        vars['LDSHARED'] = vars['LDSHARED'].replace('-bundle', '-dynamiclib')
    
    ext_modules = [
        Extension(
    	'libbasics',
            ['src/basics.cpp'],
            include_dirs=['include'],
    	language='c++',
    	extra_compile_args = cpp_args,
        ),
        Extension(
    	'libcontainers',
            ['src/containers.cpp'],
            include_dirs=['include'],
    	language='c++',
    	extra_compile_args = cpp_args,
        ),
        Extension(
            'challenge.basics',
            ['challenge/basics.cpp'],
            include_dirs=['pybind11/include', 'include'],
            language='c++',
            library_dirs=['.'],
            libraries=['basics'],
    	extra_compile_args = cpp_args,
        ),
        Extension(
            'challenge.containers',
            ['challenge/containers.cpp'],
            include_dirs=['pybind11/include', 'include'],
            language='c++',
            library_dirs=['.'],
            libraries=['basics','containers'],
    	extra_compile_args = cpp_args,
        ),
        Extension(
            'challenge.converters',
            ['challenge/converters.i'],
            include_dirs=['pybind11/include', 'include', 'challenge/include'],
            swig_opts=["-modern", "-c++", "-Ichallenge/include", "-noproxy"],
            library_dirs=['.'],
            libraries=['basics'],
            extra_compile_args=cpp_args,
        ),
    ]
    
    setup(
        name='challenge',
        version='0.0.1',
        author='Pim Schellart',
        author_email='P.Schellart@princeton.edu',
        description='Solution to the Python C++ bindings challenge with pybind11.',
        ext_modules=ext_modules,
    )

As can be seen, the C++ code to be wrapped is first built into shared libraries.

Unfortunately, distutils (and setuptools) doesn't seem to allow distinguishing between
extension modules and standard, Python independent, libraries.
This is a problem on OSX, because (unlike Linux) it has a separate concept of bundles (i.e. ``.bundle`` or often ``.so``) and dynamic libraries (i.e. ``.dylib`` sometimes also ``.so``).
Bundles are to be used as plug-ins for running programs which is why Python extension types are by default built as such.
But one cannot dynamically link against a bundle in the normal way.
And yet, this is required for using the same C++ across different pybind11 extension modules.
Therefore, the only way to get this to work is to build all extension modules as dynamic libraries instead.

.. code-block:: python

    if sys.platform == 'darwin':
        vars = sysconfig.get_config_vars()
        vars['LDSHARED'] = vars['LDSHARED'].replace('-bundle', '-dynamiclib')

Once again this shows that distutils / setuptools is not a good build system for C/C++...

On symbol visibility
""""""""""""""""""""

As an interesting side-note, when using cross module types with pybind11 it is also important to set the symbol visibility to ``default`` (as opposed to ``hidden``). This can be done with the compiler option ``-fvisibility=default`` but typically isn't necessary (since it is the default), but some examples of pybind11 online explicitly set visibility to ``hidden`` (to get smaller binaries) which creates problems when using cross-module types.

See also
========

* The full implementation of the pybind11 solution to the C++/Python bindings challenge is available in the ``pybind11`` branch of my `fork on github <https://github.com/pschella/python-cpp-challenge>`_.

* An excellent source of information is the online `pybind11 documentation <http://pybind11.readthedocs.org/en/latest/#>`_.

Relevant JIRA tickets
=====================

* `DM-5470 <https://jira.lsstcorp.org/browse/DM-5470>`_: Develop C++ code for experimenting with Python binding
* `DM-5471 <https://jira.lsstcorp.org/browse/DM-5676>`_: Wrap example C++ code with pybind11

