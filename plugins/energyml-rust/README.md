# energyml-rust — a Claude Code plugin for EPC packages

This plugin teaches Claude to read, check, analyse, export and write EnergyML data — RESQML,
WITSML and PRODML objects in EPC packages, with their HDF5 files — through
[`energyml-rust`](https://github.com/geosiris-technologies/energyml-rust), an open-source
library with a command line (`energyml`) and a Python package (`energyml-rust`, imported as
`energyml.rust`). Ask in plain words: "what is in this EPC", "check this package", "export
these surfaces to OBJ", "read the well logs", "the h5 file was renamed", "make a copy with only
the wells", "create an EPC from this CSV".

It holds one skill, `epc-handler`, and its reference pages: text only, no script, no hook, no
MCP server and no agent of its own.

`energyml-rust` and this plugin are developed by Geosiris. They are **not affiliated with or
endorsed by Energistics or The Open Group**. Energistics, RESQML, WITSML and PRODML are
trademarks of Energistics Consortium, Inc.; "EnergyML" is used as the common name of the family
of standards.

## What it needs

- The `energyml` command line, 0.3.0 recommended (the skill was written and tested against
  0.3.0), built with HDF5 to read arrays: the GitHub release archives for Linux (gnu), macOS
  and Windows are; the Linux musl archives are not. The skill checks `energyml --version` first
  and, if the command is missing, offers to install it — only after you agree.
- Optional: the Python package `energyml-rust` (for scripts, numpy arrays, copying objects
  between packages), and `energyml-mcp` (to serve one package to Claude as read-only MCP
  tools; it is not in the release archives).
- Your EPC files, each with its `.h5` file beside it.

## What it runs, sends and fetches

- **It runs `energyml` (and, when you prefer a script, `python` with `energyml.rust`) in your
  shell**, on the files you name. Reading, checking and exporting read your files and write
  nothing, or write only the new files you asked for.
- **Your originals are never written over.** Everything written goes to a new file in a new
  folder. Exports you ask for (OBJ, GeoJSON, CSV, VTK, SVG) are written straight away; anything
  that writes or rewrites a package or its data file (extract, rename, repair, relabel,
  `h5 --fix`, create, import, Python `save_as`) is shown first as a plan or dry run, and the
  skill waits for your explicit yes. Some `energyml` writes touch the files beside their output
  (an `.h5` of the same name, the data file a rename moves), so the skill works on copies in a
  folder of their own.
- **Nothing leaves your machine during normal use.** `energyml` and `energyml.rust` read and
  write local files. Two commands contact public services, and the skill runs them only when you
  ask for them: `energyml fetch` (the BRGM borehole web service, `data.geoscience.fr`; even its
  quote sends a counting request) and `energyml terrain` (IGN elevation services).
- **Choosing how to run it:** the skill first uses whatever already works (`energyml`, the
  Docker image, the Python package); if nothing does, it asks you which way you prefer.
- **Installing, only if you agree:** the skill can download a release archive and its
  `SHA256SUMS` from the GitHub releases of `geosiris-technologies/energyml-rust`, install
  nothing unless the checksum matches, and put `energyml` in a folder of yours; or run
  `pip install energyml-rust` (from PyPI) or `cargo install energyml-cli --features hdf5` /
  `cargo install energyml-mcp-cli --features hdf5-static` (from crates.io), in a user folder or a
  virtual environment; or pull the public Docker image
  `ghcr.io/geosiris-technologies/energyml` (from the GitHub container registry) and run it on
  a folder you choose, mounted at `/work`. Never with `sudo`.
- **Nothing else.** The plugin collects no telemetry and sends nothing to Geosiris, Anthropic or
  any other party. What a file contains (titles, descriptions, error messages) is treated as
  data, never as instructions.

## Install

```sh
claude plugin marketplace add geosiris-technologies/claude-plugins
claude plugin install energyml-rust@geosiris-technologies
```

`energyml-rust` itself, its documentation and its releases:
<https://github.com/geosiris-technologies/energyml-rust>. A public test package, Lutetia, is at
<https://github.com/geosiris-technologies/energyml-lutetia>.

## License

Apache License 2.0, like the rest of `energyml-rust`. Not affiliated with or endorsed by
Energistics or The Open Group.
