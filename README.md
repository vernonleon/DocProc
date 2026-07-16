# DocProc

#Of course I have no idea what I am doing and am still trying to figure out how to upload the project...

DocProc is a local-first desktop application for converting PDF records into a
single Markdown document with an LM Studio vision model. It rasterizes PDF pages
to PNG, lets the operator confirm logical record boundaries, sends every record
as a fresh OpenAI-compatible request, and assembles successful responses in
source order.

DocProc is designed to minimize PHI exposure, but installing or using it does
not by itself make a workflow HIPAA compliant. Your organization remains
responsible for access controls, encrypted storage, endpoint authorization,
retention, backups, risk analysis, and any required BAA.

## Current Features

- Native desktop interface; no local web server or browser secret handling
- Numerical `pages per record` generation for repeat document formats
- Thumbnail review with manual record-start adjustments
- LM Studio model discovery through `GET /v1/models`
- OpenAI-compatible multimodal `POST /v1/chat/completions`
- Independent request per logical record; no conversation or response IDs
- Ordered final assembly even when requests complete out of order
- Per-record status and targeted retries for failed records
- Live per-record streaming preview while LM Studio is generating
- OS-keyring API token storage, with session-memory use if storage is unavailable
- Loopback/RFC1918/IPv6-ULA endpoint allowlist by default
- Typed, per-run confirmation for a public endpoint
- LM Studio-supported sampling controls
- No `max_tokens` or `max_completion_tokens` field sent by DocProc
- Detection of LM Studio responses that finish because of a length limit
- Automatic atomic output to `<output folder>/<PDF stem>_processed.md`
- Owner-only temporary workspaces and output file permissions
- Temporary PNG cleanup after success, failure, or cancellation
- Startup cleanup of temporary data left by an interrupted prior run

## AppImage

The polished x86_64 AppImage bundles Python and DocProc's application
dependencies. It targets glibc 2.34 and newer, including Ubuntu 22.04+, Debian
12+, Fedora 39+, and current Arch-family systems.

```bash
chmod +x DocProc-0.1.1-x86_64.AppImage
./DocProc-0.1.1-x86_64.AppImage
```

Verify a downloaded artifact before running it:

```bash
sha256sum -c DocProc-0.1.1-x86_64.AppImage.sha256
```

On a system without FUSE:

```bash
APPIMAGE_EXTRACT_AND_RUN=1 ./DocProc-0.1.1-x86_64.AppImage
```

The default automatic output directory is `~/Documents/DocProc Outputs/`.
Settings and temporary data continue to use the standard XDG user directories;
nothing is written inside the read-only AppImage mount.

## Source Requirements

- Linux with Python 3.12 or newer
- LM Studio with a vision-capable model
- A desktop session supported by Qt
- An OS keyring backend for persistent token storage

On Linux, `keyring` normally uses the desktop Secret Service through GNOME
Keyring or KWallet. If a secure backend is unavailable, DocProc does not fall
back to a plaintext token file. A token entered in the UI remains usable for the
current process only.

## Installation

Create an isolated environment and install the pinned dependencies:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.lock
```

For development and tests:

```bash
.venv/bin/python -m pip install -r requirements-test.lock
```

For a disconnected workstation, download and review the wheels on an approved
connected machine, transfer the wheelhouse through your normal software-control
process, then install with `--no-index --find-links <wheelhouse>`.

## Run

Start LM Studio's local server, load a vision-capable model, then run:

```bash
.venv/bin/python app.py
```

The default endpoint is `http://127.0.0.1:1234/v1`.

## Workflow

1. Choose a PDF and load or paste a Markdown guide.
2. Enter the known number of pages per record and select **Generate Numerical
   Ranges**.
3. Review the generated ranges and thumbnails.
4. Double-click a thumbnail, or select it and use **Toggle Start at Selected
   Page**, to adjust irregular boundaries.
5. Connect to LM Studio and choose a model.
6. Run the isolated record requests.
7. Retry failed records as needed.
8. After every record succeeds, DocProc atomically writes the assembled file to
   the configured automatic output folder.
9. Review the in-app preview or use **Save Final Markdown** for an additional
   copy at another location.

If the last generated range is shorter than the configured pages-per-record
value, DocProc requires explicit confirmation before processing.

## LM Studio Model Selection

On startup, DocProc requests the available model IDs from the configured
`GET /v1/models` endpoint. The model field is a non-editable selector containing
only IDs returned by that API. Use **Connect / Refresh Models** after changing
the endpoint or loading another model in LM Studio. A refresh preserves the
current selection only if the API still reports that exact ID. On application
startup, DocProc does not restore an old selection or automatically select the
first reported model; the operator must make an explicit selection for the new
session.

The bundled LM Studio profile sends only sampler fields currently documented by
LM Studio and does not inject model-specific prompt switches.

`/v1/models` may report downloaded models that are not currently loaded. If a
record fails with HTTP 500 and succeeds after manually loading the model, LM
Studio failed while loading or executing that model. Load the selected
vision-capable model in LM Studio and use **Retry Failed Records**. LM Studio's
server logs contain the underlying engine error. DocProc does not call LM
Studio's model-load endpoint, but a Chat Completions request may cause LM Studio
to JIT-load its selected model. Requiring explicit selection prevents an
arbitrary first model from being requested.

During processing, streamed output appears in separate live record sections in
the preview. With concurrent records, those sections remain in record order;
raw chunks are never interleaved. Streaming does not change completion rules:
DocProc writes the final file only after every record succeeds.

DocProc sends supported values only:

- `temperature`
- `top_p`
- `top_k`
- `presence_penalty`
- `frequency_penalty`
- `repeat_penalty`

It deliberately does not send `min_p`, `repetition_penalty`, `max_tokens`, or
`max_completion_tokens` under the bundled LM Studio profiles.

## Network Policy

Normal mode accepts only:

- IPv4 loopback
- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`
- IPv6 loopback
- IPv6 unique-local addresses (`fc00::/7`)

Private network destinations must be entered as literal IP addresses. The sole
hostname exception is `localhost`, which DocProc pins to `127.0.0.1`. This avoids
a second DNS lookup redirecting an approved private hostname to a public
address. Mixed private/public DNS answers are treated as public. Public
destinations must use HTTPS. HTTP redirects, URL credentials, query strings,
fragments, and proxy environment variables are not accepted for endpoint
requests.

Public routing is available because it was an explicit product requirement. It
requires enabling the per-run control and typing `SEND PHI OFF PREMISES` for
each run. That confirmation is a safety interlock, not proof that disclosure is
authorized or compliant.

## Data Handling

Persisted settings contain only the endpoint, numerical range defaults,
rendering settings, output directory, and sampler values. They do not contain
the guide, PDF path, source filename, API token, page images, model selection,
or model output.

During processing, rendered pages live under the user's platform cache directory
in a randomly named `0700` run directory. Files use `0600` permissions. DocProc
removes that directory when the run terminates and removes abandoned run
directories on the next startup.

At run start, DocProc hashes the selected PDF and creates an owner-only,
hash-verified temporary snapshot. Rendering and retries use that immutable run
snapshot so records from changed source files cannot be mixed. Intermediate
model responses stay in process memory. The snapshot and rendered pages are
removed when the run ends; only the final Markdown selected by the operator is
retained. Active workspaces hold an OS file lock, allowing the next startup to
remove abandoned crash data immediately without deleting another live
instance's pages.

Secure erasure cannot be guaranteed on SSDs, copy-on-write filesystems, or
snapshotted storage. Use encrypted local storage and an approved workstation
retention policy.

## Rasterization

PNG encoding is lossless, but rasterizing vector PDF content is not a lossless
conversion of the original PDF. The default is 200 DPI; 300 DPI and higher can
substantially increase memory, base64 request size, vision-token usage, and
processing time.

## Tests

```bash
.venv/bin/python -m pytest -q
QT_QPA_PLATFORM=offscreen .venv/bin/python -c \
  "from PySide6.QtWidgets import QApplication; from docproc.ui.main_window import MainWindow; app=QApplication([]); w=MainWindow(); print(w.windowTitle()); w.close()"
```

Tests use generated, non-PHI PDFs and mocked LM Studio responses.

The packaged executable also has an offline diagnostic that does not contact LM
Studio or process user documents:

```bash
QT_QPA_PLATFORM=offscreen ./DocProc-0.1.1-x86_64.AppImage --self-test
```

## Reproducible AppImage Build

Docker is the only host build dependency. The build image, CPython runtime,
AppImage tool, AppImage runtime, Python dependencies, and checksums are pinned.

```bash
scripts/build-appimage.sh
```

Release outputs are written under `release/`:

- `DocProc-0.1.1-x86_64.AppImage`
- `DocProc-0.1.1-x86_64.AppImage.sha256`
- `DocProc-0.1.1-SBOM.cdx.json`
- `DocProc-0.1.1-build-info.json`

The build runs the complete test suite, packaged self-test, resource checks,
ELF dependency audit, and glibc ceiling check before producing the AppImage. It
also normalizes PyInstaller's standard-library archive and sets deterministic
filesystem timestamps; repeated clean builds produce the same AppImage SHA-256.

## License

DocProc is open source under `AGPL-3.0-only`. See `LICENSE` and
`THIRD_PARTY_NOTICES.md`. The AGPL license is compatible with the bundled
PyMuPDF distribution terms; Qt for Python is redistributed under its applicable
LGPL/GPL terms.
