# OMP development shell kit

A standalone Docker Sandboxes kit for [Oh My Pi (`omp`)](https://github.com/can1357/oh-my-pi) with Zsh, [fnm](https://github.com/Schniz/fnm), and repository-aware Node.js setup.

## What it does

During sandbox creation, the kit:

- installs the latest OMP prebuilt release for the sandbox architecture;
- configures OMP as the sandbox entrypoint with tool approvals delegated to the sandbox boundary;
- installs Zsh and makes it the `agent` user's login shell;
- installs fnm for the `agent` user;
- exposes OMP and fnm on the sandbox `PATH`;
- initializes fnm from `~/.zshrc`.

At sandbox startup, fnm detects the repository's requested Node.js version and installs it when necessary. Detection supports:

- `.node-version`;
- `.nvmrc`;
- `package.json#engines.node`;
- version files in parent directories.

The repository must declare a Node.js version through one of these mechanisms. Sandbox startup fails when fnm cannot resolve a version.

## Allow the kit publisher

Docker Sandboxes restricts remote kit publishers through `kit.allowedSources`. Add `github.com/fgladisch/` while preserving any sources you already trust:

```bash
sbx settings get kit.allowedSources
sbx settings set kit.allowedSources \
  '["docker.io/","github.com/docker/","github.com/fgladisch/"]'
```

The exact array is an example based on the default Docker sources. Include any additional entries from your existing setting.

## Usage

The kit is standalone; do not compose it with the upstream Pi kit:

```bash
sbx run \
  --kit "git+https://github.com/fgladisch/sbx-kit-omp-dev-shell.git" \
  omp "$PWD"
```

For reproducible sandbox configuration, pin the kit to a full commit SHA:

```bash
sbx run \
  --kit "git+https://github.com/fgladisch/sbx-kit-omp-dev-shell.git#ref=<kit-commit>" \
  omp "$PWD"
```

The mixin-style `requires.agent: pi` dependency is no longer needed: `spec.yaml` now supplies the `shell-docker` sandbox image, installs OMP, and launches `omp` directly.

## Network access

The kit allows the domains needed for:

- Ubuntu and Docker APT metadata;
- the OMP and fnm installers plus GitHub release assets;
- Node.js distributions from `nodejs.org`;
- OpenAI/Codex and Anthropic model endpoints.

Review `caps.network.allow` in [`spec.yaml`](spec.yaml) before use if your environment requires a narrower egress policy.

## Compatibility

The kit uses `schemaVersion: "1"` with the current `kind: sandbox`, `sandbox`, and `caps.network.allow` fields. It validates with Docker Sandboxes `v0.37.1`.

Validate it locally with:

```bash
sbx kit validate .
```

## Lifecycle

- `commands.install` installs OMP, Zsh, and fnm during sandbox creation.
- `commands.initFiles` creates the repository Node.js setup command with `${WORKDIR}` resolved by Docker Sandboxes.
- `commands.startup` runs the setup command whenever the sandbox starts.

All installation steps are idempotent so sandbox startup and recreation can safely repeat them.
