# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

`arcam-fmj` is a Python asyncio library for controlling Arcam FMJ AV receivers (and compatible JBL/AudioControl processors) over their TCP-based IP control protocol. It's built primarily for use by Home Assistant's Arcam integration, but works standalone. The package is published to PyPI as `arcam-fmj` and installs an `arcam-fmj` console script.

See `../AVR10_NOTES.md` for hands-on protocol/device notes from real hardware testing against an Arcam AVR10.

See `../.claude/skills/README.md` for Claude Code skills that control the AVR10 over IP (volume, power, mute, source).

## Commands

Install for development (editable, with test extras):
```bash
pip install -e .[tests]
```

Run the full test suite (matches CI in `.github/workflows/python-package.yml`):
```bash
pytest --cov
```

Run a single test file / test:
```bash
pytest tests/test_state.py
pytest tests/test_state.py::test_power_state -k <name>
```

Lint (config in `pylintrc`):
```bash
pylint src/arcam
```

The console entry point (installed as `arcam-fmj`, backed by `src/arcam/fmj/console.py`) is also useful for manual/integration testing against a real or fake device:
```bash
arcam-fmj state --host 192.168.0.2 --port 50000 --source PVR --volume 50
arcam-fmj server --host localhost --port 50000 --model AVR450   # spin up a fake device
arcam-fmj client --host localhost --port 50000 --command 13     # raw command/response
```

pytest is configured with `asyncio_mode = auto` (see `setup.cfg`), so async test functions don't need `@pytest.mark.asyncio`.

## Architecture

The package lives under `src/arcam/fmj/` (note the `build/lib/...` directory is a stale build artifact, not source of truth — always edit under `src/`).

The protocol/model layer is deliberately separated from the networking layer:

- **`__init__.py`** — the protocol foundation. Defines the wire format (`CommandPacket`/`ResponsePacket`/`AmxDuetRequest`/`AmxDuetResponse` with `to_bytes`/`from_bytes`), the low-level framed reader/writer (`read_command`, `read_response`, `write_packet`, `_read_delimited`), and all protocol enums/tables: `CommandCodes`, `AnswerCodes`, `SourceCodes`, `DecodeMode2CH`/`DecodeModeMCH`, etc. It also holds the per-model capability data used everywhere else:
  - `APIVERSION_*_SERIES` sets group receiver model names (e.g. `APIVERSION_HDA_SERIES`) into device families.
  - `ApiModel` enum represents those families (`API450_SERIES`, `API860_SERIES`, `APIHDA_SERIES`, `APISA_SERIES`, `APIPA_SERIES`, `APIST_SERIES`).
  - `SOURCE_CODES`, `RC5CODE_SOURCE`, `RC5CODE_POWER`, `RC5CODE_MUTE`, `RC5CODE_VOLUME`, `RC5CODE_DECODE_MODE_2CH`/`_MCH` are all keyed by `(ApiModel, zone)` and map logical values to either direct protocol bytes or legacy RC5 IR simulation codes — this is the biggest source of per-model variance in the codebase.
  - `*_WRITE_SUPPORTED` sets (`POWER_WRITE_SUPPORTED`, `MUTE_WRITE_SUPPORTED`, `SOURCE_WRITE_SUPPORTED`, `VOLUME_STEP_SUPPORTED`) determine whether a model supports direct command writes or must fall back to simulated RC5 IR commands — `State` branches on membership in these sets throughout.
  - `IntOrTypeEnum` is a custom `IntEnum` base that tolerates unknown wire values (via `_missing_`/`_create_member`) instead of raising, since not every device firmware version matches the documented command set.

- **`client.py`** — `Client` owns a single TCP connection (`asyncio.open_connection`) and implements the request/response correlation: it writes a request, registers a listener via `listen()`, and awaits a `Future` that resolves when a matching response arrives on the shared read loop (`process()`/`_process_data()`). It also sends periodic heartbeats (`_process_heartbeat`) and enforces a request throttle (`utils.Throttle`) plus timeout/retry (`utils.async_retry`) since devices reject requests sent too quickly. `ClientContext` is the async context manager that starts the connection and the background `process()` task together and tears both down on exit.

- **`state.py`** — `State` is the per-zone view built on top of `Client`. It listens to all incoming packets, caches the latest value per `CommandCode` in `self._state`, and exposes typed `get_*`/`set_*` accessors (e.g. `get_volume`/`set_volume`, `get_source`/`set_source`). Setters decide, based on `_api_model` and the `*_WRITE_SUPPORTED` sets, whether to send a direct command or look up an RC5 code via `get_rc5code()` and simulate an IR button press instead. `update()` polls a fixed set of command codes plus tuner presets and auto-detects `_api_model` from the AMX Duet discovery response the first time it connects.

- **`server.py`** — `Server`/`ServerContext` is a minimal fake-device TCP server used for tests and manual testing (`arcam-fmj server`). Handlers are registered per `(zone, command_code[, data])` tuple via `register_handler`; unmatched requests get a `CommandNotRecognised` response. `console.py`'s `run_server` subclasses `Server` (`DummyServer`) to implement a stateful fake receiver, including RC5 IR command dispatch — useful as a reference for how the real protocol semantics map to state changes.

- **`console.py`** — argparse-based CLI (`arcam-fmj`) with three subcommands: `client` (raw command/response), `state` (typed `State` operations, with `--monitor` for a live-updating view), `server` (runs the fake `DummyServer`).

- **`utils.py`** — cross-cutting helpers unrelated to the wire protocol: `async_retry` decorator, `Throttle` rate limiter, and UPnP/SSDP device-description helpers (`get_uniqueid_from_host` etc.) used by callers (e.g. Home Assistant) to resolve a stable unique ID for a discovered device via its `dd.xml` description.

### Adding support for a new receiver model/family

Changes typically ripple through several of the per-model tables in `__init__.py` together: add the model name to the right `APIVERSION_*_SERIES` set (and any capability sets it should join, e.g. `APIVERSION_ZONE2_SERIES`), extend `SOURCE_CODES`/`RC5CODE_*` dicts with a `(ApiModel, zone)` entry, and update `*_WRITE_SUPPORTED` sets if the new model supports direct writes instead of RC5 simulation. `tests/test_source.py` and `tests/test_state.py` iterate over all `(zone, ApiModel)` combinations, so new mappings are exercised automatically once added.
