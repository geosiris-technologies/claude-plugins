# Geosiris plugins for Claude Code

Claude Code plugins from [Geosiris](https://www.geosiris.com) for subsurface and energy data.

| Plugin | What it does |
|---|---|
| [`retp`](plugins/retp) | Work with ETP v1.2 servers (RESQML, WITSML, PRODML, OSDU Reservoir DDMS) through the [`retp`](https://github.com/geosiris-technologies/retp) command line: probe, list, read, download as EPC, check conformance, serve a dataspace over MCP. Credentials by reference only, writes only with consent. |
| [`energyml-rust`](plugins/energyml-rust) | Read, check, analyse, export and write EnergyML data (RESQML, WITSML, PRODML) in EPC packages and their HDF5 files through the [`energyml-rust`](https://github.com/geosiris-technologies/energyml-rust) command line and Python package: list, show, check, repair, export to OBJ/GeoJSON/CSV/VTK, well logs and series, extract, rename the data file, create from CSV or GeoJSON. Originals never overwritten, writes only after a plan and your yes. |

## Install

```sh
claude plugin marketplace add geosiris-technologies/claude-plugins
claude plugin install retp@geosiris-technologies
claude plugin install energyml-rust@geosiris-technologies
```

Each plugin's README says what it needs, what it runs and where it connects.

## License

Apache License 2.0, see [LICENSE](LICENSE). Not affiliated with or endorsed by Energistics or
The Open Group.
