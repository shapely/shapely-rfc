RFC 1: Roadmap for Shapely 2.0
==============================

* Request for comments: 1
* Author: Joris Van den Bossche, Casper van der Wel
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
vectorized functions (i.e. users have to loop over the array in Python to call
the scalar functions/properties) and this also has a large overhead limiting the
performance of such applications.

Further, Shapely currently wraps the GEOS library using `ctypes`. While being
a low-barrier way to wrap a C library, this runtime linking also entails overhead
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


### The array interface and ctypes attribute

Shapely provides an array interface to have easy access to the coordinates as
for example numpy arrays (
[docs](https://shapely.readthedocs.io/en/latest/manual.html#numpy-and-python-arrays)).
A small example:

```python
>>> line = shapely.geometry.LineString([(0, 0), (1, 1), (2, 2)])
>>> line.ctypes
<shapely.geometry.linestring.c_double_Array_6 at 0x7f75261eb740>
>>> line.array_interface()
{'version': 3,
 'typestr': '<f8',
 'data': <shapely.geometry.linestring.c_double_Array_6 at 0x7f752664ae40>,
 'shape': (3, 2)}
>>> np.asarray(line)
array([[0., 0.],
       [1., 1.],
       [2., 2.]])
```

So more specifically: the `ctypes` attribute is an ctypes array constructed from
the geometry coordinates. This is exposed in an "array interface" dictionary as
the `array_interface()` method, and as the `__array_interface__` dunder
attribute, which is used by numpy to create an array.

This functionality is available for Point, LineString, LinearRing (the
geometries that are directly backed by a CoordinateSequence) and MultiPoint.

There are some drawbacks / issues with this functionality:

- The current implementation is tied to ctypes (although, technically speaking,
  it is possible to provide an array interface without ctypes, and it would also
  be possible to keep exposing this as a ctypes array without using ctypes in
  the rest of Shapely).
- The
  [array interface](https://numpy.org/doc/1.18/reference/arrays.interface.html)
  and the general
  [Python buffer protocol](https://docs.python.org/3.8/c-api/buffer.html) is
  typically meant to share memory. However, this is not the case here: the
  actual coordinate values of the geometries are copied into the new array, and
  are not "viewed".
- In a similar way as having "iterable" geometries (see the previous section),
  having scalar geometries with an array interface gives difficulties when
  operating with numpy arrays of geometries and numpy ufuncs.
- The `coords` attribute already provides a more explicit way to access the
  coordinates, potentially as a numpy array.

Finally, the current array interface in Shapely provided a way to have access to
the coordinates as a numpy array without depending on numpy. This restriction is
not longer present, and we can directly return numpy arrays of coordinates where
needed.

**Proposal for Shapely 2.0** Deprecate (in 1.8) and remove the `ctypes`
attribute, the `array_interface()` method, and the `__array_interface__`
conversion to numpy arrays for the relevant geometries.

Access to the coordinates of a geometry as a numpy array is still available
through the `coords` attribute (`np.array(line.coords)` will still give the same
array as `np.array(line)` does now).


### Empty geometries

There are multiple ways that you can end up with empty geometries:

- Manually constructing one (e.g. with ``Polygon()`` or from a mapping
  representing an empty geometry)
- Importing from WKT / WKB representation
- As a result from a spatial operation (e.g. empty intersection)

Shapely 1.x is inconsistent in creating empty geometries between those different
ways. A small example for an empty Polygon geometry:

```python
# Method 1: using an empty constructor
>>> g1 = shapely.geometry.Polygon()
>>> type(g1)
<class 'shapely.geometry.polygon.Polygon'>
>>> g1.geom_type
GeometryCollection
>>> g1.wkt
GEOMETRYCOLLECTION EMPTY

# Method 2: converting from WKT
>>> g2 = shapely.wkt.loads("POLYGON EMPTY")
>>> type(g2)
<class 'shapely.geometry.polygon.Polygon'>
>>> g2.geom_type
Polygon
>>> g2.wkt
POLYGON EMPTY

# Method 3: result of a spatial operation
>>> g3 = shapely.geometry.box(0, 0, 1, 1).intersection(shapely.geometry.box(2, 2, 3, 3))
>>> type(g3)
<class 'shapely.geometry.polygon.Polygon'>
>>> g3.geom_type
Polygon
>>> g3.wkt
POLYGON EMPTY
```

Similar results can be seen for other geometry types. See
https://github.com/Toblerity/Shapely/issues/742 for more examples.

With Shapely 1.x, we can see that:

- The Python constructor always uses a GeometryCollection for empty geometries.
  However, GEOS supports empty geometries for other geometry types, as can be
  seen from the other examples (the empty geometries from WKB/WKT reading and
  spatial operations are results from calling GEOS).
- The empty geometry as result of the Python constructor shows an inconsistent
  state: the Python class indicates it is a "Polygon", but this contradicts the
  actual stored GEOSGeometry type (GeometryCollection).

It would be good to ensure the above examples return the same empty geometry
type consistently in all cases. The easiest option for this is to "follow"
GEOS' behaviour: *using type-specific empty geometries*. Not doing this would
mean that we would need to add special cases when wrapping all GEOS functions
that can result in empty geometries, which doesn't seem desirable. Following
GEOS' behaviour only requires to update the Python constructors to no longer use
empty GeometryCollection by default for empty geometries. Moreover, this is
consistent with PostGIS > 2 (older PostGIS versions treated all empty geometries
as GEOMETRYCOLLECTION EMPTY, but more recent versions support different empty
geometry types).

**WKB representation of empty points**

There is one additional, special case to consider: the WKB representation of
empty points (`POINT EMPTY`). This is the only empty geometry type for which
there is no standardized representation in WKB following the OGC standard. For
this reason, other software libraries and file formats have adopted the WKB
representation of "POINT (NaN NaN)" to represent empty points (e.g. PostGIS, R
`sf` package, GeoPackage file format, ...).

GEOS itself actually reads this WKB blob as POINT EMPTY, but when trying to
write the resulting empty geometry back to WKB, it refuses to do so ("Empty Points cannot be represented in WKB").

See this [Shapely#742](https://github.com/Toblerity/Shapely/issues/742#issuecomment-508990898)
comment with some more background on this issue.

For compatibility with other software (PostGIS, sf, etc), and to ensure all
geometry objects can be pickled using WKB, it seems good for Shapely to make
the same special case for writing the WKB of empty points.

**Proposal for Shapely 2.0** Follow GEOS behaviour to use type-specific empty
geometries (empty point, empty linestring, empty polygon, etc). More
specifically:

- Change the behaviour of the Python constructors (e.g. `Polygon()` to return
  POLYGON EMPTY instead of GEOMETRYCOLLECTION EMPTY).
- Deprecate the `shapely.geometry.base.EmptyGeometry` class.
- Change the `shapely.wkb` writer to use the "POINT (NaN NaN)" WKB to represent
  "POINT EMPTY".

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
