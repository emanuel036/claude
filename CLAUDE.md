# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## System Overview

This is a **multi-submodule workspace** for an end-to-end fleet monitoring pipeline. An embedded device in each truck collects CAN bus telemetry and video, processes it through business logic, and publishes events via MQTT to Azure IoT Hub. Cloud services parse and route those messages to a frontend portal.

**Data flow:** CAN Bus / Video → vtp (hardware APIs) → lta (business logic) → MQTT/JSON → Azure IoT Hub → core_vtp (parser/router) → Frontend Portal

## Submodules

| Directory | Repo | Stack | Branch | Role |
|-----------|------|-------|--------|------|
| `lta/` | ttm-telematics-linux-telematics-apps-na | C++14, CMake, Docker | develop | Business logic apps — **most changes happen here** |
| `vtp/` | vehicle-telematics-platform | C/C++, Make | (tag-based) | Hardware platform SDK; docs in Confluence |
| `test_framework/` | embedded_test_automation | Python 3.10+, pytest-bdd, uv | main | BDD tests on real hardware |
| `core_vtp/` | ttm-telematics-core-vtp | Java, Maven | develop | Cloud message parser & router |
| `psvi-analytics/` | psvi-analytics | Python, matplotlib, folium | main | Debug analytics for cloud messages |

## Change Impact Matrix

| Changed | Must also review |
|---------|-----------------|
| vtp (new API/topic) | lta, test_framework |
| lta (new message/field/app) | test_framework, core_vtp, psvi-analytics |
| test_framework, core_vtp, psvi-analytics | No downstream dependents |

## LTA Development (Primary Submodule)

### Build Commands (inside Docker container)

```bash
# Build Docker image and start container
docker build -t lta-dev:latest .
docker compose up -d
docker compose exec lta-dev bash

# Inside container — `build` is aliased to /root/workspace/scripts/build
build -m dev x1n image       # Full firmware image for x1n (ARM)
build -m dev adplus2 image   # Full firmware image for adplus2 (ARM)
build tests                  # Build + run all unit tests (x86_64)
build tests coverage         # Tests + lcov coverage report
build void                   # Build without hardware deps

# Flags: -f force LTA rebuild, -F force complete rebuild, -m MODE (dev/prod/bdd)
```

### Running a Single Test

Tests run via CppUTest on x86_64 only. After `build tests`, use ctest:
```bash
cd /root/workspace/build-x86_64
ctest --output-on-failure -R <test_name>   # e.g., -R alarms_test
```

### Architecture

- **Apps** (`lta/apps/<name>/`): Independent processes. Each subscribes to MQTT topics, applies rules, publishes results. Entry point pattern: `SomeDataSource::Init(); appcore::RunVtp();`
- **Appcore** (`lta/libs/appcore/`): Shared framework — MQTT topic classes (~80+), config binding, SQLite, timer management, VTP function abstractions
- **HAL libs** (`lta/libs/<target>_hal/`): Hardware abstraction per target (adplus2, x1n, streamax)
- **vtp_references**: Function pointers wrapping VTP SDK calls — tests replace these with mocks for dependency injection
- **Topic extension**: `appcore::AddExtend<TopicClass>(callback)` to hook into topic message handling

### Code Style (enforced by clang-format + clang-tidy, warnings are errors)

- Google C++ Style base, **4-space indent**, 80-char column limit
- Pointer alignment: left (`int* ptr`), K&R braces (never on own line)
- **Naming:** `lower_case` namespaces/vars, `CamelCase` classes/structs, `kCamelCase` constants/enums, `lower_case_` class members (trailing underscore)
- No exceptions — error handling via return codes and `ERROR()`/`WARN()`/`INFO()` logging macros
- Header guards: `PATH_TO_FILE_H_` convention (e.g., `APPCORE_SQLITE_H_`)

### Testing Patterns

- Framework: CppUTest. Tests in `apps/<app>/tests/` and `libs/appcore/tests/`
- `__UNIT_TEST__` preprocessor guard: code calling real VTP/Streamax SDK must be guarded with `#ifndef __UNIT_TEST__` or use `vtp_references` function pointers
- Mocks: function pointers in `vtp_references` namespace + `libs/lta_mocks/` for Streamax SDK fakes
- Production DB: `/opt/data/dsn_prod.db` (`BUILD_PRODUCTION=ON`); Dev DB: `/opt/data/halpersist.db`

### Common Pitfalls

- Tests only compile for x86_64 — device-specific HAL/CAN code is excluded via `LTA_TARGET`
- Many apps/libs only build when `LTA_TARGET != void` — check CMakeLists.txt conditions before adding dependencies
- New MQTT topics must be added to both the topic header and `appcore.h`
- Image size regression: build script warns if new image is smaller than base (may indicate missing components)

## Test Framework (BDD)

```bash
cd test_framework
uv sync
uv run pytest --gherkin-terminal-reporter tests/step_defs/events/ -vv -m "not manual and not skip" --collect_logs
uv run pytest --gherkin-terminal-reporter tests/ -m "TC_001_001"   # Single test by ID
```

Requires `config.yml` with device credentials (see `src/config/config_template.yml`).

## PSVI Analytics

```bash
cd psvi-analytics
pip install -r requirements.txt
python main.py --device "SERIAL:ENV" --general   # Message type counts
python main.py --device "SERIAL:ENV" --video_events --days 7
```

Credentials from 1Password TouCAN vault → `config/credentials.json`.

## CI/CD

- **lta**: GitHub Actions (`.github/workflows/build-lta.yml`) — builds x1n, adplus2, void, and tests on every PR and push to develop/main
- **Workspace**: Daily submodule sync workflow (`.github/workflows/update-submodules.yml`) — updates all submodules to configured branches, syncs vtp to matching VTP SDK tag
