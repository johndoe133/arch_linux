# VS Code (OSS build)

Shared between machines. The OSS build (`code` package) has no Settings Sync, so settings live here instead.

## Settings

`settings.json` is symlinked into place, so changes made in VS Code land here directly. On a new machine:

```bash
ln -sf ~/Projects/arch_linux/vscode/settings.json "$HOME/.config/Code - OSS/User/settings.json"
```

Fonts need `ttf-jetbrains-mono-nerd` installed.

## Extensions

Not covered by `settings.json`. Save the list after installing or removing extensions:

```bash
code --list-extensions > ~/Projects/arch_linux/vscode/extensions.txt
```

Install them on another machine:

```bash
xargs -L1 code --install-extension < ~/Projects/arch_linux/vscode/extensions.txt
```
