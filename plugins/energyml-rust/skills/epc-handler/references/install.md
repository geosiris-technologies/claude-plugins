# Installing `energyml`, `energyml-rust` (Python) and `energyml-mcp`

Ask the user before installing anything, say what and where, and install into a folder of
theirs or a virtual environment: never a system folder, never `sudo`. Check first:
`energyml --version`, `python -c "import energyml.rust"` (in the environment they use),
`energyml-mcp --help`.

## The command line from a release archive (recommended)

Each [GitHub release](https://github.com/geosiris-technologies/energyml-rust/releases) carries
one archive per platform, `energyml-vX.Y.Z-<target>.tar.gz` (`.zip` on Windows), and a
`SHA256SUMS` file. Targets of 0.3.0: `x86_64`/`aarch64` × `unknown-linux-gnu`,
`unknown-linux-musl`, `apple-darwin`, `pc-windows-msvc`. Each archive holds `energyml`
(`energyml.exe`), `README.md`, `LICENSE` and `NOTICE`; **not** `energyml-mcp`, and not the
`enyml` alias the upstream README mentions.

- The **gnu, macOS and Windows** archives are built with HDF5 linked in: they read arrays with
  nothing else installed (checked on Linux x86_64: the binary needs only libc, libm, libgcc).
- The **musl** archives are built **without HDF5**: they list and check, but every array read
  is refused ("no support for `application/x-hdf5`"). Prefer gnu unless the machine has no
  glibc.
- No archive of 0.3.0 has the `png` feature (`profile --png` is refused; `--svg` works).

Steps, with the user's agreement:

1. Pick the archive (`uname -sm` on Linux and macOS; `$env:PROCESSOR_ARCHITECTURE` in
   PowerShell) and the folder `energyml` will go to (`~/.local/bin`,
   `%LOCALAPPDATA%\Programs\energyml`). Say both.
2. Download the archive **and** `SHA256SUMS` of the same release into a new temporary folder:

   ```sh
   V=v0.3.0; T=x86_64-unknown-linux-gnu
   B=https://github.com/geosiris-technologies/energyml-rust/releases/download/$V
   mkdir -p ~/tmp/energyml-$V && cd ~/tmp/energyml-$V
   curl -fsSLO "$B/energyml-$V-$T.tar.gz" && curl -fsSLO "$B/SHA256SUMS"
   ```

3. Check it: `sha256sum -c --ignore-missing SHA256SUMS` must print `...: OK` for the archive
   (`shasum -a 256 -c --ignore-missing SHA256SUMS` on macOS; in PowerShell compare
   `(Get-FileHash -Algorithm SHA256 ARCHIVE).Hash` with the archive's line). On a mismatch or a
   missing line, stop and tell the user; install nothing.
4. Extract, and copy only the `energyml` binary to the chosen folder. Do not replace an
   existing `energyml` without asking. Change the PATH only if the user asks.
5. Confirm: `energyml --version` prints `energyml 0.3.0`.

macOS may quarantine a downloaded binary; that is the user's call (System Settings), not
something to bypass silently.

## Python: `pip install energyml-rust`

```sh
python3 -m venv ~/.venvs/energyml && ~/.venvs/energyml/bin/pip install energyml-rust==0.3.0
~/.venvs/energyml/bin/python -c "from energyml.rust.epc import Epc; print('ok')"
```

The distribution is `energyml-rust`, the import `energyml.rust`. Wheels exist for Linux
(manylinux and musllinux, x86_64 and aarch64), macOS and Windows, Python 3.9 or later (`abi3`),
with HDF5 inside: no `h5py`, no HDF5 install, no Rust needed. Versions on PyPI: 0.1.2, 0.2.0,
0.3.0. It needs numpy (installed with it). It installs under `energyml/rust/` and does not
overwrite the other `energyml.*` distributions.

## Cargo (builds from source)

```sh
cargo install energyml-cli --version 0.3.0 --locked --features hdf5          # needs libhdf5-dev
cargo install energyml-mcp-cli --version 0.3.0 --locked --features hdf5-static  # builds HDF5 itself (CMake, a C compiler)
```

`--features hdf5` needs the HDF5 C library and headers at build time (`libhdf5-dev` on
Debian/Ubuntu); without the feature, array reads are refused by name. `hdf5-static` compiles
HDF5 from source and needs CMake and a C compiler. A full build takes several minutes and
about 2 GB of disk; warn the user. Add `--root DIR` to install into `DIR/bin`.

## Docker

```sh
docker pull ghcr.io/geosiris-technologies/energyml:0.3.0     # tags: 0.3.0, 0.3, latest, edge
docker run --rm -v "$PWD:/work" ghcr.io/geosiris-technologies/energyml:0.3.0 list model.epc
docker run --rm -i -v "$PWD:/work" ghcr.io/geosiris-technologies/energyml:0.3.0 energyml-mcp model.epc
```

The image is public (no login). It holds `energyml` with HDF5 **and PNG**, `energyml-mcp`
and the `enyml` alias; it runs as uid 1000 in `/work`. Checked with Docker Desktop on Windows,
on Lutetia: the same output as the 0.3.0 Windows archive (`h5`, `profile --svg`), about 2 s per
command instead of 0.3 s, and `profile --png` works there while the archives refuse it.

- The `.epc` and its `.h5` must both be inside the mounted folder; give paths relative to it.
  Outputs are written into it, owned by uid 1000 on Linux (add `--user "$(id -u):$(id -g)"` if
  the user cannot then edit them).
- **PowerShell**: `-v "${PWD}:/work"`. **Git Bash on Windows** rewrites paths: prefix the line
  with `MSYS_NO_PATHCONV=1` and mount `"$(pwd -W):/work"`, or Docker sees a mangled path.
- `--rm` leaves nothing behind but the image; `docker image rm ghcr.io/geosiris-technologies/energyml:0.3.0`
  removes it.

## After installing

`energyml --version`, then `energyml list` on one of the user's files. If an array read is
refused with "build with the `hdf5` feature", the binary has no HDF5: see above.
