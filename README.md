# prime-agent.nix

<p align="center">
  <a href="https://lukasl-dev.github.io/prime-agent.nix/">
    <img src="https://img.shields.io/badge/docs-options-5277C3?style=for-the-badge&logo=nixos&logoColor=white" alt="Options">
  </a>
</p>

A Nix flake for [Prime Agent](https://github.com/PrimeIntellect-ai/prime-agent), a self-improving RLM agent for coding and long-running autonomous work.

It provides:

- packages for `nix run` and `nix build`
- a default npm-built package and an optional Bun-built variant
- NixOS and Home Manager modules
- an overlay exposing `pkgs.prime-agent` and `pkgs.prime-agent-bun`
- `lib.mkAgent` for constructing a configured wrapper
- automatic stable-release and dependency-lock updates

> [!IMPORTANT]
> This is an unofficial Nix flake and is not maintained by Prime Intellect.

## Quick start

```bash
nix run github:lukasl-dev/prime-agent.nix --accept-flake-config
```

Or build either implementation locally:

```bash
nix build .#prime-agent --accept-flake-config
nix build .#prime-agent-bun --accept-flake-config
```

The package includes Node.js, Git, SSH, ripgrep, fd, and uv in its runtime path. Prime Agent uses uv for the one-time provisioning of its persistent IPython kernel under `~/.prime/agent/kernel-venv`.

## Usage

```nix
{
  inputs.prime-agent.url = "github:lukasl-dev/prime-agent.nix";
}
```

### NixOS

```nix
{ inputs, config, ... }:
{
  imports = [ inputs.prime-agent.nixosModules.default ];

  programs.prime-agent = {
    enable = true;
    # rules = ''Be concise.'';
    # skills = [ ./skills ];
    # extensions = [ ./extensions/my-extension.ts ];
    # themes = [ ./themes/custom.json ];
    # promptTemplates = [ ./prompts ];
    # models = inputs.prime-agent + "/models.json";
    # settings.defaultProvider = "openai";
    # settings.defaultModel = "gpt-5";
    # jail.enable = true;
    # extraArgs = [ "--provider" "openai" "--model" "gpt-5" ];
    # environment.OPENAI_API_KEY.file = config.sops.secrets.openai-api-key.path;
  };
}
```

### Home Manager

```nix
{ inputs, config, ... }:
{
  imports = [ inputs.prime-agent.homeModules.default ];

  programs.prime-agent = {
    enable = true;
    settings.defaultProvider = "openai";
    settings.defaultModel = "gpt-5";
    environment.PRIME_AGENT_CODING_AGENT_DIR.value =
      "${config.home.homeDirectory}/.prime/agent";
  };
}
```

### Skills

Pass the common `skills/` directory rather than its individual skill directories:

```nix
programs.prime-agent.skills = [ ./skills ];
```

Each immediate child remains a correctly named skill directory, for example
`skills/github/SKILL.md` with `name: github`. Do not instead use
`[ ./skills/github ./skills/zig ]`: Nix materializes each path separately as a
store path such as `/nix/store/<hash>-github`, and Prime Agent then rejects the
skill because its declared name no longer matches its parent directory name.

### Overlay

```nix
{ inputs, ... }:
{
  nixpkgs.overlays = [ inputs.prime-agent.overlays.default ];
  environment.systemPackages = [ pkgs.prime-agent ];
}
```

### Custom package

```nix
{ inputs, pkgs, ... }:
let
  agent = inputs.prime-agent.lib.mkAgent {
    inherit pkgs;
    modules = [{
      prime-agent = {
        rules = ''Be concise.'';
        skills = [ ./skills ];
      };
    }];
  };
in
agent.package
```

### Jail

On Linux, Prime Agent can run in a [jail.nix](https://sr.ht/~alexdavid/jail.nix/) bubblewrap sandbox:

```nix
programs.prime-agent.jail.enable = true;
```

The default jail permits network access and mounts the invocation's working directory read-write. It also retains `~/.prime/agent`, the configured `PRIME_AGENT_CODING_AGENT_DIR`, and Prime Agent's daemon socket state, allowing the daemon and IPython kernel to survive invocations. Files referenced by `environment.*.file` are mounted read-only automatically. Other home files and host tools remain unavailable unless explicitly exposed.

```nix
programs.prime-agent.jail.permissions = combinators: with combinators; [
  network
  mount-cwd
  (add-pkg-deps [ pkgs.gnumake pkgs.python3 ])
  (try-readonly (noescape "~/.gitconfig"))
];
```

### Selecting the Bun package

```nix
programs.prime-agent.package =
  inputs.prime-agent.packages.${pkgs.system}.prime-agent-bun;
```

## OpenRouter coding models

This fork ships a curated set of extra coding models in `models.json`, which
Prime Agent reads from `$PRIME_AGENT_CODING_AGENT_DIR/models.json` (default
`~/.prime/agent/models.json`). All four models are served through OpenRouter
so a single `OPENROUTER_API_KEY` covers them.

Enable it by pointing `programs.prime-agent.models` at the repository file and
select one of the models:

```nix
{ inputs, config, ... }:
{
  imports = [ inputs.prime-agent.nixosModules.default ];

  programs.prime-agent = {
    enable = true;
    models = inputs.prime-agent + "/models.json";
    settings = {
      defaultProvider = "openrouter";
      defaultModel = "google/gemini-3.7-flash";
    };
    environment.OPENROUTER_API_KEY.file =
      config.sops.secrets.openrouter-api-key.path;
  };
}
```

For Home Manager the same `programs.prime-agent.models` option works; keep the
model and provider settings and export `OPENROUTER_API_KEY`, for example:

```nix
environment.OPENROUTER_API_KEY = {
  file = config.sops.secrets.openrouter-api-key.path;
};
```

The four included models are:

| Model ID | Context | Max tokens | Reasoning |
| --- | --- | --- | --- |
| `google/gemini-3.7-flash` | 1,048,576 | 65,536 | yes |
| `qwen/qwen3.8-27b` | 262,144 | 131,072 | yes |
| `deepseek/deepseek-v4-pro-0813` | 1,048,576 | 384,000 | yes |
| `anthropic/claude-opus-5` | 1,000,000 | 128,000 | yes |

> [!WARNING]
> The `models.json` file only references the API key as `env:OPENROUTER_API_KEY`.
> Never hardcode a key in the file or in your Nix configuration. The actual
> secret is read from the environment at runtime (e.g. via
> `environment.OPENROUTER_API_KEY.file` for sops-nix secrets), never baked
> into `models.json`.

## Options

Generate the complete option reference in Markdown or HTML:

```bash
nix build .#docs-md
nix build .#docs-html
```

The output is available at `result/index.md` or `result/index.html`.
