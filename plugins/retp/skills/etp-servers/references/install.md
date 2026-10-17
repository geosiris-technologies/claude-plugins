# Installing `retp` and `retp-conformance`

Ask the user before installing anything, and say where it will go. Check first:
`retp --version` (and `retp-conformance --version`). A version below 0.3.0 has no server
catalogue; see the table in `SKILL.md`.

## The release archives (recommended)

Each [GitHub release](https://github.com/geosiris-technologies/retp/releases) carries one
archive per platform, `retp-vX.Y.Z-<target>.tar.gz` (`.zip` on Windows), and a `SHA256SUMS`
file. Targets: `x86_64`/`aarch64` × `unknown-linux-gnu`, `unknown-linux-musl`, `apple-darwin`,
`pc-windows-msvc`. The archives are built with `mcp` and the shell's line editor; all but the
musl ones are built with HDF5 (the musl ones cannot write arrays in `download-epc`). **They
contain `retp` only, not `retp-conformance`.**

Install from an archive with the user's agreement, and only after checking it:

1. Say which archive fits the machine (`uname -sm` on Linux and macOS; on Windows,
   `$env:PROCESSOR_ARCHITECTURE`) and where `retp` will go (a folder of the user, such as
   `~/.local/bin` or `%LOCALAPPDATA%\Programs\retp`). Never a system folder, never `sudo`.
2. Download the archive **and** the release's `SHA256SUMS` into a new temporary folder.
3. Compare the archive's checksum with its line in `SHA256SUMS`: `sha256sum ARCHIVE`
   (`shasum -a 256 ARCHIVE` on macOS, `Get-FileHash -Algorithm SHA256 ARCHIVE` in
   PowerShell). On a mismatch or a missing line, stop and tell the user; install nothing.
4. Extract the archive and copy only `retp` (`retp.exe`) into the chosen folder. Do not
   replace an existing `retp` without asking. Change the PATH only if the user asks.
5. Run `retp --version` from that folder to confirm.

If the user prefers to do it themselves, give them the release page and these steps.

## Cargo

```sh
cargo install retp-cli --features mcp           # add hdf5 when libhdf5 is installed
cargo install retp-conformance
```

`--features hdf5` needs the HDF5 C library at build time (`libhdf5-dev` on Debian/Ubuntu);
without it `download-epc` refuses to write arrays and says so (`--no-arrays` still works).

## Docker

```sh
docker pull ghcr.io/geosiris-technologies/retp:latest     # or a version tag: 0.3.0, 0.3
```

The image has `retp` (with HDF5 and, from 0.3.0, `mcp`) as its entry point and
`retp-conformance` beside it. It runs as uid 1000 in `/work`: mount a folder there for
anything that writes files.

```sh
IMG=ghcr.io/geosiris-technologies/retp:latest
docker run --rm $IMG dataspaces wss://etp.example.com/etp
docker run --rm -v "$PWD:/work" $IMG download-epc wss://etp.example.com/etp demo --out demo.epc
docker run --rm -v "$PWD:/work" --entrypoint retp-conformance $IMG \
    wss://etp.example.com/etp --out report.md
```

PowerShell writes `-v "${PWD}:/work"`. On Linux, if the container cannot write in the folder,
add `--user "$(id -u):$(id -g)"`. Credentials in a container: see
[authentication.md](authentication.md#docker) and [catalogue.md](catalogue.md#in-docker).
