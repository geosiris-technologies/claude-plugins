# Writing: extract, rename, create, edit

Every command here writes. The rules of `SKILL.md` section 2 apply in full: never over the
original, plan first, an explicit yes, then verify.

## The safe sequence

1. **A work folder of its own.** Create a new, empty folder. If the command reads an existing
   package (`extract`, `rename`, `h5 --fix`, `create --into`, `repair --apply`, `relabel`),
   copy the `.epc` **and every `.h5` it names** (`energyml h5 FILE.epc` lists them) into it,
   and run the command on the copy. Why, as measured with 0.3.0:
   - `extract --out` writes the arrays it keeps into an `.h5` named like the source's, beside
     the output: with the output in the source's folder, it rewrote the source `.h5`;
   - `rename --apply` moves the data file itself, whatever `--out` says;
   - `create --into SRC.epc --out OTHER/NEW.epc` writes a package that still names the
     source's `.h5` by a relative path, so in another folder it reads `NOT FOUND` until the
     `.h5` is copied beside it; new arrays go to `--set dataFile=...` (default `data.h5`) beside
     the output.
2. **The plan.** Run the form that writes nothing (table below) and show the user: what objects
   (count, types, titles), which arrays, which files will appear, and where.
3. **An explicit yes** to that write.
4. **The write**, to a path that does not exist yet.
5. **Verify**: `energyml check NEW.epc | tail -1`, `energyml h5 NEW.epc` (every file found),
   and for a copy, the object count (`energyml list NEW.epc 2>&1 | tail -1`). Report the files
   written and that the originals are unchanged (`sha256sum` before and after, when it matters).

| Write | Plan form (writes nothing) | Write form |
|---|---|---|
| keep some objects | `extract F.epc --select sel.json --out W/x.epc --dry-run` | without `--dry-run` |
| rename the data file | `rename W/F.epc --from OLD.h5 --to NEW.h5` | `... --apply --out W/F-renamed.epc` |
| create objects | `create OP ... --plan` | `create OP ... --out W/new.epc` |
| import a SIGEOL export | `import FILE.geojson --stage S --plan` | `--out W/new.epc` |
| play a recipe | none that works on every recipe (see below) | `create --recipe R.toml --out W/x.epc` |
| fix the data-file names | `rename W/F.epc --from OLD.h5 --to NEW.h5` (or `h5 W/F.epc`) | `rename ... --apply --out W/F-fixed.epc` (`h5 --fix` misses WITSML channel data) |
| put repaired parts back | `repair W/F.epc --apply parts/` | `... --out W/F-fixed.epc` |
| relabel a version | `relabel W/F.epc --package P --schema-version V` | `... -o W/F-relabelled.epc` |

## A copy with only some objects ("only the wells")

A partial package must be **closed**: everything a kept object references must come along
(its interpretation, feature, CRS, datum...). Two ways:

**Python, closes the set for you** (recommended):

```python
from energyml.rust.epc import Epc
src = Epc.open("FILE.epc")
wells = [o.uri for o in src.objects() if o.qualified_type.endswith("WellboreTrajectoryRepresentation")]
needed, missing = src.needs(wells)            # the plan: what will come along
print(len(wells), "asked,", len(needed), "with what they need,", len(missing), "missing")
small = Epc.create("wells.epc")
small.set_data_root("W")                      # where its .h5 goes: the new folder
done = small.copy_from(src, wells)            # {'objects', 'added', 'missing', 'arrays', 'arrays_missed', 'unreadable'}
small.save_as("W/wells.epc")                  # W must exist
```

On Lutetia: 44 trajectories → 118 objects with what they need, 91 arrays, 0.19 s; the copy
checks clean. The new `.h5` keeps the source's file name (`W/Lutetia.h5`).

**Command line**, with a selection manifest:

```json
{ "manifest": 1, "chosen": ["eml:///resqml22.WellboreTrajectoryRepresentation(…)", "…"] }
```

`chosen` holds full URIs (from `energyml graph F.epc -o g.json`, or Python `o.uri`). `extract`
does **not** add what is missing: a selection that is not closed is refused at write time
(`this selection is not closed: 132 reference(s) would be broken`, exit 2), while `--dry-run`
lists the broken references and exits 0. Build `chosen` from Python `needs(...)` to get a
closed set. `--keep-arrays` leaves the arrays in the source file instead of copying them (the
new package then depends on the source `.h5`): only on request. `extract` does not create its
output folder.

"Only the wells" is ambiguous: RESQML trajectories and frames (`WellboreTrajectoryRepresentation`,
`WellboreFrameRepresentation`), WITSML `Well`/`Wellbore`/`Trajectory`/`Log`, or all of them.
Say which you chose, or ask.

## Renaming the data file

```sh
mkdir W && cp FILE.epc FILE.h5 W/
energyml rename W/FILE.epc --from FILE.h5 --to FILE_2026.h5        # the quote
energyml rename W/FILE.epc --from FILE.h5 --to FILE_2026.h5 --apply --out W/FILE-renamed.epc
energyml h5 W/FILE-renamed.epc                                     # → W/FILE_2026.h5
```

`--to` is a bare name, not a path. Without `--from`, 0.3.0 refuses with "declares no external
data file - pass `--from <NAME>`" on packages whose arrays name their file directly (Lutetia):
take the name from `energyml h5`. The quote on Lutetia: `738 field(s) in 663 object(s), 0
relationship(s)`. `--apply` moves the `.h5`, so the copy's original `.epc` now names a file that
is gone (it still reads only because the folder holds a single `.h5`): keep the renamed one.
Python: `plan_rename(old, new)` then `rename_data_file(old, new)` **and** `save_as(...)`
straight after: the rename moves the file at once but rewrites the package only on save.

## Creating objects: `energyml create`

```sh
energyml create --list --json | python3 -c 'import json,sys; [print(o["name"], "-", o["category"]) for o in json.load(sys.stdin)]'
energyml create --list --json OPERATION          # one operation: needs, takes, gives
energyml create --list --from input.geojson      # what this input allows, with refusals (long: 1 200 lines)
energyml create crs --set epsg=2154 --set vertical=5720 --plan
energyml create crs --set epsg=2154 --set vertical=5720 --out W/crs.epc
```

0.3.0 lists 79 operations (categories: create, geometry, synthetic, wells, import, faults,
analysis, frame, properties). Parameters are `--set name=value` (a number, `true`/`false`, a
JSON list `[...]` for list parameters, or text); inputs are `--from FILE`, `--from
FILE.epc#UUID`, or `--from SLOT=...` when the operation names slots (`edit-object` needs
`--from object=...`). `--now 2026-10-17T00:00:00Z` fixes the time stamp so two runs give the
same bytes. The written count is the plan's plus the activity records (`crs`: plan 4 objects,
6 written).

**From a CSV** (wells, intervals, properties): a column template is required — an OSDU
`ColumnBasedTableTemplate` (`--template t.json`) saying what each column means; a table without
one is refused naming `--template`. Look at an example template before writing one
(`recipe/donnees/bss/passes-etages.template.json` in energyml-lutetia).

```sh
energyml create wells-from-table --from wells.csv --template wells.template.json \
    --set project=2154 --set vertical=5720 --plan
energyml create wells-from-table --from wells.csv --template wells.template.json \
    --set project=2154 --set vertical=5720 --out W/wells.epc     # writes W/wells.epc and W/data.h5
```

Measured: 20 rows → 3 wellbores, 19 objects, 9 arrays, notes `[columns-ignored]` and
`[vertical-datum-offset]` (report them). `wells-and-properties-from-table` also writes the
columns as properties. Into an existing package that already holds the same wells, it refuses
by name (`matches 2 of the given wells by title and wellhead`).

**From a GeoJSON**: `create pointset` or `create polylines --from FILE.geojson --set
sourceEpsg=4326 --set project=2154 --set title=...`. `energyml import` reads only BRGM
SIGEOL `GeologicUnit` exports (it refuses a file without `--stage` when the name does not say
it).

**Editing one object**: `create edit-object --from object=W/F.epc#UUID --set
'fields=["Citation.Description = \"...\""]' --into W/F.epc --plan`, then `--out
W/F-edited.epc`. The plan shows old and new values. A title is not rewritten in place (every
reference holds a copy of it); `--set copy=NEW TITLE` makes a copy under a new uuid.

**Operations that invent data** (category `synthetic`: horizons by a law, well logs by a
gradient, grid properties, production) require a `source` and mark the objects
`synthetic`. Ask the user for the source text; never make one up.

## Recipes

```sh
energyml create --recipe recipe.toml --out W/model.epc        # all steps, one package
energyml create --recipe recipe.toml --without erreur --out W/clean.epc
```

The Lutetia recipe (105 steps played, 17 on hold) writes the 4.7 MB package and its 22.5 MB
`.h5` in 3.75 s. `--plan` **failed** on it in 0.3.0 (`step etages ... refused, and the run stops
there`: a later step reads arrays an earlier step only planned). For such a recipe the plan is
the write into a scratch folder, reported before anything is kept. Recipe keys are French
(`etape`, `nom`, `operation`, `entrees`, `attente`); file paths are relative to the recipe's
folder.

## Python `put` and `save_as`

`Epc.put(obj)` takes a dictionary or JSON text with `$type` (not XML) and checks it against the
schema (a bad field is refused by name). `template("resqml22.BoundaryFeature", uuid)` gives an
empty object to fill; the uuid is always given, never invented: ask, or derive it. `save_as`
returns the write's warnings and **does not create its folder**; it accepts the path of the
open file, so never pass the original's path.
