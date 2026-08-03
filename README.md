# OMP development shell kit

A standalone Docker Sandboxes kit for [Oh My Pi (`omp`)](https://github.com/can1357/oh-my-pi) with Zsh, [fnm](https://github.com/Schniz/fnm), [pnpm](https://pnpm.io/), and repository-aware Node.js setup.

## What it does

During sandbox creation, the kit:

- installs the latest OMP prebuilt release for the sandbox architecture;
- configures OMP as the sandbox entrypoint with tool approvals delegated to the sandbox boundary;
- installs Zsh and makes it the `agent` user's login shell;
- installs the native compilation toolchain from Ubuntu's `build-essential` package;
- installs fnm for the `agent` user;
- exposes OMP and fnm on the sandbox-wide `PATH`;
- maps the `anthropic` and `sonarqube` sandbox secrets to proxy-managed `ANTHROPIC_API_KEY` and `SONARQUBE_TOKEN` environment variables;
- initializes fnm from `~/.zshrc`.

At sandbox startup, fnm detects the repository's requested Node.js version and installs it when necessary. Detection supports:

- `.node-version`;
- `.nvmrc`;
- `package.json#engines.node`;
- version files in parent directories.

It installs pnpm and the TypeScript language server once per selected Node.js version, then links the selected runtime and tools into the `agent` user's `~/.local/bin`.

The repository must declare a Node.js version through one of these mechanisms. Sandbox startup fails when fnm cannot resolve a version.

The selected fnm runtime becomes the `agent` user's Node.js default, so OMP tool calls and non-interactive commands use the repository version instead of the base image's system Node.js.

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
2. seed a newly created sandbox with the host's `~/.omp` configuration;
3. copy the version-controlled OMP skills into a newly created sandbox;
4. update OMP and its installed plugins;
5. attach to OMP and forward any supplied OMP arguments.

Link it into `~/bin`:

```bash
ln -s "$(pwd)/bin/omp-sbx" "$HOME/bin/omp-sbx"
```

Then launch OMP from any repository:

```bash
omp-sbx
```

Resume the latest project session with `--continue`, or open OMP's session
picker with `--resume`:

```bash
omp-sbx --continue
omp-sbx --resume
```

The launcher copies the host's `~/.omp` directory into `/home/agent/.omp` only
when it creates a sandbox. Later runs preserve the sandbox's own sessions.
Removing the named sandbox with `sbx rm` also removes those sessions.

### Skills

Keep shared skills in a dedicated Git repository. The launcher copies
`~/code/omp-skills/skills` by default; override the source directory with
`OMP_SKILLS_DIR`.

Expose the same repository to host OMP through its canonical user skills path:

```bash
skills_dir="${OMP_SKILLS_DIR:-$HOME/code/omp-skills/skills}"
mkdir -p "$HOME/.agents"
ln -sfn "$skills_dir" "$HOME/.agents/skills"
rm -f "$HOME/.omp/agent/skills"
```

When creating a sandbox, the launcher copies the source directory to
`/home/agent/.agents/skills`. Existing sandboxes and their skills are left
unchanged on later launches.

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

- `commands.install` installs OMP, Zsh, and fnm during sandbox creation.
- `commands.initFiles` writes one compact setup command with the workspace path.
- `commands.startup` selects the repository's Node.js version and provisions pnpm and the TypeScript language server.

All installation steps are idempotent so sandbox startup and recreation can safely repeat them.
