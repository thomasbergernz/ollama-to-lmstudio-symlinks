# Ollama ↔ LM Studio Symlink Utility

A fast, cross-platform Go CLI to create symbolic links between **Ollama** and **LM Studio**, sharing models bidirectionally without duplicating gigabytes of storage space.

---

## 🍴 Fork Notes (thomasbergernz)

This fork is identical to upstream [`qaribhaider/ollama-to-lmstudio-symlinks`](https://github.com/qaribhaider/ollama-to-lmstudio-symlinks) at commit `42cd89e`. It was security-reviewed before use.

### Security review summary

- **No network code** in the Go binary: no HTTP calls, telemetry or self-update.
- **Commands it runs:** only `ollama rm` (`delete` subcommand) and `ollama create` (`--reverse`). Both are called without a shell, with validated model names.
- **Deletions:** limited to symlinks or regular files (checked with `Lstat`) and empty LM Studio model directories. Nothing deletes whole folder trees.
- **Forward mode (Ollama → LM Studio)** only creates symlinks under `<lmstudio>/models/ollama/`. It never writes to `~/.ollama`.
- **Not used: `install.sh` / `uninstall.sh`.** They pipe `curl | bash`, download from upstream rather than this fork, and use `sudo`. The checksum comes from the same release, so it proves integrity, not authenticity. The macOS binary has only an ad-hoc signature.

### How it is installed here

Built from source and installed to `~/bin`, with no `sudo` and no release binary:

```bash
git clone https://github.com/thomasbergernz/ollama-to-lmstudio-symlinks.git ~/src/ollama-to-lmstudio-symlinks
cd ~/src/ollama-to-lmstudio-symlinks
go build -o ollama-symlinks ./cmd/ollama-symlinks
mkdir -p ~/bin && mv ollama-symlinks ~/bin/
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
```

`--version` reports `dev` because the build omits `-ldflags`. Use `./build.sh` to embed the `VERSION` file instead.

### What is actually used

Only **forward mode** (Ollama → LM Studio), pointed at the current LM Studio models dir. The tool's default is the older `~/.cache/lm-studio/models`, so pass the directory explicitly:

```bash
# preview, then link all Ollama models into LM Studio
ollama-symlinks --interactive=false --dry-run --lmstudio-dir ~/.lmstudio/models
ollama-symlinks --interactive=false --lmstudio-dir ~/.lmstudio/models

# check links / find broken ones
ollama-symlinks status --lmstudio-dir ~/.lmstudio/models
```

Links land in `~/.lmstudio/models/ollama/<model>/<model>.gguf` and point to `~/.ollama/models/blobs/sha256-*`. They appear in LM Studio under the **ollama** provider.

**Maintenance:** when Ollama updates or removes a model, its blob is replaced and the LM Studio link breaks. Run `ollama-symlinks cleanup`, then re-run the link command. Deleting a model in LM Studio removes only the link.

**Deliberately not used:**
- `--reverse`, which writes into `~/.ollama` and registers models.
- `delete` on the Ollama side, which runs `ollama rm` and removes real models.
- `--hardlinks`, which is only needed on Windows.

### Updating from upstream

```bash
cd ~/src/ollama-to-lmstudio-symlinks
git fetch upstream && git diff HEAD upstream/main   # review before merging
git merge upstream/main
go build -o ~/bin/ollama-symlinks ./cmd/ollama-symlinks
```

---

## ✨ Features

- 🔄 **Bidirectional Linking**: Link Ollama models to LM Studio OR LM Studio models to Ollama.
- 🧩 **Sharded GGUF Support**: Automatically groups and links multi-file models (`00001-of-0000N.gguf`).
- 👁️ **Vision & Multimodal Adapters**: Pairs and links multimodal projector adapters (`mmproj-*.gguf`).
- 🌐 **Registries & Namespaces**: Full support for Hugging Face (`hf.co/...`), custom namespaces, and standard models.
- 📊 **Storage Analytics (`status`)**: Measures active links, detects broken symlinks, and calculates deduplicated disk savings.
- 🧹 **Dual-Directory Cleanup (`cleanup`)**: Scans both LM Studio and Ollama blobs to find and wipe broken ghost-links.
- 🛡️ **Safe & Interactive (`delete`)**: Selectively remove links without touching original weights. Never overwrites existing files.
- 💻 **Cross-Platform**: Works natively on macOS, Linux, and Windows (with `--hardlinks` fallback).

---

## 📥 Installation

### Quick Install (macOS & Linux)
```bash
curl -fsSL https://raw.githubusercontent.com/qaribhaider/ollama-to-lmstudio-symlinks/main/install.sh | bash
```

### Manual Download
Download pre-compiled binaries for macOS, Linux, or Windows from the [Releases](https://github.com/qaribhaider/ollama-to-lmstudio-symlinks/releases) page:
```bash
chmod +x ollama-symlinks-*
sudo mv ollama-symlinks-* /usr/local/bin/ollama-symlinks
```

---

## 🚀 Quick Start

Run the interactive terminal menu:

```bash
ollama-symlinks
```

The interactive UI allows you to view storage savings, link models forward or reverse, or clean up links with arrow-key navigation.

---

## 📖 Command Cheatsheet

### 1. View Status & Storage Savings
Inspect active links, broken symlinks, and exact disk space saved across both applications:
```bash
ollama-symlinks status
# Add --verbose to list individual files and targets
ollama-symlinks status --verbose
```

### 2. Link Ollama → LM Studio (Forward Mode)
Scans Ollama manifests and creates symlinks inside LM Studio:
```bash
# Interactive selection
ollama-symlinks

# Automated (links all discovered models without prompts)
ollama-symlinks --interactive=false

# Dry run (preview without modifying files)
ollama-symlinks --dry-run
```

### 3. Link LM Studio → Ollama (Reverse Mode)
Scans LM Studio GGUFs and registers them with Ollama:
```bash
# Interactive selection with default 'lms-' prefix
ollama-symlinks --reverse

# Automated with custom model prefix
ollama-symlinks --reverse --name-prefix="myorg" --interactive=false
```

### 4. Cleanup Broken Symlinks
Scans both LM Studio and Ollama blobs for orphaned symlinks (e.g. after running `ollama rm`):
```bash
# Preview broken links
ollama-symlinks cleanup --dry-run

# Interactively remove broken links
ollama-symlinks cleanup
```

### 5. Selectively Delete Symlinks
Safely remove symlinks without touching original model data:
```bash
# Remove models linked into LM Studio
ollama-symlinks delete --from lmstudio

# Remove models registered in Ollama
ollama-symlinks delete --from ollama
```

---

## ⚙️ CLI Reference

### Global Flags

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--interactive`, `-i` | `true` | Launch interactive selection menus (`false` for automation). |
| `--reverse` | `false` | Enable Reverse Mode: Link LM Studio models to Ollama. |
| `--name-prefix` | `lms` | Prefix for models imported into Ollama (e.g. `lms-llama-3`). |
| `--ollama-dir` | *Auto* | Path to Ollama models directory (default: `~/.ollama/models`). |
| `--lmstudio-dir` | *Auto* | Path to LM Studio models directory (default: `~/.cache/lm-studio/models`). |
| `--skip-provider` | `ollama` | Provider folder name in LM Studio where links are placed. |
| `--hardlinks` | `false` | Use hard links instead of symlinks (resolves Windows permissions / 0-byte issues). |
| `--skip-checks` | `false` | Skip pre-flight executable validation checks. |
| `--dry-run` | `false` | Preview operations without creating or modifying files. |
| `--verbose` | `false` | Display verbose diagnostic logs. |
| `--deep-scan` | `false` | Scan all available Windows drives for model folders. |
| `--version` | `false` | Display binary version. |

### Subcommands

* `ollama-symlinks status` — Shows storage savings, link counts, and health.
* `ollama-symlinks cleanup` — Scans LM Studio and Ollama for broken links and removes them.
* `ollama-symlinks delete --from <lmstudio|ollama>` — Interactively selects and removes symlinks.

---

## 🗑️ Uninstallation

To remove the binary from your system:

### Automatic (macOS & Linux)
```bash
curl -fsSL https://raw.githubusercontent.com/qaribhaider/ollama-to-lmstudio-symlinks/main/uninstall.sh | sudo bash
```

### Manual
```bash
sudo rm -f /usr/local/bin/ollama-symlinks
```

*(Note: Uninstalling the binary leaves your existing symlinks and original model files intact. Use `ollama-symlinks delete` prior to removal if you wish to clean up created links first).*

---

## 📚 Documentation & Guides

- 🪟 **[Troubleshooting & Windows Guide](TROUBLESHOOTING.md)**: Windows Developer Mode, Administrator rights, hard link resolution for 0-byte models, and FAQ.
- 🛠️ **[Contributing Guide](CONTRIBUTING.md)**: Building from source, project architecture, and developer workflows.

---

## 📄 License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
