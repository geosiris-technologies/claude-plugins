# Reading a package

Every command below reads and writes nothing (except `graph -o`, `wells -o` and `docs --out`,
which write the file you name). Outputs shown are from energyml 0.3.0 on Lutetia
(`package/Lutetia.epc` of
[energyml-lutetia](https://github.com/geosiris-technologies/energyml-lutetia), 1 303 objects of
67 types, one 22.5 MB HDF5 file). Global flags (`--strict`, `--lenient`, `--h5`, `--log`) go
before or after the command.

## Identity: uuid or URI

An object is named by its uuid or by its ETP URI, `eml:///<qualifiedType>(<uuid>)`, or
`eml:///<qualifiedType>(uuid=<uuid>,version='<v>')` when it carries an `objectVersion`.
A uuid is enough unless two versions of the object are in the package: then the bare uuid
answers the unversioned one **without a warning**; use the full URI for the other. Quote URIs in
double quotes in every shell.

## list — what the package holds

```sh
energyml list FILE.epc                 # uuid  qualifiedType  title
energyml list --quick FILE.epc         # uuid  qualifiedType (reads no object XML)
energyml list --filter wellbore FILE.epc
```

```text
0022827d-0618-535b-b4eb-e5171866fdf4  eml23.Activity        contacts-from-frames → 1 point set(s)
fdf4510a-3aa1-578d-8935-0c9962e3773b  resqml22.TriangulatedSetRepresentation  F1
```

The count goes to stderr (`1303 object(s) in FILE.epc`). Group by type:
`energyml list FILE.epc | awk 'NF{print $2}' | sort | uniq -c | sort -rn`. The prefix is the
standard and version: `resqml20` (2.0.1), `resqml22`, `eml20`…`eml23`, `witsml20`, `witsml21`,
`prodml20`, `prodml22`, `prodml23`. A part that cannot be parsed keeps its row with the uuid as
title. A second version of an object is listed with `(version N)` after its title.

## show — one object

```sh
energyml show FILE.epc UUID            # XML, as the library writes it back
energyml show --json FILE.epc UUID     # Energistics JSON, `$type` on every object
energyml show --raw FILE.epc UUID      # the part's bytes exactly as stored
energyml show --json --with-data FILE.epc UUID   # each referenced object inlined under `_data` (reads the whole package)
```

Unknown object: `error: no object at \`UUID\`` and exit 2. The XML of `show` is re-serialised;
use `--raw` to quote the file itself.

## h5 and arrays — where the numbers are

```sh
energyml h5 FILE.epc                   # one line per data file named: pointers, where it was found
energyml arrays --summary FILE.epc     # pointers grouped by file
energyml arrays FILE.epc               # one line per pointer: uuid, field path, file:dataset
energyml arrays --values 3 FILE.epc    # shape, element type, first values (needs HDF5)
```

```text
Lutetia.h5  738 pointer(s)  → /…/package/Lutetia.h5
Lutetia.h5  738 pointer(s)  → ren/data-renamed.h5  [another file in the package's folder]
Lutetia.h5  738 pointer(s)  → NOT FOUND  (looking for `/resqml22/…/values`)      # exit 1
```

`arrays` says `0 unresolved` even when the file is absent: it counts names, not files. Only
`h5` (and an actual read) tells whether the data is there.

## geometry — what can be drawn

```sh
energyml geometry FILE.epc                       # every representation: patches, points, CRS
energyml geometry --filter TriangulatedSet FILE.epc
energyml geometry --filter PointSet --points 2 --metres FILE.epc   # first points, placed, in metres
```

```text
fdf4510a-…  resqml22.TriangulatedSetRepresentation  2 patch(es), 24 point(s)  [EPSG:2154, m]
    patch 0 @0  12 point(s)  12 triangle(s)
159 representation(s) evaluated, 307416 point(s)
```

It ends with what it could **not** evaluate, by name (`N not evaluated:`). `--points` prints
local coordinates; `--projected` applies the local placement (offset, rotation, depth flip),
`--metres` also converts units and refuses a unit it cannot convert. Reprojection between EPSG
systems is not done here (see `export --epsg`).

## properties, colours, tables

```sh
energyml properties FILE.epc                        # what each hangs on, kind, unit, count
energyml properties --representation UUID FILE.epc
energyml properties --check FILE.epc                # reads values; counts that contradict the document
energyml colors FILE.epc [--object UUID]            # colour of each object and the level it came from
energyml tables FILE.epc [--rows 10]                # lookup tables (ColumnBasedTable)
```

`colors` answers "no colour at any level - nothing is invented" rather than a default.
`tables` warns about keys that make rows unreachable.

## wells and series — logs, time series, production

```sh
energyml wells FILE.epc                       # CSV on stdout, one row per frame node
energyml wells FILE.epc --well GTH-1          # by uuid or a piece of the title
energyml wells FILE.epc -o out/wells.csv      # or --separate -o out/wells/ (one file each)
energyml series FILE.epc UUID                 # WITSML Log/ChannelSet, PRODML TimeSeriesData,
                                              # flow test, WellProductionParameters
energyml series FILE.epc UUID --json --rows 20
```

`series` on a WITSML log (Lutetia, `GTH-1 - Log (HDF5)`):

```text
channel set 23f4d208-…  GTH-1 - channels (HDF5)
  index MD (measured depth, m, increasing) 58 sample(s), 0 → 284.99999999999994
  channel temperature  degC  double: 53 good, 5 absent, 12.559619373186125 → 17.818141876879178
  channel pressure  bar  double: 55 good, 3 absent, 0.17718330261475715 → 18.34610064824716
```

On PRODML time series it prints the keys (asset identifier, flow, qualifier) and the samples;
on a flow test, one block per channel; on `WellProductionParameters`, one line per period.
`wells` leaves out a property whose value count does not match its frame and says so on stderr
(`[left out] GTH-1: ... holds 696 value(s) where the frame has 58 node(s)`). To find the
uuids: `energyml list --filter Log FILE.epc`, `--filter TimeSeriesData`, `--filter Frame`.

## graph — who references whom

```sh
mkdir -p out && energyml graph FILE.epc -o out/graph.json         # {nodes, edges}
energyml graph FILE.epc --format dot --filter Wellbore -o out/wb.dot
```

Nodes carry `uri`, `uuid`, `type`, `concept`, `title`, `present`, `externallyDefined`; edges
`from`, `to`, `path`. On Lutetia: 1 320 nodes, 6 516 edges; the 17 nodes with `present: false`
are Energistics catalogue `PropertyKind`s (`externallyDefined: true`), expected. `-o` does not
create its folder. For one object, Python `pkg.references(uri)` / `pkg.referenced_by(uri)` is
lighter.

## The model itself

```sh
energyml types --package witsml                 # the types this build reads
energyml types --describe TrianglePatch         # fields, cardinality, accepted subtypes, enumerations
energyml docs --package resqml --version 2.2 --out out/docs/   # xsd:documentation as JSON
energyml rules --house [--theme colour] [--json]   # the library's decisions where the spec is silent
```

## Selecting by region

```sh
energyml select FILE.epc --type WellboreTrajectory --outer zone.geojson --test head
```

Prints uuids (or `--format uri|json`) of objects in a region; the count on stderr
(`44 kept, 0 dropped, 0 refused`). GeoJSON rings are WGS 84 unless `--rings-epsg`.
