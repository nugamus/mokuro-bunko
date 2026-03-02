# Mokuro Bunko

A self-hosted manga library server with WebDAV, built-in OCR processing, and multi-user support. Designed as a backend for [Mokuro Reader](https://reader.mokuro.app).

> [!WARNING]
> **v0.1 -- Early alpha.** Core functionality works but many features are untested or incomplete. Expect rough edges. No binary releases or Docker images are published yet -- run from source for now.

## What it does

- Serves a shared manga library over WebDAV so Mokuro Reader can connect directly
- Tracks per-user reading progress (each user gets their own progress files transparently)
- Runs [mokuro](https://github.com/kha-white/mokuro) OCR automatically on uploaded manga (CUDA, ROCm, or CPU)
- Manages users with role-based permissions (anonymous browse, registered, uploader, editor, admin)
- Provides a web catalog UI for browsing the library and an admin panel for user/config management

## Changelog

### 0.1.1

- **Fix: 503 Service Unavailable on Windows.** Cheroot worker threads can die from unhandled Windows socket errors ([cheroot#375](https://github.com/cherrypy/cheroot/issues/375), [cheroot#710](https://github.com/cherrypy/cheroot/issues/710)), eventually leaving zero threads to process requests. Added a thread pool watchdog that detects dead threads and replaces them, plus a resilient serve loop that recovers from the interrupt flag a dying thread sets.
- **Fix: Windows compatibility.** `os.rename()` replaced with `os.replace()` for atomic file moves (Windows fails if the destination exists). `os.umask()`/`os.chmod()` guarded on Windows where they have no effect on NTFS.
- **Fix: SQLite concurrency.** Enabled WAL journal mode, `busy_timeout`, and longer connection timeout to prevent database locking under concurrent access.
- **Audit logging for account self-deletion.** Deleting your own account now logs a `self_delete_account` audit event before removal.
- **Soft-delete users.** Deleting a user sets status to `deleted` instead of removing the row, preserving audit trail and upload ownership records.
- **30-day audit log retention.** Audit entries older than 30 days are automatically pruned.
- **Debug request logging.** Set `MOKURO_DEBUG=1` to log every request with thread name, method, path, status code, and timing to stderr.

### 0.1.0

- Initial release.

## Quick start

```bash
git clone https://github.com/Gnathonic/mokuro-bunko.git
cd mokuro-bunko
uv sync
uv run mokuro-bunko setup   # interactive first-time config
uv run mokuro-bunko serve
```

Requires Python 3.10+ and [uv](https://docs.astral.sh/uv/).

## Configuration

On first run, `mokuro-bunko setup` walks you through creating an admin account and writing a config file. After that, edit `config.yaml` directly or use the admin panel at `/_admin`.

Copy [`config.example.yaml`](config.example.yaml) for a documented starting point. Key settings:

| Setting | Default | Description |
|---------|---------|-------------|
| `server.port` | `8080` | Listen port |
| `storage.base_path` | `~/.local/share/mokuro-bunko` | Library and database location |
| `registration.mode` | `self` | `disabled`, `self`, `invite`, or `approval` |
| `ocr.backend` | `auto` | `auto`, `cuda`, `rocm`, `cpu`, or `skip` |
| `catalog.enabled` | `false` | Web-based library browser |

Environment variable overrides: `MOKURO_HOST`, `MOKURO_PORT`, `MOKURO_STORAGE`, `MOKURO_CONFIG`.

## OCR

Mokuro Bunko manages an isolated Python environment for OCR dependencies (PyTorch + mokuro). This keeps the heavy ML stack separate from the server itself.

```bash
mokuro-bunko install-ocr                # auto-detect best backend
mokuro-bunko install-ocr --backend cuda # force a specific backend
mokuro-bunko install-ocr --list-backends # show what's available
```

When OCR is enabled, the server watches for new uploads and processes them in the background. Results (`.mokuro` overlay files and `.webp` thumbnails) are placed alongside the source volumes.

The installer manages Python packages only -- CUDA/ROCm drivers must be installed on the host.

## User roles

| Role | Browse | Download | Upload | Edit/Delete | Invite | Admin |
|------|--------|----------|--------|-------------|--------|-------|
| Anonymous | configurable | configurable | -- | -- | -- | -- |
| Registered | yes | yes | -- | -- | -- | -- |
| Uploader | yes | yes | yes | own uploads | -- | -- |
| Editor | yes | yes | yes | all | -- | -- |
| Inviter | yes | yes | yes | all | yes | -- |
| Admin | yes | yes | yes | all | yes | yes |

Roles are a strict hierarchy: Admin > Inviter > Editor > Uploader > Registered > Anonymous. Each role inherits all capabilities of the roles below it.

## CLI reference

```
mokuro-bunko serve          # start the server
mokuro-bunko setup          # first-time setup wizard
mokuro-bunko install-ocr    # install/reinstall OCR environment
mokuro-bunko admin          # user management (create, delete, list, set-role)
mokuro-bunko config         # view/edit config
mokuro-bunko ssl            # manage SSL certificates
mokuro-bunko tunnel         # cloudflare tunnel management
mokuro-bunko dyndns         # dynamic DNS management
```

## Development

```bash
uv sync --extra dev
uv run pytest               # run tests
uv run ruff check src tests # linting
uv run mypy src             # type checking
```

## License

[Mozilla Public License 2.0](LICENSE)
