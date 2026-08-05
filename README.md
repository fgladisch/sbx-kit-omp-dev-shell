# OMP development shell kit

A standalone Docker Sandboxes kit for [Oh My Pi (`omp`)](https://github.com/can1357/oh-my-pi) with Zsh, [fnm](https://github.com/Schniz/fnm), [pnpm](https://pnpm.io/), architecture-native Chromium, and repository-aware Node.js setup.

## What it does

During sandbox creation, the kit:

- installs the latest OMP prebuilt release for the sandbox architecture;
- configures OMP as the sandbox entrypoint with tool approvals delegated to the sandbox boundary;
- installs Zsh and makes it the `agent` user's login shell;
- installs the native compilation toolchain from Ubuntu's `build-essential` package;
- installs fnm for the `agent` user;
- installs Playwright's Chromium build and browser system libraries for the sandbox architecture;
- exposes OMP and fnm on the sandbox-wide `PATH`;
- maps the `anthropic`, `cortecs`, and `sonarqube` sandbox secrets to proxy-managed `ANTHROPIC_API_KEY`, `CORTECS_API_KEY`, and `SONARQUBE_TOKEN` environment variables;
- initializes fnm from `~/.zshrc`.

At sandbox startup, fnm detects the repository's requested Node.js version and installs it when necessary. Detection supports:

- `.node-version`;
- `.nvmrc`;
- `package.json#engines.node`;
- version files in parent directories.

It installs pnpm plus the TypeScript and ESLint language servers once per selected Node.js version. A pinned Playwright release installs an architecture-native Chromium build once per sandbox, then exposes it as `chrome`, `chromium`, and `google-chrome` in the agent's `PATH`. OMP automatically enables ESLint when the repository root contains an ESLint configuration file.

If fnm cannot select or install the repository's requested version, the setup installs and uses the latest Node.js LTS release.

The selected or fallback fnm runtime becomes the `agent` user's Node.js default, so OMP tool calls and non-interactive commands use it instead of the base image's system Node.js.

## Allow the kit publisher

Docker Sandboxes restricts remote kit publishers through `kit.allowedSources`. Add `github.com/fgladisch/` while preserving any sources you already trust:

```bash
sbx settings get kit.allowedSources
sbx settings set kit.allowedSources \
  '["docker.io/","github.com/docker/","github.com/fgladisch/"]'
```

The exact array is an example based on the default Docker sources. Include any additional entries from your existing setting.

## Usage

```bash
sbx run \
  --kit "git+https://github.com/fgladisch/sbx-kit-omp-dev-shell.git" \
  omp "$PWD"
```

### Launcher

[`bin/omp-sbx`](bin/omp-sbx) wraps the full per-project lifecycle. Run it from a repository to:

1. create or reuse a sandbox named `omp-<repository>`;
2. seed a newly created sandbox with the host's portable `~/.omp/agent` state;
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

The launcher copies the host's `~/.omp/agent` directory into
`/home/agent/.omp/agent` only when it creates a sandbox. Host-specific caches,
native binaries, browser downloads, and live daemon sockets stay on the host.
Later runs preserve the sandbox's own sessions. Removing the named sandbox with
`sbx rm` also removes those sessions.

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

Store all three credentials globally before creating a sandbox:

```bash
sbx secret set -g anthropic
sbx secret set -g cortecs
sbx secret set -g sonarqube
```

The sandbox receives proxy-managed sentinel values rather than the real credentials. The host-side proxy replaces them in requests to the configured Anthropic, Cortecs, and SonarCloud API endpoints.

## Network access

The kit allows the domains needed for:

- Ubuntu and Docker APT metadata;
- the OMP, fnm, pnpm, and Playwright installers plus their release assets;
- Node.js distributions from `nodejs.org`;
- OpenAI/Codex, Anthropic, and Cortecs model endpoints.

Review `caps.network.allow` in [`spec.yaml`](spec.yaml) before use if your environment requires a narrower egress policy.

## Compatibility

The kit uses `schemaVersion: "1"` with the current `kind: sandbox`, `sandbox`, and `caps.network.allow` fields. It validates with Docker Sandboxes `v0.37.1`.

Validate it locally with:

```bash
sbx kit validate .
```

## Lifecycle

- `commands.install` installs OMP, Zsh, fnm, and a pinned architecture-native Chromium build with Playwright's browser dependencies during sandbox creation.
- `commands.initFiles` writes the repository-aware Node.js setup command and executable LSP bootstraps, making both server commands discoverable before OMP starts.
- `commands.startup` selects the repository's Node.js version and provisions its pnpm and language-server tools. A bootstrap invocation shares the locked setup path if OMP launches a server before provisioning finishes.

All installation steps are idempotent so sandbox startup and recreation can safely repeat them.
