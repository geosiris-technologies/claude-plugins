# Troubleshooting

Symptoms seen with energyml 0.3.0 and 0.4.0, what they mean, and what to do. For anything else, rerun the
failing command with `--log debug` (stderr only) and quote the few lines that matter; upstream
issues: <https://github.com/geosiris-technologies/energyml-rust/issues>, with
`energyml --version`.

## Symptoms

| Symptom | Meaning | Do |
|---|---|---|
| `no support for \`application/x-hdf5\` (build with the \`hdf5\` feature)` | this binary has no HDF5 (a Linux musl archive, or a cargo build without `--features hdf5`) | install a gnu/macOS/Windows archive or the Python wheel ([install.md](install.md)); the data is not missing |
| `is in none of the 0 candidate file(s) for \`X.h5\`: file not found` | the package names `X.h5` and it is not beside it | `energyml h5 FILE.epc`; ask where the file is; `--h5 PATH` for one command, or `rename --from OLD --to NEW` on a copy |
| `energyml h5` prints `NOT FOUND` and exits 1 | same | same |
| `check` says `0 error(s)` but geometry is empty | `check` does not look for the `.h5` | run `energyml h5`; say the arrays are missing |
| `[another file in the package's folder]` in `h5` | the `.h5` was renamed and found by a search | tell the user; offer `rename --from OLD --to NEW` on a copy ([checking-and-repairing.md](checking-and-repairing.md#arrays-not-found-h5---h5-h5---fix)) |
| `warning: N dataset(s) read from X instead of the file they point at` (0.4.0, stderr) | same: the numbers came from a file found by the search | same; the result is right only if X is the right file, so name it to the user |
| `warning: --h5 X does not exist and is ignored` (0.4.0) | a mistyped `--h5` path | fix the path; the command ran without it |
| `refused: \`data/x.h5\` is a path` (`rename`) | an EPC names its data file by a bare name | put the `.h5` beside the `.epc` |
| `error: no object at \`UUID\`` (exit 2) | not in this package | `energyml list FILE.epc \| grep -i PART-OF-UUID` |
| an object shown is not the version the user meant | two versions share the uuid; the bare uuid gives the unversioned one | use `eml:///TYPE(uuid=UUID,version='V')` |
| fewer export files than `N representation(s) written` | two versions of one uuid wrote the same file name | export to one file, or with Python `export_of` |
| `not a zip: invalid Zip archive` | the file is not an EPC (a standalone `.xml`/`.json`, a truncated download) | see [python.md](python.md#a-standalone-object-file); check the size and the source |
| `XML error: ...` in `objects:` of `check`, `repair` exits 1 | a part no parser reads | `energyml repair` ([checking-and-repairing.md](checking-and-repairing.md)) |
| `... declares no external data file - pass \`--from <NAME>\`` (`rename`) | the arrays name their file directly | add `--from NAME.h5` (the name `h5` shows) |
| `this selection is not closed: N reference(s) would be broken` (`extract`) | the manifest misses objects the kept ones reference | build `chosen` from Python `needs(...)`, or use `copy_from` |
| `is the package being read: extracting onto it would destroy it` (`extract`, 0.4.0) | `--out` is the source `.epc` | a new path in a new folder |
| `cannot answer ...: N object(s) of the package could not be read` (Python `needs`, `referenced_by`, 0.4.0) | an unreadable part might be the one that points there | report the part; `energyml repair` |
| `No such file or directory (os error 2)` (`os error 3` on Windows) on writing | `extract`, `graph -o`, `view draw --svg`, `save_as` do not create the folder | `mkdir -p` it first |
| `they cannot be placed in metres. Pass --time-equivalence` | an object in a seismic time CRS, no velocity in the package | report it; a velocity is the user's assumption, not yours |
| GeoJSON skips objects `without a usable EPSG code` | GeoJSON must be WGS 84 | report them; `--native-crs` only with the user's agreement |
| `this build cannot write a PNG` (`profile --png`) | the 0.3.0 and 0.4.0 archives have no `png` feature | the SVG was written anyway; offer it |
| `this build cannot write a PNG` (`view draw --png`, 0.4.0) | same | nothing was written: run it again without `--png` |
| an offered view is refused at `view draw` (`well-sheet: the frame carries no stratigraphy ...`) | `view list` reads no array; the drawing needs data the object lacks | quote the refusal; offer another view of that object |
| dozens of `house/provenance/activity-follows-its-template [warning]` (0.4.0 `check`) | a house rule new in 0.4.0: activity records written by an older tool disagree with their templates | report them as one group about provenance records ([checking-and-repairing.md](checking-and-repairing.md)) |
| `step \`X\` ... refused, and the run stops there` with `create --recipe --plan` | the plan of a later step needs arrays an earlier step only planned | play the recipe with `--out` into a scratch folder instead |
| `` `mcp` is not a command of `energyml` `` | `energyml-mcp` is not installed (it is not in the archives) | [mcp.md](mcp.md) |
| `fields ... is not rewritten in place` (`edit-object`) | a title is copied in every reference | `--set copy=NEW TITLE`, or another field |
| `CONNECT proxy failed` / network errors with `fetch` | `fetch` (even its quote) and `terrain` contact public servers | check the network; nothing else needs it |
| hundreds of `[Warning] ... PropertyKind references ..., which is not in this package` on a write | catalogue property kinds are not carried, by design | count them; not a defect |

## Where the upstream documentation and the binary disagree

Found while testing 0.3.0 (release archive for Linux x86_64 gnu and musl, the PyPI wheel,
`cargo install energyml-mcp-cli`), and checked again on 0.4.0 (release archive for Windows
x86_64, the PyPI wheel) where marked. The binary wins; this skill follows the binary.

- **`enyml` alias**: the README, `docs/cli.md` and the CLI crate README say `energyml` installs
  with the alias `enyml`; the release archives hold `energyml` only (0.3.0 and 0.4.0).
- **`cli-reference.md`** says it was generated with the features `hdf5, png`; the 0.3.0 and
  0.4.0 archives have no `png` (`profile --png` and `view draw --png` are refused). Their
  option lists otherwise match `--help` exactly (checked for all 27 commands of 0.3.0; on
  0.4.0 the same 27 have the same options and `view` is new).
- **`energyml-mcp`** is not in the release archives; `docs/cli.md` presents `energyml mcp
  file.epc` without saying so.
- **Publication TODOs** (0.3.0 docs): `docs/getting-started.md` said the Docker image was not
  yet published and `docs/python.md` that `pip install energyml-rust` was to be verified; the
  0.4.0 docs no longer say so. The Lutetia README still says the wheel is built from the
  repository "until it is on PyPI" (0.1.2 to 0.4.0 are there).
- **`rename` without `--from`**: the help says the package's own declaration answers when it
  declares one file; on Lutetia (one `.h5`, named directly by the arrays) 0.3.0 and 0.4.0
  refuse with "declares no external data file".
- **Export file names**: `docs/cli.md` says the uuid in `qualifiedType_uuid.ext` makes a name
  unique; two versions of one uuid collide and one file is overwritten.
- **`create --recipe ... --plan`** (`docs/creating-objects.md`, Lutetia `docs/build.md`) fails on
  the Lutetia recipe; `--out` plays it (0.3.0 and 0.4.0).
- **"About ninety" operations** (`docs/creating-objects.md`): 0.3.0 lists 79, 0.4.0 lists 83.
- **`extract --select`** expects `{"manifest": 1, "chosen": [uri, ...]}`, a format no page
  documents ("the JSON the front end writes").
- **`extract` next to its source** rewrites the source's `.h5` when the output is in the same
  folder (the arrays go into a file of the same name); the help only says the datasets are
  copied "beside the new package". 0.4.0's `docs/cli.md` says `extract` "never writes onto the
  package it reads": it refuses the source `.epc` as output, and still rewrote the source
  `.h5` in the same folder.
- **`h5 --fix`** ("write a package whose declarations name the files the search found")
  rewrote 735 of Lutetia's 738 pointers and left the 3 WITSML channel-data pointers on the old
  name; `rename` rewrote all 738 (0.3.0 and 0.4.0; Python `repair_arrays` 0.4.0 likewise, with
  `lost: 0`).
- **Python `check(schema=False)`**: `docs/python.md` says it returns `checked: False`; the
  flag is in `result["schema"]["checked"]`.
- **Python `uri_of`**: its docstring says it raises when a uuid is held under several
  versions; with one unversioned and one `version='2'` it returned the unversioned URI (0.3.0
  and 0.4.0).
- **Python `rename_data_file`**: `docs/python.md` says the operation "refuses as a whole rather
  than doing part"; it moves the file at once and rewrites the package only at `save_as`
  (0.3.0 and 0.4.0).
- **Metadata keys that match nothing**: `docs/library.md` says such a path "yields no key";
  GeoJSON writes it as `null` (the CLI says so).
- **`fetch` quote**: `docs/data-sources.md` says `fetch` quotes first and downloads only when
  told to; the quote itself sends a counting request (`resultType=hits`) to the service.

## Upstream pages worth reading

`docs/troubleshooting.md` (HDF5 builds on Windows, `DLL load failed`, namespace packages),
`docs/house-rules.md` (`energyml rules --house`), `docs/colour-inheritance.md` (why an object
has a colour), `docs/section-view.md` (`profile`), `docs/creating-objects.md` and `docs/adr/`
(why the library refuses rather than guesses).
