# Repository guidance

## Setup and commands

Use Python 3.10+ and run commands from the repository root.

```bash
python3 -m pip install -r requirements.txt
python3 -m pip install -e ".[dev]"
python3 -m pip install pre-commit
pre-commit run --all-files
```

`requirements.txt` includes `requests` for iOS support, which `setup.py` does
not include. The authoritative style checks are in `.pre-commit-config.yaml`:
Ruff import sorting (`--select I --fix`), Ruff formatting, typos, and
PyMarkdown (`fix`). These hooks can modify files. To check only changed files:

```bash
pre-commit run --files path/to/changed_file.py
```

The READMEs document `pytest tests/`, but this checkout has no test suite or
pytest configuration. When adding tests, use these full-suite and single-test
commands, replacing the example path and function with actual test names:

```bash
python3 -m pytest tests/
python3 -m pytest tests/test_example.py::test_example
```

List supported apps without connecting to a device or model:

```bash
python3 main.py --list-apps
python3 main.py --device-type hdc --list-apps
python3 main.py --device-type ios --list-apps
```

Check a running OpenAI-compatible model endpoint independently of a phone:

```bash
python3 scripts/check_deployment_en.py \
  --base-url http://localhost:8000/v1 --model MODEL_NAME
```

Real automation requires an authorized device and a model endpoint. Android
uses ADB and ADB Keyboard for text input; HarmonyOS uses HDC. For iOS, follow
`docs/ios_setup/ios_setup.md` to deploy WebDriverAgent using macOS/Xcode and
expose its HTTP endpoint, normally port 8100. Supply an explicit task argument
when running the CLI without an interactive terminal.

## Architecture

- `main.py` is the unified CLI: it selects `adb`, `hdc`, or `ios`, handles
  device-management commands, checks prerequisites and model connectivity,
  and constructs the appropriate agent. `ios.py` is a separate iOS CLI.
- `PhoneAgent` in `phone_agent/agent.py` handles Android and HarmonyOS.
  Each step captures a screenshot and current app, builds multimodal messages,
  requests one model action, executes it, and appends the assistant response.
  `run()` starts a fresh task; `step()` and `reset()` support manual stepping.
  Historical screenshots are removed after inference to limit context size;
  text and assistant responses remain.
- `phone_agent/model/client.py` owns the OpenAI-compatible streaming client,
  response splitting, inference timings, and `MessageBuilder`. Both agent
  implementations share this layer and `ModelConfig`.
- Android/HarmonyOS actions pass through `actions/handler.py` and the
  process-global `device_factory.py`, whose lazy backend modules expose
  matching device, input, screenshot, and connection operations. Select HDC
  with `set_device_type(DeviceType.HDC)` before using the Python API; the
  default backend is ADB.
- iOS uses `IOSPhoneAgent`, `IOSAgentConfig`, `IOSActionHandler`, and
  `phone_agent/xctest/` directly, not the device factory. The iOS agent
  initializes a WebDriverAgent session and passes its URL/session ID through
  operations. Changes to shared agent behavior or action semantics need
  consideration in both agent and handler implementations.

## Codebase conventions

- The model action protocol is Python-like `do(action="Tap", ...)` or
  `finish(message="...")`, not JSON. `parse_action()` in `actions/handler.py`
  converts it into dictionaries with `_metadata` set to `do` or `finish`.
  Preserve AST/literal-based parsing rather than evaluating model output.
  The model client also accepts legacy `<think>`/`<answer>` responses.
  Action changes must stay aligned with the prompt definitions, parser,
  handler dispatch tables, and applicable backends.
- Model coordinates use a 0-1000 scale. Handlers convert them using screenshot
  dimensions before calling device utilities. iOS utilities additionally
  account for screenshot pixels versus WDA points using `SCALE_FACTOR` in
  `xctest/device.py`; avoid applying that conversion twice.
- App names are lookup keys shared by launching and current-app detection.
  Maintain the platform-specific mappings in `config/apps.py`,
  `config/apps_harmonyos.py` (including `APP_ABILITIES`), or
  `config/apps_ios.py`, rather than embedding package/bundle IDs in handlers.
- Active system prompts are `config/prompts_zh.py` and `config/prompts_en.py`,
  selected by `config.get_system_prompt()` with language codes `cn` and `en`.
  `config/prompts.py` is not the active selector. User-facing translations
  belong in `config/i18n.py`; preserve Chinese support and English equivalents.
- CLI environment defaults are read while parsing arguments in the entry
  points. Setting `PHONE_AGENT_*` variables does not automatically configure
  library-created `ModelConfig` or `AgentConfig` instances.
- Shared Android/HarmonyOS delays live in `config/timing.py` as
  `TIMING_CONFIG`, with `PHONE_AGENT_*_DELAY` overrides loaded on initialization
  and `update_timing_config()` for runtime changes. Reuse these settings for
  related delays. HDC subprocess calls use `hdc.connection._run_hdc_command()`
  to preserve the existing verbosity control.
- Preserve the confirmation callback for sensitive taps carrying `message`
  and the takeover callback for login/CAPTCHA/manual intervention. Their
  defaults prompt on stdin, so unattended callers need explicit callbacks.
- The contribution guide in `.github/PULL_REQUEST_TEMPLATE.md` requires
  English identifiers and comments. Consult `README.md`, `README_en.md`,
  and `README_coding_agent.md` for deployment and usage; confirm CLI details
  against the entry points when examples differ.
