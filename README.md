# snapper-delegate

A dependency-light consult delegate for [Snapper](https://github.com/mateusz-klatt/snapper).
It connects to the authenticated Snapper control and wake interfaces, discovers
MCP tools, and runs bounded chat-completions consults under server-authoritative
hold and resume control.

## Development

Use Python 3.14 or newer and uv 0.11.2. From this repository's root:

```bash
uv sync --frozen --extra test --no-install-project --no-build
uv run --no-sync --no-build python -m pytest
```

`uv.lock` pins and hashes the runtime and test dependencies from public PyPI.
The frozen setup installs only binary distributions and leaves the local project
uninstalled; pytest loads its source directly through the configured Python path.
Its runtime package versions match the Docker runtime lock. Refresh both locks
with the commands below after changing runtime dependencies.

The test suite uses recorded fixtures and mocked transports. It runs independently
of a Snapper checkout. Snapper's own test suite covers its managed-process
registration adapter and backend integration.

Tests measure line and branch coverage across the complete package and write
`coverage.xml`. This report includes the optional registration adapter, whose
backend integration tests run in Snapper rather than the standalone suite.
The GitHub Actions CI runs the standalone tests on pushes to `master` or `main`
and on pull requests, then submits the coverage and source analysis to the independent
[SonarCloud project](https://sonarcloud.io/dashboard?id=mateusz-klatt_snapper-delegate).
The workflow waits for its quality gate and fails when the gate fails. Repository
administrators must configure the `SONAR_TOKEN` Actions secret for analysis.

`requirements.txt` contains exact direct runtime pins. `requirements.lock`
contains the universal, hashed binary resolution consumed by the Docker build.
Its generated header records the original command from the Snapper checkout;
from this repository, the equivalent command is:

```bash
uv pip compile requirements.txt --python-version 3.14 --universal \
    --generate-hashes --only-binary=:all: --output-file requirements.lock
```

Then align the development lock with every runtime pin without adding duplicate
constraints to package metadata:

```bash
python - <<'PY'
import re
import subprocess
from pathlib import Path

pins = re.findall(
    r"^([A-Za-z0-9_.-]+==[^\s;\\]+)",
    Path("requirements.lock").read_text(),
    re.MULTILINE,
)
command = ["uv", "lock", "--default-index", "https://pypi.org/simple", "--python", "3.14"]
for pin in pins:
    command.extend(["--upgrade-package", pin])
subprocess.run(command, check=True)
PY
```

## Container

Build from this repository's root:

```bash
docker build -t snapper-delegate:blackbox .
```

When checked out as Snapper's submodule, build from the Snapper root with:

```bash
docker build -t snapper-delegate:blackbox integrations/snapper-delegate
```

The image runs `python -m snapper_delegate.pid1` as an unprivileged user. Supply
configuration through environment variables and mount credentials as readable
files:

| Variable | Purpose |
| --- | --- |
| `SNAPPER_DELEGATE_MODEL_ALIAS` | One model route token |
| `SNAPPER_DELEGATE_BASE_URL` | Required HTTPS provider origin on port 443 |
| `SNAPPER_DELEGATE_ENDPOINT_PATH` | Chat-completions path, default `/v1/chat/completions` |
| `SNAPPER_DELEGATE_SNAPPER_URL` | HTTP or HTTPS Snapper control origin |
| `SNAPPER_DELEGATE_API_KEY_FILE` | Required exact path `/run/secrets/delegate/api_key` |
| `SNAPPER_DELEGATE_TOKEN_FILE` | Required exact path `/run/secrets/delegate/delegate_token` |
| `SNAPPER_DELEGATE_MAX_TOOL_ROUNDS` | Required tool-round limit, from 1 to 8 |
| `SNAPPER_PID1_STRICT` | Set to `1` to exit with an error on invalid startup configuration |
| `SNAPPER_LOG_FILE` | Set to `/app/data/log/delegate/model/delegate.log` to enable file logging |

Missing or invalid configuration leaves the default PID1 in a signal-aware idle
state. Production deployments can require strict startup. Snapper's Compose
configuration supplies the network, secret mounts, filesystem restrictions, and
service wiring.

## Snapper integration

The standalone runner imports the `snapper_delegate` package and communicates
with Snapper through its authenticated interfaces. The optional
`snapper_delegate.registration.ManagedDelegateRunner` adapter integrates with
Snapper's process registry and requires the Snapper backend on the Python path.
The container omits this backend-specific adapter.

Snapper checks out this repository at `integrations/snapper-delegate` as a Git
submodule. Commit and publish delegate changes here before updating the gitlink
in Snapper.

## History and license

The implementation was extracted from Snapper's
`integrations/snapper-delegate` directory at source commit
[`987624bddb05c06515989705165800f0e9d47106`](https://github.com/mateusz-klatt/snapper/commit/987624bddb05c06515989705165800f0e9d47106).
Its directory history was preserved through `git subtree split` at
`1cbcecb903fb60a59b2af4cbf5ba3e9943b7997e` and merged with this repository's
original placeholder history. Snapper's published history remains intact.

Current original code and documentation are licensed under the Apache License,
Version 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Earlier versions retain
the licenses under which they were released. Third-party dependencies retain
their own licenses.
