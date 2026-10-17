# The `energyml.rust` Python package

`pip install energyml-rust` (import `energyml.rust`) wraps the same Rust library as the command
line, with HDF5 inside the wheel: no `h5py`. Use it when the user wants numbers in numpy, a
loop over objects, a closed copy of some objects, or an object built from a dictionary. Run
scripts from a folder of your own with `python -I`, never from inside a data folder. Tested:
0.3.0 on CPython 3.11 (Linux x86_64).

Modules: `energyml.rust.epc` (`Epc`, `Summary`, `ArrayRef`), `energyml.rust.resqml`
(`geometry_of`, `mesh_of`, `properties_of`, `export_of`, `formats`, `standard_metadata`),
`energyml.rust.model` (`types`, `describe`, `template`), `energyml.rust.storage` (`Storage`,
the abstract interface). Each method has a docstring: `help(Epc.copy_from)`.

## Opening and finding

```python
from energyml.rust.epc import Epc
pkg = Epc.open("FILE.epc")                 # .h5 files are looked for beside it
objs = pkg.objects()                       # Summary: uri, uuid, qualified_type, title, object_version
pkg.find("Representation")                 # type or title contains, case-insensitive
pkg.find("Crs", "default")                 # type AND title
pkg.uri_of("fdf4510a-…")                   # uuid → uri, or None
pkg.dataspaces(), pkg.store_kind, pkg.path, pkg.attachments()
```

`uri_of` on a uuid held in two versions (one without `objectVersion`, one with) returned the
unversioned URI **without raising** in 0.3.0, although its docstring says it raises: compare
`[o for o in objs if o.uuid == u]` when versions matter.

## Reading objects and references

```python
obj = pkg.get(uri)                         # dict with "$type"; pkg.get_json(uri) for the text
pkg.references(uri)                        # [{uri, uuid, qualified_type, title, object_version, path, present}]
pkg.referenced_by(uri)                     # reads every object: slow on big packages
needed, missing = pkg.needs([uri, ...])    # the closure an extraction or a copy must carry
pkg.headers([uri, ...])                    # citations without reading the objects
```

## Arrays

```python
refs = pkg.arrays(uri)                     # [ArrayRef(file:path)], .file, .path_in_file
pkg.array_shape(refs[0])                   # {'dimensions': [12, 3], 'element_type': 'int64'}
values, dims = pkg.read_array(refs[0])     # flat numpy copy + dimensions → values.reshape(dims)
part, dims = pkg.read_array_part(refs[0], start=[0, 0], count=[2, 3])
```

Arrays come back **flat**; reshape with the dimensions. A missing `.h5` raises naming the file
and the dataset. `pkg.pin_data_file(name, path)` reads a file the package names `name` from
`path` (the `--h5 name=path` of the command line).

## Geometry, properties, export

```python
from energyml.rust.resqml import geometry_of, mesh_of, properties_of, export_of, standard_metadata
shape = geometry_of(pkg, uri)              # {'uuid','uri','title','qualified_type','kind','epsg','placement','patches'}
shape = geometry_of(pkg, uri, "metres")    # placement: 'local' | 'projected' (default) | 'metres'
points, triangles = mesh_of(pkg, uri, "metres")   # flat arrays, merged and triangulated
values, problems = properties_of(pkg, uri)        # values that fit the shape; problems = those that do not
data = export_of(pkg, [uri, ...], "obj")          # bytes; formats(): obj, csv, geojson, vtk, vtp, vtu
```

Patches carry `topology` (`points`, `triangles`, `polylines`, `hexahedra`, ...), `points`, and
for IJK grids `cells`, `dimensions`, `cell_index`. Each property in `values` is a dict
`{uuid, name, kind, uom, scope, components, values, colour_components, colours}`; `colours` is
empty when the package states no colour map (none is invented). On Lutetia's
`Ypresian-Chalk grid`: 7 properties of 65 520 cells, and 2 problems (the year-by-year series,
720 720 values). `export_of` with several URIs writes them all to one file, versions included.

## Checking

```python
r = pkg.check()                            # same numbers as `energyml check`
r["errors"], r["warnings"], r["sound"]
r = pkg.check(schema=False)                # r["schema"] == {'checked': False, 'issues': []}
```

Keys: `errors, warnings, sound, diagnostics, unreadable, schema, properties, counts, rules,
references, relationships`. "Not checked" is not "clean": say which sections ran.

## Writing (rules of SKILL.md section 2 apply)

```python
small = Epc.create("new.epc")              # empty; where it goes is decided at save_as
small.set_data_root("W")                   # where its data files are written
done = small.copy_from(src, uris)          # closes the set; returns counts and what was added
small.save_as("W/new.epc")                 # returns the warnings; W must exist

from energyml.rust.model import template, describe, types
obj = template("resqml22.BoundaryFeature", "6f0e0c5e-…")   # uuid given, never invented
obj["Citation"].update(Title="Top Chalk", Originator="me",
                       Creation="2026-10-17T12:00:00Z", Format="energyml-rust")
pkg.put(obj)                               # returns the uri; JSON text works too, XML does not
pkg.is_dirty                               # a property, not a method
pkg.save_as("W/edited.epc")                # never the original's path: save_as accepts it

pkg.plan_rename("OLD.h5", "NEW.h5")        # {from, to, objects, fields, relationships, refusals, allowed}
pkg.rename_data_file("OLD.h5", "NEW.h5")   # moves the file NOW; then save_as at once
```

`put` refuses by name: an unknown type (`unknown type \`resqml22.Nope\` (not registered)`), a
missing uuid, a field of the wrong type. There is no `save()`: always `save_as` to a new path.

## A standalone object file

The command line reads only EPC packages (`not a zip` on an `.xml` or `.json`). A **JSON**
object (Energistics JSON with `$type`) can be wrapped:

```python
pkg = Epc.create("one.epc"); pkg.put(open("object.json", encoding="utf-8").read()); pkg.save_as("W/one.epc")
```

Its references then point at objects the new package does not hold (`check` reports them) and
its arrays at an `.h5` that may not be beside it. An **XML** object outside a package cannot be
read by 0.3.0 (`put` takes JSON only); ask for the package it came from.
