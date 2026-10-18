# Checking and repairing a package

## `energyml check`

```sh
energyml check FILE.epc                          # every section, at most 50 findings each
energyml check FILE.epc --errors-only --max 1000 # errors only, the full list
energyml check FILE.epc --no-schema              # skip schema validation (the slow section)
energyml check FILE.epc --rules mine.json        # add a business rule set to the shipped one
```

Exit code `1` when it finds at least one error, `0` otherwise, `2` when it could not run.
Python gives the same numbers: `Epc.open(p).check()` →
`{"errors", "warnings", "sound", ...}`.

### Its sections

The report starts with `== FILE ==` and then, in order:

| Section | What it says | Counts as |
|---|---|---|
| `package:` | the zip, content types, relationships, nil uuids | errors/warnings as marked |
| `objects: N read, M unreadable` | parts that do not parse (see `repair`) | each unreadable part is an error |
| `schema:` | schema validation of every object | issues |
| `properties:` | property kinds (abstract kind used, unknown kind), units against kinds | issues, **and notes** |
| `property counts:` | values held against values declared (needs HDF5) | errors and `[remark]`s |
| `business rules: N violation(s) over 31 rule(s)` | rules from the technical guides (`trg/...`) and house rules (`house/...`), each with `[error]` or `[warning]` and the document it cites | errors/warnings as marked |
| `references:` | references to objects the package does not hold | issues |
| `references to the Energistics catalogue: N` | well-known `PropertyKind`s not carried, **expected** | nothing |
| `relationships:` | `.rels` disagreeing with the objects | warnings |
| last line | `N error(s), M warning(s)` | the verdict |

### How to summarise it

1. The verdict line, verbatim, and the exit code.
2. The section headers with their counts (`grep -E '^[a-z].*:|error\(s\)'` on the output).
3. Findings grouped by rule id or kind, errors before warnings: rule, how many objects, one
   example with its uuid and title, the document it cites. Not one line per finding.
4. The notes that are **not** findings, counted in one sentence each: "was not checked: this
   build converts four families of unit" (units the build cannot check), "not checkable: its
   lookup has 0 key column(s)", "(249 that could not be counted, so not checked)". Lutetia's
   clean package has 30 + 449 such notes and `0 error(s), 0 warning(s)`.

On `package-with-errors/Lutetia.epc` (0.3.0, 0.65 s) the verdict is `8 error(s), 1 warning(s)`:
two abstract property kinds (`property`, `force per length`, in `properties:`), and seven
business-rule violations: `trg/contact-element-is-geological`,
`house/discrete-value-in-lookup`, `trg/one-definitive-trajectory-per-wellbore`,
`trg/is-listric-iff-normal`, `trg/homogeneous-collection`,
`trg/wellbore-interpretation-set-holds-wellbores` (errors) and
`house/time-series-asset-resolves` (warning). That is what the variant's
`docs/errors.md` says it should find; the check reports 1 342 objects where the inventory says
1 341 (the second version of one object is counted).

### What `check` does not tell you

- **Whether the `.h5` is there.** With the data file deleted, `check` still ends `0 error(s)`
  and exits 0 (its property counts are silently skipped). Run `energyml h5 FILE.epc` too.
- Whether the geometry evaluates: `energyml geometry FILE.epc | tail` lists what is not
  evaluated, by name.
- A house rule (`house/...`) is the library's reading where the specification is silent;
  `energyml rules --house --id ID` gives its text and reason. Say "house rule" when you report
  one.

## `energyml repair` — parts no parser reads

```sh
energyml repair FILE.epc                          # list them, with the parser's reason
energyml repair FILE.epc --extract parts/         # their bytes, under their part names
# ... the user (or you, with their agreement) fixes the files in parts/ ...
energyml repair FILE.epc --apply parts/           # says, per part, whether it now parses; writes nothing
energyml repair FILE.epc --apply parts/ --out NEW/fixed.epc
```

```text
1 unreadable part(s) in broken.epc:
  BoundaryFeature_74d880d4-….xml  (14 byte(s))
    eml:///resqml22.BoundaryFeature(74d880d4-…)
    XML error: syntax error: tag not closed: `>` not found before end of input
```

`repair` exits 1 when there are unreadable parts. A clean package answers `every part of
FILE reads as an object`. `--force` stages a part that still does not parse: do not use it
unasked. Fixing the bytes is editing someone else's data: show the change and get a yes.

## Arrays not found: `h5`, `--h5`, `h5 --fix`

```sh
energyml h5 FILE.epc                                    # where each data file is, exit 1 if not found
energyml --h5 /path/renamed.h5 geometry FILE.epc        # offer a file for this run
energyml --h5 old.h5=/path/renamed.h5 export FILE.epc ... # pin a path to the name the package uses
energyml --h5 COPY/renamed.h5 h5 COPY/FILE.epc --fix NEW/fixed.epc   # write the new name down
```

Measured: a renamed `.h5` that is the only HDF5 file in the package's folder is found by
itself (`[another file in the package's folder]`). A file elsewhere (a `data/` subfolder) or
nowhere gives `NOT FOUND`, and every read refuses with `is in none of the 0 candidate file(s)`.
Ask the user where the file is; do not guess another `.h5`.

**To write the new name into the package, prefer `rename` to `h5 --fix`.** On Lutetia,
`h5 --fix` rewrote 660 declarations (735 of the 738 pointers) and left the 3 WITSML
channel-data pointers (`/witsml21/.../ChannelData`) on the old name; they kept working only
because the folder search found the file. `rename` rewrote all 738, and it accepts an old name
whose file is gone:

```sh
mkdir W && cp FILE.epc W/ && cp path/to/renamed.h5 W/       # side by side: names are bare, no subfolder
energyml rename W/FILE.epc --from OLD.h5 --to renamed.h5    # the quote: 738 field(s) in 663 object(s)
energyml rename W/FILE.epc --from OLD.h5 --to renamed.h5 --apply --out W/FILE-fixed.epc
#   no file was found at `OLD.h5`; the package now names `renamed.h5` all the same
energyml h5 W/FILE-fixed.epc                                # renamed.h5  738 pointer(s) → W/renamed.h5
```

`--to` is a bare name: `--to data/x.h5` is refused. The `.h5` must sit beside the `.epc`; a
subfolder layout can only be served with `--h5 PATH` on each command. To rename a file on
purpose, use `rename` the same way (see [writing-and-creating.md](writing-and-creating.md)),
never `mv` alone.

## `energyml relabel` — a wrong schema version

```sh
energyml relabel FILE.epc --package resqml --schema-version 2.2_dev3            # what it would change
energyml relabel FILE.epc --package resqml --schema-version 2.2_dev3 --from 2.2 -o NEW/relabelled.epc
```

For a package that declares a release but holds a development drop (or the reverse). It does
not read the objects; it rewrites the declared version. `--from` is required when the package
holds several versions of that standard. Run it only when the user knows which version the
producer really wrote: relabelling a correct package corrupts it.

## `--lenient` and `--strict`

`--lenient` reads past an absent required element or an unreadable value and reports each
replacement: good to see what a broken producer wrote, never to save. `--strict` refuses a part
that omits something required: use it when the objects will be written back. Malformed XML is
refused in both modes; that is `repair`'s job.
