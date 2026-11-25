# claude-dev-setup

Schritt-für-Schritt vom frisch installierten **Manjaro KDE Plasma** zur Arbeitsumgebung mit **Claude Code**, **Claude Desktop** und **GitHub** – inkl. Werkzeugen für Home-Assistant-Add-ons, pfSense-Pakete und Android-Debugging.

> Alle Befehle als normaler Benutzer im Terminal (Konsole). `sudo` nur wo angegeben.

---

## 1. System aktualisieren

```bash
sudo pacman-mirrors --fasttrack && sudo pacman -Syyu
```

Danach neu starten.

## 2. Basis-Pakete

Werkzeuge, die Claude Code ständig selbst aufruft (Suche, JSON, Git, Python, Node …):

```bash
sudo pacman -S --needed base-devel git github-cli yay \
  ripgrep fd jq yq curl wget unzip zip rsync openssh \
  python python-pip uv nodejs npm \
  shellcheck yamllint tmux htop bind nmap sqlite \
  wl-clipboard xclip
```

| Paket | Wofür |
|---|---|
| `ripgrep`, `fd` | schnelle Datei-/Codesuche |
| `jq`, `yq` | JSON/YAML in der Shell |
| `github-cli` (`gh`) | Issues, PRs, Releases direkt aus Claude |
| `uv` | Python-Tools & MCP-Server per `uvx` |
| `nodejs`, `npm` | JS-Tools, ältere MCP-Server per `npx` |
| `shellcheck`, `yamllint` | Linting für Skripte und HA-YAML |
| `wl-clipboard`, `xclip` | Bilder/Screenshots in Claude Code einfügen (Wayland/X11) |

## 3. Git & GitHub

```bash
git config --global user.name  "Dein Name"
git config --global user.email "du@example.com"
git config --global init.defaultBranch main

ssh-keygen -t ed25519 -C "manjaro-desktop"
gh auth login            # GitHub.com → SSH → vorhandenen Key hochladen → Browser-Login
gh auth status
```

## 4. Claude Code (CLI)

Offizieller Native-Installer (aktualisiert sich selbst):

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Neues Terminal öffnen, dann:

```bash
claude --version
claude          # beim ersten Start Login im Browser (Pro/Max/Team)
```

Falls `claude` nicht gefunden wird: `~/.local/bin` in den `PATH` aufnehmen  
(`echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc` bzw. `~/.zshrc`).

## 5. Claude Desktop (optional)

Offiziell gibt es nur ein `.deb` für Ubuntu/Debian (Beta). Für Manjaro gibt es ein Community-Paket im AUR, das das offizielle `.deb` umpackt:

```bash
yay -S claude-desktop
```

MCP-Konfiguration der Desktop-App: `~/.config/Claude/claude_desktop_config.json`  
(App vor dem Bearbeiten beenden, sonst überschreibt sie die Datei).

## 6. MCP-Server (Beispiel Home Assistant)

Eigenen HA-Benutzer + Long-Lived-Token anlegen, dann in Claude Code:

```bash
claude mcp add homeassistant \
  -e HOMEASSISTANT_URL=http://homeassistant.local:8123 \
  -e HOMEASSISTANT_TOKEN=<token> \
  -- uvx ha-mcp@latest

claude mcp list
```

Für Claude Desktop denselben Eintrag in `claude_desktop_config.json` unter `mcpServers` eintragen.

---

## 7. Home-Assistant-Add-ons entwickeln

Docker + Multi-Arch-Builds (aarch64/armv7 über QEMU):

```bash
sudo pacman -S --needed docker docker-buildx docker-compose \
  qemu-user-static qemu-user-static-binfmt
sudo systemctl enable --now docker
sudo usermod -aG docker $USER      # danach ab- und wieder anmelden
docker run --rm hello-world
```

Add-on lokal mit dem offiziellen HA-Builder testen (im Repo-Root):

```bash
docker run --rm --privileged \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$PWD":/data \
  ghcr.io/home-assistant/amd64-builder --amd64 --test --target /data/<addon-ordner>
```

Optional: `uv tool install pre-commit` für Lint-Hooks im Repo.

## 8. pfSense-Pakete entwickeln

pfSense-Pakete sind PHP + XML auf FreeBSD. Lokal reicht für Syntax-Checks und Deployment:

```bash
sudo pacman -S --needed php sshpass libxml2
php -l datei.inc                    # PHP-Syntax prüfen
xmllint --noout paket.xml           # XML prüfen
```

- SSH-Key auf der pfSense hinterlegen (*System → User Manager → admin → Authorized SSH Keys*), dann `ssh admin@pfsense` / `scp` / `rsync` zum Testen.
- Für echte `.pkg`-Builds: FreeBSD-VM (z. B. auf Proxmox) mit passender pfSense-Basisversion.

## 9. Android-Debugging

```bash
sudo pacman -S --needed android-tools android-udev scrcpy jdk17-openjdk
sudo usermod -aG adbusers $USER     # ab- und wieder anmelden
```

Am Handy: *Entwickleroptionen → USB-Debugging* aktivieren, dann:

```bash
adb devices          # Gerät bestätigen
adb logcat           # Logs
scrcpy               # Bildschirm spiegeln/steuern
```

Für App-Builds (Gradle) optional das SDK aus dem AUR: `yay -S android-sdk-cmdline-tools-latest`.

### Wireless Debugging & Handy-Auswahl (`bin/phone`)

Der Port von *Wireless Debugging* ändert sich bei jedem Einschalten, und das `adb` aus `android-tools` ist ohne mDNS gebaut (`adb mdns` geht nicht). Das Skript [`bin/phone`](bin/phone) findet die Handys deshalb per Avahi (WLAN) bzw. `adb devices` (USB), zeigt bei mehreren ein Auswahlmenü und startet `scrcpy`.

```bash
sudo pacman -S --needed avahi
sudo systemctl enable --now avahi-daemon

mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/betterbitsbites/claude-dev-setup/main/bin/phone \
  -o ~/.local/bin/phone
chmod +x ~/.local/bin/phone
```

Einmalig pro Handy koppeln: *Entwickleroptionen → Wireless Debugging → Gerät über Kopplungscode koppeln*, dann `adb pair <ip>:<port>` mit dem angezeigten Code.

| Aufruf | Wirkung |
|---|---|
| `phone` | ein Handy → direkt starten, mehrere → Menü |
| `phone pixel` | nach Name filtern |
| `phone usb` / `phone wifi` | nur USB / nur WLAN |
| `phone all` | ein scrcpy-Fenster pro Gerät |
| `PHONE_ASK=1 phone` | Menü immer anzeigen |
| `phone usb --turn-screen-off` | weitere Optionen gehen an scrcpy |

Hinweise: Handy und PC müssen im selben Subnetz/VLAN sein (sonst Avahi-Reflector auf der pfSense). Im Handy unter *Über das Telefon → Gerätename* eindeutige Namen vergeben. Feste DHCP-Leases halten die IPs stabil.

---

## 10. Feinschliff für Claude Code

- **Globale Regeln** in `~/.claude/CLAUDE.md` (z. B. Sprache, Commit-Stil, „keine Session-Links in Commits“).
- **Projektregeln** pro Repo in `CLAUDE.md` im Repo-Root.
- **Berechtigungen** in `~/.claude/settings.json`, z. B. häufige Befehle freigeben:

```json
{
  "permissions": {
    "allow": ["Bash(git status)", "Bash(git diff:*)", "Bash(rg:*)", "Bash(gh pr view:*)"]
  }
}
```

- Im Terminal: `Shift+Tab` wechselt den Berechtigungsmodus, `/help` zeigt alle Befehle.

## Kurz-Check

```bash
claude --version && gh auth status && docker info >/dev/null && adb version && uv --version
```

Alles grün → fertig.
