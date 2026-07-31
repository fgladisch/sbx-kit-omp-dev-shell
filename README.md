# OMP development shell kit

A standalone Docker Sandboxes kit for [Oh My Pi (`omp`)](https://github.com/can1357/oh-my-pi) with Zsh, [fnm](https://github.com/Schniz/fnm), [pnpm](https://pnpm.io/), and repository-aware Node.js setup.

## What it does

During sandbox creation, the kit:

- installs the latest OMP prebuilt release for the sandbox architecture;
- configures OMP as the sandbox entrypoint with tool approvals delegated to the sandbox boundary;
- installs Zsh and makes it the `agent` user's login shell;
- installs fnm for the `agent` user;
- installs pnpm for the `agent` user;
- exposes OMP, fnm, and pnpm on the sandbox `PATH`;
- maps the `anthropic` and `sonarqube` sandbox secrets to proxy-managed `ANTHROPIC_API_KEY` and `SONARQUBE_TOKEN` environment variables;
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

### Launcher

[`bin/omp-sbx`](bin/omp-sbx) wraps the full per-project lifecycle. Run it from a repository to:

1. create or reuse a sandbox named `omp-<repository>`;
2. copy the host's `~/.omp` configuration into the sandbox;
3. update OMP and its installed plugins;
4. attach to the OMP session.

Link it into `~/bin`:

```bash
ln -s "$(pwd)/bin/omp-sbx" "$HOME/bin/omp-sbx"
```

Then launch OMP from any repository:

```bash
omp-sbx
```

The launcher replaces the sandbox's `/home/agent/.omp` directory with the current host configuration each time it runs.

## Credentials

Store both credentials globally before creating a sandbox:

```bash
sbx secret set -g anthropic
sbx secret set -g sonarqube
```

The sandbox receives proxy-managed sentinel values rather than the real credentials. The host-side proxy replaces them in requests to the configured Anthropic and SonarCloud API endpoints.

## Network access

The kit allows the domains needed for:

- Ubuntu and Docker APT metadata;
- the OMP, fnm, and pnpm installers plus GitHub release assets and the npm registry;
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

- `commands.install` installs OMP, Zsh, pnpm, and fnm during sandbox creation.
- `commands.initFiles` creates the repository Node.js setup command with `${WORKDIR}` resolved by Docker Sandboxes.
- `commands.startup` runs the setup command whenever the sandbox starts.

All installation steps are idempotent so sandbox startup and recreation can safely repeat them.
