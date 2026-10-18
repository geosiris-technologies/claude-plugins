# Exporting representations

`energyml export` turns representations (surfaces, grids, point sets, polylines, wells) into
interchange files. It reads the package and writes only under `-o`. Check what the build offers:

```sh
energyml export FILE.epc --list
```

```text
obj        .obj       model/obj              Wavefront OBJ: triangles and polylines, placed and in metres
csv        .csv       text/csv               long-format CSV: one row per element and attribute, placed and in metres
geojson    .geojson   application/geo+json   RFC 7946 GeoJSON in WGS 84 - needs a reprojection backend
vtk        .vtk       application/octet-stream VTK legacy ASCII, DATASET POLYDATA
vtp        .vtp       application/xml        VTK XML PolyData (.vtp), surfaces and lines
vtu        .vtu       application/xml        VTK XML UnstructuredGrid (.vtu), the general container
```

## Choosing what to export

- `--filter SUBSTRING` matches the **qualified type**, case-insensitively (`TriangulatedSet`,
  `Grid2d`, `IjkGrid`, `PointSet`, `PolylineSet`, `WellboreTrajectory`, `resqml22.`). It does
  not match titles. To export by title, find the uuids first (`energyml list FILE.epc | grep
  -i 'top chalk'`) and export with a filter, or use Python `export_of(pkg, [uri, ...], "obj")`,
  which takes URIs.
- "The triangulated surfaces" = `--filter TriangulatedSet`; "the horizons as maps" =
  `--filter Grid2d`; "the grid" = `--filter IjkGrid`; "the wells" = `--filter
  WellboreTrajectory` (lines; `--tube RADIUS` sweeps them into solids for OBJ/VTK viewers).

## Where it goes

- `-o DIR/` (created if missing): one file per representation, named
  `qualifiedType_uuid.ext`. **Two versions of one uuid get the same name and the second
  overwrites the first**: on Lutetia `--filter TriangulatedSet` says `8 representation(s)
  written` and leaves 7 files. When the package holds versions (`list` shows `(version N)`), say
  so, or export to a single file.
- `-o FILE.ext`: every representation in one file where the format allows it (OBJ: one `o`
  block per object, checked: 8 blocks).
- Never `-o` into the package's own folder or onto existing files: pick a new folder.

## Reading the result

The summary is on stderr and the exit code is 0 even when nothing was written:

```text
158 representation(s) written as obj
1 skipped: `obj` cannot write them
  skipped: 0ba698fa-… resqml22.Grid2dRepresentation (H6 - top of the Chalk, in time)
8 could not be evaluated:
  0c53403a-… resqml22.TriangulatedSetRepresentation  external data `…` is in none of the 0 candidate file(s) for `Lutetia.h5`: file not found
```

Report the counts, and every skipped or refused object with its reason (grouped when many share
one). Measured on Lutetia: all 159 representations to OBJ in 0.84 s (45 MB), to GeoJSON in
1.10 s (23 MB), the 3 IJK grids to VTU with properties in 1.04 s (21 MB).

## Coordinates

- OBJ, CSV and VTK are **placed and in metres** in the package's projected system (Lutetia:
  EPSG:2154); each OBJ starts with comments naming the object, its uri, interpretation,
  feature and CRS.
- GeoJSON is **always WGS 84** (RFC 7946) and every feature has a `bbox`. Objects whose CRS has
  no usable EPSG code are **skipped**, by name. `--native-crs` writes stored coordinates
  instead, with a legacy `crs` member: knowingly non-conformant; say so if you use it.
- `--epsg CODE` reprojects (`4326`, `3857`, ...); the elevation is **not** converted.
- Reprojection from a definition that states no datum is refused; `--allow-silent-datum`
  overrides that and can put the model about a hundred metres off. Only with the user's
  informed agreement.
- Objects in a seismic **time** system are refused ("they cannot be placed in metres");
  `--time-equivalence M_PER_S` writes them anyway with a stated velocity (written into each
  file). It is an assumption: ask the user for the number, do not pick one.

## Shapes: `--as PATTERN=SHAPE`

`points`, `lines`, `surface`, `volume`, `auto`, and the summaries `outline` (convex hull, one
polygon) and `centroid` (one point). Rules are tried in order, first match wins:
`--as Grid2d=surface --as '*=points'`. A shape an object cannot take is refused per object
(`a set of polylines cannot be drawn as a surface`), not replaced.

## Metadata and properties

- `--standard-metadata`: uuid, uri, title, type, crs, interpretation and feature uuids and
  titles, content type, originator, on every feature (GeoJSON `properties`, VTK `FieldData`, OBJ
  comments).
- `--metadata keys.json`: `{"key": "dotted.path"}`, paths may cross references
  (`RepresentedInterpretation.InterpretedFeature.Citation.Title`). A key that matched nothing
  is reported, and written as `null` in GeoJSON.
- `--properties` (or `--property UUID`): VTK gets one array per property; GeoJSON a summary.
  It reads every property's arrays: slow on large packages. A property whose value count does
  not fit the geometry is **refused, not attached** (Lutetia: the two year-by-year grid series,
  720 720 values for 65 520 cells). `--properties-on points|cells` transfers values with
  `--transfer-method average|majority|minimum|maximum`: `majority` for categories (facies),
  never `average`.

## Python

```python
from energyml.rust.epc import Epc
from energyml.rust.resqml import export_of, standard_metadata
pkg = Epc.open("FILE.epc")
uris = [o.uri for o in pkg.find("TriangulatedSetRepresentation")]
open("out/surfaces.obj", "wb").write(export_of(pkg, uris, "obj"))
rich = export_of(pkg, uris, "geojson", metadata=standard_metadata())
```

## Section views: `profile`

```sh
energyml profile FILE.epc --track "x,y;x,y;x,y" --left 200 --right 150            # wells near the track
mkdir -p out && energyml profile FILE.epc --track track.geojson --left 200 --right 150 --svg out/long.svg
energyml profile FILE.epc --track ... --left 300 --right 300 --cross-every 300 --draw cross:2 --svg out/cross-2.svg
```

Without `--svg` it lists the wells whose head falls in the band (distance along, offset, side).
With `--svg` it draws wells, surfaces and grids cut along the track, and prints each refused
object (a well too far, an object in time without `--time-factor`). The track is in the
package's projected coordinates unless `--placement`. `--png` needs a build with `png`; the
0.3.0 archives refuse it (exit 2) after having written the SVG.
