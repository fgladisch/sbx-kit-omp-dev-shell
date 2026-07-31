# Pi development shell kit

A Docker Sandboxes mixin for the [Pi coding agent](https://github.com/badlogic/pi-mono) that adds Zsh, [fnm](https://github.com/Schniz/fnm), and repository-aware Node.js setup.

It is designed to compose with Docker's upstream [Pi sandbox kit](https://github.com/docker/sbx-kits-contrib/tree/main/pi).

## What it does

During sandbox creation, the kit:

- installs Zsh and makes it the `agent` user's login shell;
- installs fnm for the `agent` user;
- exposes fnm as `/usr/local/bin/fnm`;
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

Compose this mixin with the upstream Pi kit:

```bash
sbx run \
  --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=pi" \
  --kit "git+https://github.com/fgladisch/sbx-kit-pi-dev-shell.git" \
  pi "$PWD"
```

For reproducible sandbox creation, pin both kits to full commit SHAs:

```bash
sbx run \
  --kit "git+https://github.com/docker/sbx-kits-contrib.git#ref=<pi-kit-commit>&dir=pi" \
  --kit "git+https://github.com/fgladisch/sbx-kit-pi-dev-shell.git#ref=<kit-commit>" \
  pi "$PWD"
```

The mixin can also be added to an existing sandbox, although recreating the sandbox gives the clearest lifecycle behavior:

```bash
sbx kit add <sandbox-name> \
  "git+https://github.com/fgladisch/sbx-kit-pi-dev-shell.git"
```

## Pi commands through Zsh

The kit sets Zsh as the login shell. If Pi is configured with [`@fgladisch/pi-zsh-shell`](https://github.com/fgladisch/pi-extensions/tree/main/packages/pi-zsh-shell), add the fnm initialization to `~/.pi/agent/zsh-functions` so Pi's non-interactive `zsh -f` commands select the repository Node.js version:

```zsh
eval "$(fnm env --use-on-cd --version-file-strategy=recursive --resolve-engines --shell zsh)"
```

The `pi-sbx` wrapper in the author's setup copies that file from the host and appends this line idempotently.

## Network access

The kit allows the domains needed for:

- Ubuntu and Docker APT metadata;
- the fnm installer and GitHub release assets;
- Node.js distributions from `nodejs.org`.

Review `caps.network.allow` in [`spec.yaml`](spec.yaml) before use if your environment requires a narrower egress policy.

## Compatibility

The kit currently uses `schemaVersion: "1"`, matching Docker Sandboxes `v0.37.1` and the upstream Pi kit at the time of publication.

Validate it locally with:

```bash
sbx kit validate .
```

## Lifecycle

- `commands.install` runs during sandbox creation.
- `commands.initFiles` creates the repository Node.js setup command with `${WORKDIR}` resolved by Docker Sandboxes.
- `commands.startup` runs the setup command whenever the sandbox starts.

All installation steps are idempotent so sandbox startup and recreation can safely repeat them.
