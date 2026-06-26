<!-- generated-by: gsd-doc-writer -->
# Testing

This document covers how to run and extend the test suite for the Schellenberg USB integration. See [docs/DEVELOPMENT.md](DEVELOPMENT.md) for the broader local development setup.

## Why Tests Run in WSL Only

`homeassistant/runner.py` imports the Unix-only `fcntl` module. On native Windows this produces:

```
ModuleNotFoundError: No module named 'fcntl'
```

This is not a bug and must not be "fixed." The Home Assistant test infrastructure is Linux-only. All pytest invocations must go through WSL.

## Test Environment Overview

| Tool | Where it runs | Venv |
|------|---------------|------|
| `pytest` | WSL/Linux only, via `.wsl_exec.sh` | `.venv` (WSL/Linux venv on `/mnt/c`) |
| `ruff` | Native Windows | `.venv-win` |
| `mypy` | Native Windows | `.venv-win` |

## Running the Full Test Suite

### From PowerShell (standard)

```powershell
wsl -e env -u HOME -u WSLENV bash /mnt/c/Users/holger.rabbach/Coding/schellenberg_usb/.wsl_exec.sh "uv run pytest -p no:cacheprovider -q"
```

Substitute the absolute path to `.wsl_exec.sh` if your checkout is in a different location.

### From Git Bash (MSYS_NO_PATHCONV required)

The Git Bash shell applies MSYS path conversion and silently mangles `/mnt/c/...` into `C:/Program Files/Git/mnt/c/...`. The suite will appear to start but tests will never run. Always prefix `MSYS_NO_PATHCONV=1`:

```bash
MSYS_NO_PATHCONV=1 wsl -e env -u HOME -u WSLENV bash /mnt/c/Users/holger.rabbach/Coding/schellenberg_usb/.wsl_exec.sh "uv run pytest -p no:cacheprovider -q"
```

### Why `.wsl_exec.sh` Is Mandatory

The project `.venv` lives on `/mnt/c` (DrvFs) while uv's cache lives on WSL ext4 (`$HOME`). Hardlinks cannot span the two filesystems. Running `uv run pytest` directly causes uv to re-sync the entire environment on every call, and an interrupted re-sync corrupts the venv (symptoms: missing `homeassistant.helpers`, etc.).

`.wsl_exec.sh` exports `UV_LINK_MODE=copy` so the install completes in one pass and stays stable. It also fixes the WSL `HOME` variable, which is mangled to a Windows path when inherited from the Windows environment.

**Never invoke `uv` or `pytest` outside `.wsl_exec.sh` against this venv.**

### Venv Health Check

```powershell
wsl -e env -u HOME -u WSLENV bash /mnt/c/Users/holger.rabbach/Coding/schellenberg_usb/.wsl_exec.sh ".venv/bin/python -c 'import homeassistant.helpers, pytest'"
```

If this fails, repair the venv:

```powershell
wsl -e env -u HOME -u WSLENV bash /mnt/c/Users/holger.rabbach/Coding/schellenberg_usb/.wsl_exec.sh "uv sync --frozen"
```

## Running a Single Test File

Pass the file path as a pytest argument inside the quoted command string:

```powershell
wsl -e env -u HOME -u WSLENV bash /mnt/c/Users/holger.rabbach/Coding/schellenberg_usb/.wsl_exec.sh "uv run pytest tests/test_cover.py -p no:cacheprovider -q"
```

Run a specific test by node ID:

```powershell
wsl -e env -u HOME -u WSLENV bash /mnt/c/Users/holger.rabbach/Coding/schellenberg_usb/.wsl_exec.sh "uv run pytest tests/test_api.py::test_api_initialization -p no:cacheprovider -v"
```

## Test Organization

All tests live in `tests/`. The suite has approximately 207 tests across these files:

| File | What it covers |
|------|----------------|
| `test_api.py` | `SchellenbergUsbApi` initialization, device registration, connect/disconnect |
| `test_api_extended.py` | Edge cases for the API: retry logic, stick-busy handling, futures, error paths |
| `test_api_messages.py` | Protocol message parsing — frame decoding, device ID extraction, status messages |
| `test_const.py` | Constants and type aliases exported from `const.py` |
| `test_init.py` | `__init__.py` setup/teardown, subentry wiring, platform forwarding |
| `test_init_extended.py` | Edge cases for integration lifecycle: reload, subentry changes, error handling |
| `test_config_flow.py` | Blind subentry manual-add flow, serial port validation, config entry creation |
| `test_options_flow.py` | Hub options flow (`ignore_unknown` toggle, serial port change) |
| `test_cover.py` | `SchellenbergCover` entity: open/close/set position, calibration, position tracking |
| `test_sensor.py` | Sensor entities: stick connection status, firmware version, device mode |
| `test_switch.py` | `SchellenbergLedSwitch` entity: on/off commands, state reporting |
| `test_timed_calibration_flow.py` | Timed calibration flow (Phase 4): happy path, guard conditions (too short/too long) |
| `test_timed_cal_handler_structure.py` | RED-phase structural tests for `TimedCalibrationFlowHandler` interface and guard constants |

### Shared Fixtures (`conftest.py`)

- `mock_serial_port` — returns `/dev/ttyUSB0` as a fixture string
- `mock_config_entry_data` — config entry data dict keyed by `CONF_SERIAL_PORT`
- `mock_api` — fully mocked `SchellenbergUsbApi` with `AsyncMock` for async methods
- `mock_storage` — mocked `homeassistant.helpers.storage.Store`
- `mock_serial` — patches `serial.Serial` for config flow validation tests

### Test File Naming Convention

Test files follow `test_<module>.py` naming. For extended coverage of a module, a companion `test_<module>_extended.py` file is used. Structural or RED-phase tests use `test_<feature>_structure.py`.

## Test Configuration

Configuration lives in `pyproject.toml` under `[tool.pytest.ini_options]`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
norecursedirs = [".git"]
addopts = """
-n4
--strict-markers
--cov=custom_components"""
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
```

Key settings:
- `-n4` — runs tests in parallel across 4 workers (pytest-xdist)
- `--strict-markers` — unregistered pytest marks cause an error
- `--cov=custom_components` — coverage measured over `custom_components/`
- `asyncio_mode = "auto"` — all `async def` test functions are treated as asyncio tests automatically

## Coverage

Coverage is enabled by default via `--cov=custom_components` in `addopts`. After a test run, a summary is printed to the terminal. To generate an HTML report:

```powershell
wsl -e env -u HOME -u WSLENV bash /mnt/c/Users/holger.rabbach/Coding/schellenberg_usb/.wsl_exec.sh "uv run pytest -p no:cacheprovider --cov=custom_components --cov-report=html"
```

The HTML report is written to `htmlcov/index.html`. Coverage configuration:

```toml
[tool.coverage.run]
source = ["custom_components"]

[tool.coverage.report]
show_missing = true
```

No minimum coverage threshold is configured — coverage is informational only.

## Lint and Type Checking (Native Windows)

Ruff and mypy run natively on Windows in a separate `.venv-win` virtual environment. There is no pre-commit hook installed, so these must be run manually as part of the quality gate before committing.

### Ruff (lint + format)

```powershell
$env:UV_PROJECT_ENVIRONMENT = ".venv-win"
uv run ruff check custom_components/ tests/
uv run ruff format --check custom_components/ tests/
```

To auto-fix lint issues:

```powershell
$env:UV_PROJECT_ENVIRONMENT = ".venv-win"
uv run ruff check --fix custom_components/ tests/
```

Ruff is configured with a max line length of 80 characters (`[tool.ruff.lint.pycodestyle]` in `pyproject.toml`).

### mypy (static types)

```powershell
$env:UV_PROJECT_ENVIRONMENT = ".venv-win"
uv run mypy custom_components/
```

mypy targets Python 3.13 and uses `follow_imports = "silent"` and `ignore_missing_imports = true` to avoid noise from untyped third-party dependencies.

## Writing New Tests

- Place new test files in `tests/` following the `test_<module>.py` naming convention.
- All `async def` test functions are picked up automatically (`asyncio_mode = "auto"`).
- Use `hass: HomeAssistant` as a fixture parameter — it is provided by `pytest-homeassistant-custom-component`.
- For subentry flows, enter the flow via `hass.config_entries.subentries.async_init(...)`, not `async_step_<type>` directly (the HA framework routes through `async_step_user` first).
- Place shared fixtures in `tests/conftest.py`.
- Patch serial I/O using the `mock_serial` fixture or `unittest.mock.patch("serial.Serial")`.
