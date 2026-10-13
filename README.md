# OMP development shell kit

A standalone Docker Sandboxes kit for [Oh My Pi (`omp`)](https://github.com/can1357/oh-my-pi) with Zsh, [fnm](https://github.com/Schniz/fnm), [pnpm](https://pnpm.io/), architecture-native Chromium, and repository-aware Node.js setup.

## What it does

During sandbox creation, the kit:

- installs the latest OMP prebuilt release available for the sandbox architecture;
- configures OMP as the sandbox entrypoint with `--approval-mode=yolo`;
- loads the repository's `AGENTS.md` through the kit's `agentInstructions`;
- installs Zsh, Ubuntu's `build-essential` toolchain, and fnm;
- installs Playwright's pinned, architecture-native Chromium build and its system libraries;
- exposes OMP, fnm, Chromium, pnpm, and the installed language servers on the sandbox `PATH`;
- binds the `sipgate_os`, `sonarqube`, and `langsmith` sandbox secrets to proxy-managed `SIPGATE_OS_API_KEY`, `SONARQUBE_TOKEN`, and `LANGSMITH_API_KEY` environment variables;
- initializes fnm from `~/.zshrc`.

At sandbox startup, fnm detects the repository's requested Node.js version and installs it when necessary. Detection supports:

- `.node-version`;
- `.nvmrc`;
- `package.json#engines.node`;
- version files in parent directories.

It installs pnpm plus the TypeScript and ESLint language servers once per selected Node.js version. A pinned Playwright release installs an architecture-native Chromium build once per sandbox, then exposes it as `chromium` in the agent's `PATH`. OMP automatically enables ESLint when the repository root contains an ESLint configuration file.

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
  "git+https://github.com/fgladisch/sbx-kit-omp-dev-shell.git" \
  "$PWD"
```

### Launcher

[`bin/omp-sbx`](bin/omp-sbx) wraps the full per-project lifecycle. Run it from a repository to:

1. derive the sandbox name `omp-<repository>` from the current directory name;
2. create the sandbox when necessary, with the current directory as its workspace;
3. copy the host's complete `~/.omp/agent` directory into a newly created sandbox;
4. replace the copied commands and add the shared skills configured by `OMP_COMMANDS_DIR` and `OMP_SKILLS_DIR`;
5. update OMP and its installed plugins on every launch;
6. attach to OMP and forward every argument to OMP unchanged.

Link it into `~/bin`:

```bash
ln -s "$(pwd)/bin/omp-sbx" "$HOME/bin/omp-sbx"
```

Then launch OMP from any repository:

```bash
omp-sbx
```

Every argument is passed to OMP, including subcommands such as `acp`. In a
terminal the launcher attaches with `sbx run`. When stdin or stdout is not a
terminal, as when an editor spawns it, it starts OMP with `sbx exec -i` instead,
because `sbx run` relays no output without a TTY. Setup commands read no stdin
and write to stderr, so OMP's stdin and stdout carry only the session. That
keeps stdio-based protocols like ACP intact.

Resume the latest project session with `--continue`, or open OMP's session
picker with `--resume`:

```bash
omp-sbx --continue
omp-sbx --resume
```

To run OMP as an ACP agent in Zed, register the launcher as a custom agent
server in Zed's `settings.json`:

```json
{
  "agent_servers": {
    "omp": {
      "type": "custom",
      "command": "omp-sbx",
      "args": ["acp"]
    }
  }
}
```

The launcher uses the published Git kit by default. Set `OMP_KIT` to a local
directory or another supported kit reference when testing a different build:

```bash
OMP_KIT=/path/to/sbx-kit-omp-dev-shell omp-sbx
```

The initial copy places the host's full `~/.omp/agent` directory at
`/home/agent/.omp/agent`, then replaces its `commands` entry with the configured
commands directory. Existing sandboxes keep independent configuration,
databases, sessions, commands, and skills. Changes made to the corresponding
host files are not synchronized on later launches. Removing the named sandbox
with `sbx rm` also removes that sandbox-local state.

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

### Commands

Keep shared commands in a dedicated Git repository. The launcher copies
`~/code/omp-skills/commands` by default; override the source directory with
`OMP_COMMANDS_DIR`.

When creating a sandbox, the launcher replaces the copied host
`~/.omp/agent/commands` entry, including a symlink, with the source directory at
`/home/agent/.omp/agent/commands`. Existing sandboxes and their commands are
left unchanged on later launches.

## Credentials

Store the three credentials used by the kit globally before creating a sandbox:

```bash
sbx secret set -g sipgate_os
sbx secret set -g sonarqube
sbx secret set -g langsmith
```

Schema v2 third-party kits also require credential bindings. Before using the
non-interactive launcher, merge these approvals into
`~/.config/sbx/credentials.yaml` without removing existing bindings:

```yaml
bindings:
  sipgate_os:
    apiKey:
      domains:
        - coding.sipgate.ai
  sonarqube:
    apiKey:
      domains:
        - api.sonarcloud.io
  langsmith:
    apiKey:
      domains:
        - eu.api.smith.langchain.com
```

Alternatively, run the kit interactively once and approve each requested
binding at the prompt. Non-interactive `sbx create` runs with unbound
credentials withheld.

Existing sandboxes retain the network policy and credential injection domains
from creation time. After changing a binding or kit domain, remove and recreate
the sandbox before retrying the affected service.

The sandbox receives proxy-managed sentinel values rather than the real
credentials. The host-side proxy injects bearer tokens for Sipgate and
SonarCloud requests, and the `X-Api-Key` header for LangSmith requests.

## Network access

The kit allows the domains needed for:

- Ubuntu and Docker APT metadata;
- OMP, fnm, Node.js, npm, pnpm, and Playwright installation;
- GitHub API access and release downloads;
- OpenAI and Anthropic authentication and model endpoints;
- Sipgate model endpoint;
- SonarCloud and LangSmith API access.

Review `permissions.network.allow` in [`spec.yaml`](spec.yaml) before use if your environment requires a narrower egress policy.

## Compatibility

The kit uses `schemaVersion: "2"` with the current `kind: sandbox`, `sandbox`,
`agentInstructions`, `permissions`, `credentials`, and `setup` fields. It
validates with Docker Sandboxes `v0.39.0`.

Validate it locally with:

```bash
sbx kit validate .
```

## Lifecycle

- `setup.install` installs OMP, Zsh, fnm, and a pinned architecture-native Chromium build with Playwright's browser dependencies during sandbox creation.
- `setup.files` writes the repository-aware Node.js setup command plus executable LSP bootstraps.
- `setup.startup` selects the repository's Node.js version and provisions its pnpm and language-server tools. A bootstrap invocation shares the locked setup path if OMP launches a server before provisioning finishes.

All installation steps are idempotent so sandbox startup and recreation can safely repeat them.
