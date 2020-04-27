RFC 1: Roadmap for Shapely 2.0
==============================

* Request for comments: 1
* Author:
* Date: 2020-04-22

## Abstract

This proposal summarizes changes for a new major version of Shapely: a refactoring
of the internals (switching from ctypes to wrapping GEOS objects in a Python
extension type written in C), providing performant vectorized operations
combined with a small clean-up of the API.

## Introduction

Shapely provides a user-friendly interface for the manipulation and analysis of
geometric objects, based on the GEOS C++ library. Shapely is widely used and has
become the de-facto standard for working with "Simple Feature" geometries in
Python.

Currently, Shapely only provides an interface for scalar geometry objects. When
organizing many Shapely geometries in arrays, there is missing functionality for
vectorized functions (i.e. users have to manually loop over the array to call
the scalar functions/properties) and this also has a large overhead limiting the
performance of such applications.

Further, Shapely currently wraps the GEOS library using `ctypes`. While being
a low-barrier way to wrap a C library, this runtime linking also gives overhead
and robustness issues (see below).

Over the last years, effort has been put in improving the performance through
vectorized interfaces to GEOS. After experiments in cython in GeoPandas, this
effort is now concentrated in the
[PyGEOS package](https://github.com/pygeos/pygeos/). PyGEOS is a new package
providing all GEOS functionality as vectorized functions operating on numpy
arrays (numpy ufuncs), and can provide considerable performance benefits
(https://caspervdw.github.io/Introducing-Pygeos/).

However, creating a separate project with its own Geometry class that is not
fully compatible with the widely used Shapely package fragments the user
experience. For having a clear story for the community around geometry
operations in Python and to avoid a split in the community, this RFC proposes to
integrate the improvements developed in PyGEOS into Shapely. See also
[#782](https://github.com/Toblerity/Shapely/issues/782/) for some discussion on
this.

More specifically, this RFC proposes to:

* Refactor the internals to remove the usage of `ctypes`, and instead wrap the
  GEOS geometry objects in a Python extension type.
* Provide functionality to work with arrays of geometry objects with a familiar
  NumPy array interface, i.e. optimized, vectorized ufuncs for all GEOS
  operations. This adds a required dependency on numpy.
* Use the vectorized operations for the implementation of the scalar operations,
  preserving the existing Shapely scalar API.
* Make some API changes that are partly required and also desirable (e.g.
  regarding mutability of geometries, ctypes interface, etc).

In practice, the above proposal means to fully migrate the PyGEOS code into
Shapely's core, and refactoring the rest of Shapely on top of that (instead
of taking PyGEOS as a dependency of Shapely).

Note, although this expands the scope of Shapely to *array* functionality,
this still explicitly excludes feature attribute information (which is left
to other libraries such as GeoPandas).

On a practical note, the following workflow is proposed:

* The upcoming Shapely 1.8 (the current master branch) includes deprecation
  warnings for behaviour that will change (in addition to dropping python 2
  support, [#743](https://github.com/Toblerity/Shapely/issues/743)).
* Shapely 2.0 becomes the release with the refactor from integrating PyGEOS. A
  development branch is created for this, which means that initially, we will
  need to keep the two (or three including 1.7-maint) branches in sync to a
  certain extent.


## Details

### Geometry extension type and Python subclasses

The approach in PyGEOS is to store the pointer to the actual GEOS object in a
lightweight [Python extension type](https://docs.python.org/3/extending/newtypes_tutorial.html).
This way, the GEOS pointer is accessible from C without Python overhead as a
static attribute of the Python object (an attribute of the C struct that makes
up a Python object).

Hence, a single `Geometry` Python extension type is defined in C wrapping a
`GEOSGeometry` pointer. In addition, this extension type can further be
subclassed in Python to provide the geometry type-specific classes (Point,
LineString, Polygon, etc).

In [pygeos/pygeos#104](https://github.com/pygeos/pygeos/pull/104), we
experimented with this Python subclassing approach (a registry of subclasses is
constructed such that the C functions creating new geometries can return the
correct Python subclasses from C). The conclusion here is that this approach
only adds minimal overhead to the C functions. This
[branch](https://github.com/Toblerity/Shapely/compare/master...jorisvandenbossche:geometry-subclasses?expand=1)
starts to refactor the existing Shapely geometry classes as subclasses of the
C extension type.

Compared to the current `ctypes` based approach, this has some benefits:

* `ctypes` is not (or no longer) the recommended way to wrap a complex C/C++
  library. It adds overhead and possible runtime issues.
* Storing the C-pointer in a read-only static attribute on the extension type
  instead of a dynamic attribute (current ctypes-based geometry classes)
  gives a more robust implementation. For example, this makes it impossible
  to crash the kernel if you change or deallocate the pointer.
* Connecting the object to the garbage collector on the C level (to ensure the
  GEOS object is destroyed when the reference count becomes zero) instead of
  coding this in Python makes memory leaks much harder to come by.
* The Python extension type geometry objects are immutable, making hashing and
  copying easier.

Concluding: for scalar geometries, using a Python extension type will not
give a big speed difference (as the time is typically dominated by the GEOS
operation), but it gives a more robust GEOS interface, and allows for
extending Shapely with high-performant vectorized functions (see below).

Current implementation in PyGEOS: https://github.com/pygeos/pygeos/blob/51d3798d577574c40c3ca6645a60d21576b892c7/src/pygeom.c#L198


### Vectorized array functions: numpy ufuncs

An additional advantage of the Python extension type to hold the GEOS object, is
that the extension type's struct with the GEOSGeometry point is still accessible
from C or Cython without much overhead. This allows writing vectorized
functions on that level, avoiding Python overhead while looping over the array.

PyGEOS creates those array functions as
[Numpy "universal functions"](https://numpy.org/doc/stable/reference/ufuncs.html)
or *ufuncs*, making use of the Numpy C API to write new ufuncs
([ref](https://numpy.org/doc/stable/user/c-info.ufunc-tutorial.html)).

While this adds a dependency on `numpy`, this has some advantages:

* This makes it straightforward to create functions that work on arrays of
  geometry objects (numpy provides a C API and templates to make it easier to
  implement such functions).
* This automatically gives us the features of ufuncs familiar from numpy:
  vectorized element-wise functions, broadcasting over different dimensions, ..

An illustrative example (using the PyGEOS API, this needs to be adapted to
Shapely of course):

```python
import pygeos
import numpy as np

points = np.array([
    pygeos.Geometry("POINT (1 9)"),
    pygeos.Geometry("POINT (3 5)"),
    pygeos.Geometry("POINT (7 6)")
], dtype=object)
box = pygeos.box(2, 2, 7, 7)

pygeos.contains(box, points)
```

gives

```python
array([False,  True, False])
```

Here, we computed whether the three points are contained in `box` in one go.

Apart from the nicer interface (no need to manually loop through the
geometries), this also provides a considerable speedup. Depending on the
operation, this can give a performance increase with factors of 4x to 100x
(the high factors are obtained for lightweight GEOS operations such as contains
in which case the Python overhead is the dominating factor). See
https://caspervdw.github.io/Introducing-Pygeos/ for more detailed examples.

In addition, this also opens up possibilities for multithreading (release the
GIL during GEOS operations for better performance). See
[pygeos/pygeos#113](https://github.com/pygeos/pygeos/pull/113) for experiments
on this.

Concluding: adding the GEOS operations as ufuncs provides performant geospatial
operations using the familiar NumPy array interface. This gives a nice user
interface and will provide a substantial performance improvement to GeoPandas
and others who work with arrays of geometries.

### Incorporation in Shapely

The Geometry extension type will simply be used as new base class for the
different Geometry subclasses, but the vectorized functions detailed above need
to find a new home in the Shapely package.

There are different sets of functions to consider:

- Functions which right now are only exposed as properties or methods on the
  Geometry subclasses: unary and binary predicates (is_empty, is_valid,
  contains(), within(), etc), unary and binary spatial analysis methods
  (centroid, boundary, difference(), union(), buffer(), etc). Those currently
  don't exist as *function*, and thus need to be added in a certain module.

- Other scalar functionality already lives in separate modules (and not as
  attributes/methods on the Geometry classes), e.g. `ops`, `algorithms`,
  `affinity`, `validation`, `wkt`/`wkb`, etc. Vectorized versions of those
  functions could also replace the existing ones in their current submodules
  (with the presumption that the behaviour of the ufunc for scalar input is
  compatible with the current scalar functions), instead of placing the ufuncs
  in a new submodule.

- A set of functions to create arrays of geometries (e.g. `points()` to create
  an array of points from numpy arrays of x/y coordinates). Those could be added
  to the `shapely.geometry` module, similarly as this already has a `box()`
  function.

So multiple options exist for this incorporation: 1) put all ufuncs in a single
"vectorized" or "geos" submodule or 2) put the ufuncs in their respective
submodule depending on the functionality (`affinity`, `geometry`, etc), and only
create a new submodule(s) for the ones that don't already exist / fit in an
existing submodule.

An advantage of the second approach is to avoid duplication in the API surface.
As an example, consider the `shapely.affinity` submodule which has a
`affine_transform(geom, matrix)` function. In Shapely 2.0, we can provide a
vectorized version of this function (but that, given a scalar geometry, still
returns a scalar transformed geometry). Putting all ufuncs in a separate module
would mean that we have a vectorized `shapely.vectorized.affine_transform`
handling both single geometries and arrays of geometries and a
`shapely.affinity.affine_transform` only handling single geometries. This seems
a duplication in the API we would like to avoid. The second approach would be to
have the version in `shapely.affinity` handle both cases (this would be a
backward-compatible change).

On the other hand, a single namespace (one submodule) with all ufuncs can also
be beneficial for usability/discoverability of those functions (the top-level
namespace of PyGEOS basically currently provides this).

**Proposal for Shapely 2.0**: to discuss.


## Proposed API changes

In addition to the core Geometry base class and the vectorized ufuncs,
there are some additional proposals for changing the Shapely API, detailed
below.

### Mutability

Currently, some of the geometry classes are mutable. Illustrative code:

```python
>>> line = shapely.geometry.LineString([(0,0), (2, 2)])
>>> print(line)
LINESTRING (0 0, 2 2)

>>> line.coords = [(0, 0), (10, 0), (10, 10)]
>>> print(line)
LINESTRING (0 0, 10 0, 10 10)
```

Currently, the `pygeos.Geometry` class is immutable by design.

**Proposal for Shapely 2.0** The proposal is to deprecate mutability in Shapely
1.8, and have immutable geometries in Shapely 2.0. This also ensures that the
geometries can become hashable again (xref
https://github.com/Toblerity/Shapely/issues/209).


### Adapters (`CachingGeometryProxy`)

Shapely provides the functionality to create geometry-like proxy objects with
coordinates stored outside Shapely geometries in another sequence or array.

```python
>>> from shapely.geometry import asLineString, asShape

>>> arr = np.array([[1, 2], [3, 4]])
>>> asLineString(arr)
<shapely.geometry.linestring.LineStringAdapter at 0x7f8e6c2a3dc0>

>>> context = {'type': 'LineString', 'coordinates': ((1.0, 2.0), (3.0, 4.0))}
>>> asShape(context)
<shapely.geometry.linestring.LineStringAdapter at 0x7f8e6c2acdf0>
```

The Adapter classes trade performance for (initial) reduced storage of
coordinate values. While it would be possible to keep this functionality, it
adds complexity to the library for limited value.

https://github.com/Toblerity/Shapely/issues/124
https://github.com/Toblerity/Shapely/issues/69

**Proposal for Shapely 2.0** is to remove those adapter classes (and deprecate
in Shapely 1.8).


### Iteration / getitem of "Multi" geometries

Currently, one can iterate through the "Multi" geometry types, but you get an
error with non-multi types:

```python
>>> from shapely.geometry import Point, MultiPoint
>>> p = Point(1, 1)
>>> mp = MultiPoint([(1, 1), (2, 2), (3, 3)])
>>> print(mp)
MULTIPOINT (1 1, 2 2, 3 3)

# using list(..) as way to iterate over it
>>> list(p)
...
TypeError: 'Point' object is not iterable
>>> list(mp)
[<shapely.geometry.point.Point at 0x7f98475a0d90>,
 <shapely.geometry.point.Point at 0x7f98475a0550>,
 <shapely.geometry.point.Point at 0x7f98475a0d30>]
# getitem also works
>>> mp[0]
<shapely.geometry.point.Point at 0x7f98462cab50>
```

This can certainly be convenient in certain cases, but also creates an
inconsistency between the different geometry types and it creates some problems
in other cases. For example, putting such scalars in a numpy object array:

```python
# single point is fine
>>> np.array([p], dtype=object)
array([<shapely.geometry.point.Point object at 0x7f98476cb2e0>],
      dtype=object)

# MultiPoint raises an error
>>> np.array([mp], dtype=object)
...
TypeError: object of type 'Point' has no len()

# MultiPolygon gets splitted
>>> from shapely.geometry import MultiPolygon, box
>>> mpoly = MultiPolygon([box(0, 0, 1, 1), box(2, 2, 3, 3)])
>>> np.array([mpoly], dtype=object)
array([[<shapely.geometry.polygon.Polygon object at 0x7f9846379ac0>,
        <shapely.geometry.polygon.Polygon object at 0x7f9846379670>]],
      dtype=object)
```

When working with arrays of geometry objects, the "iterability" of the
geometries becomes an annoyance.

The "Multi" geometry types also have a `.geoms` attribute, which gives access to
a list of geometries to iterate over. So the question here can be: isn't this
sufficient? Accessing this specific attribute doesn't seem problematic, as you
need to know anyway that you have a "Multi" geometry, as the others will raise an
error on iter/getitem.

**Proposal for Shapely 2.0** Deprecate and eventually remove the iterability of
"Multi" geometries, and point users to the `.geoms` attribute.

This means that code like `mpoly[0]` or `for geom in mpoly: ...` needs to be
replaced with `mpoly.geoms[0]` or `for geom in mpoly.geoms: ...`.


### Empty geometries

TO FILL IN

inconsistencies in emtpy geometries, cfr https://github.com/Toblerity/Shapely/issues/742


### Spatial indexing with STRTree

TO FILL IN

PyGEOS also implements an STRtree. However, the API is not compatible with the
one that is now in Shapely (but more similar to the API of `rtree`): the tree in
PyGEOS returns integer indices, while the tree of Shapely returns actual
geometry objects (cfr https://github.com/Toblerity/Shapely/issues/615).

The behaviour of Shapely can easily be implemented on top of the behaviour of
PyGEOS (to avoid a duplicate tree implementation). But the question here will
still be how we want to provide this in the public API. Can we somehow unify
this? Or just provide two tree objects users can choose from? Or is it
acceptable to deprecate the current shapely behaviour? (or provide a keyword to
still return geometries instead of integers?)


### Prepared Geometries

TO FILL IN

In Shapely, there is a `PreparedGeometry` (and `prep` function), which needs to
be used explicitly by the user. In PyGEOS, we are however thinking to "just" use
this when appropriate under the hood, without the user needing to think about
this. For example the STRtree already uses this (when a predicate is specified),
and in https://github.com/pygeos/pygeos/pull/92 I am looking at enabling this in
the vectorized spatial predicates like `contains` etc. The question here is if
we will always know when this is "appropriate", or if there are cases that it
can unexpectedly give a large overhead for situations prepared geoms were used
unnecessarily (feedback on this in the linked PR is very welcome!). But if we do
it under the hood, this could also mean that `PreparedGeometry` could be
deprecated.

### The array interface and ctypes attribute

TO FILL IN


## Considerations

The changes proposed in this RFC will have a profound impact on the Shapely
package. Among other things:

- Introduction of a hard dependency on numpy.
- Increased complexity (usage of C instead of ctypes for the core GEOS
  wrappers), giving a higher barrier for entry for contributors for that
  part of the codebase (and requiring a C compiler to install from source).
- Expanding the scope of Shapely.
- A set of API changes.

It is therefore important to seek feedback from a wide variety of contributors
and users on the exact details.

However, we think that those proposed changes will be beneficial for the long
term health of the Shapely project and the broader Python geospatial ecosystem.
First and foremost, it avoids a split in the Python GEOS community with two
potentially incompatible GEOS wrappers. Rather, it expands the familiar
Shapely library with improved and new functionality. This will also bring
new contributors to Shapely.

Additional considerations:

- With the improvements in the Python packaging ecosystem (binary wheels,
  conda), the dependency should not be problematic for users (numpy is easy to
  install on a wide variety of platforms).
- Trading ctypes with the C extension type / ufuncs can also improve the general
  design of the internals (e.g. removal of the implementation backend layer),
  and can remove some duplicated functionality (the faster cython functions vs
  the main ctypes based functionality).
- Although the proposed API changes will always impact some users, the changes
  are limited to more "esoteric" aspects (mutability, adapter, iteration) which
  will provide a more robust API, while the majority of the core Shapely
  geometry API remains intact (backwards compatible). Additionally, the Shapely
  1.8 release will provide appropriate warnings for all behaviour that will
  change in Shapely 2.0, smoothing the upgrade path.
