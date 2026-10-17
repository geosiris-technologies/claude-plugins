# Geosiris plugins for Claude Code

Claude Code plugins from [Geosiris](https://www.geosiris.com) for subsurface and energy data.

| Plugin | What it does |
|---|---|
| [`retp`](plugins/retp) | Work with ETP v1.2 servers (RESQML, WITSML, PRODML, OSDU Reservoir DDMS) through the [`retp`](https://github.com/geosiris-technologies/retp) command line: probe, list, read, download as EPC, check conformance, serve a dataspace over MCP. Credentials by reference only, writes only with consent. |

## Install

```sh
claude plugin marketplace add geosiris-technologies/claude-plugins
claude plugin install retp@geosiris-technologies
```

Each plugin's README says what it needs, what it runs and where it connects.

## License

Apache License 2.0, see [LICENSE](LICENSE). Not affiliated with or endorsed by Energistics or
The Open Group.
