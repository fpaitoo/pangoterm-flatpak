# pangoterm-flatpak

Flatpak packaging for [PangoTerm](https://pangoterm.com) — a modern cross-platform SSH client.

This repo is the **public staging / source-hosting** home for PangoTerm's
Flathub submission. Once the app is accepted to Flathub, the canonical
manifest lives at [`github.com/flathub/com.pangoterm.desktop`](https://github.com/flathub/com.pangoterm.desktop);
this repo remains as the public host of the source tarballs that the
Flathub manifest points at, plus our own staging copy of the manifest.

## What's in here

| File | Purpose |
|---|---|
| `com.pangoterm.desktop.yml` | Flatpak build manifest (GNOME 46 runtime, webkit2gtk-6.0) |
| `com.pangoterm.desktop.metainfo.xml` | AppStream metadata (description, screenshots, release notes, OARS content rating) |
| `com.pangoterm.desktop.desktop` | Linux desktop entry |
| `cargo-sources.json` | Pre-declared Cargo crate sources (Flathub builds are offline) |
| `node-sources.json` | Pre-declared npm package sources (production deps only) |
| `icons/*.png` | App icons in hicolor scheme sizes |
| `.github/workflows/*` | Automation |

## How to install PangoTerm via Flatpak

Once we're on Flathub:

```bash
flatpak install flathub com.pangoterm.desktop
flatpak run com.pangoterm.desktop
```

Before Flathub approval, you can install from a local build of this manifest:

```bash
# One-time: Flathub runtimes
flatpak remote-add --if-not-exists --user flathub \
  https://flathub.org/repo/flathub.flatpakrepo
flatpak install --user -y flathub \
  org.gnome.Platform//46 org.gnome.Sdk//46 \
  org.freedesktop.Sdk.Extension.rust-stable//23.08 \
  org.freedesktop.Sdk.Extension.node20//23.08

# Build + install locally
flatpak-builder --force-clean --user --install \
  /tmp/build-pangoterm com.pangoterm.desktop.yml

flatpak run com.pangoterm.desktop
```

## Release workflow

Source for PangoTerm lives in a private repo. This repo holds release
tarballs as GitHub Release assets — the Flathub manifest fetches those
tarballs by URL + sha256.

Per release:

1. **Tag the private source repo:**
   ```bash
   git tag v1.0.6
   git push origin v1.0.6
   ```

2. **GitHub Actions takes over.** A workflow on the private repo
   (`.github/workflows/publish-flatpak-tarball.yml`) runs on tag push.
   It `git archive`s the source, sha256sums it, creates a matching
   release on this public repo (`pangoterm-flatpak`), and attaches
   both files.

3. **Update this repo's manifest:**
   - `com.pangoterm.desktop.yml` — bump the `url:` and `sha256:` fields.
   - `com.pangoterm.desktop.metainfo.xml` — add a new `<release>` block
     at the top of `<releases>`.
   - If `Cargo.lock` or `package-lock.json` changed, regenerate
     `cargo-sources.json` and `node-sources.json` (see the private
     repo's `scripts/flatpak-gen-sources.sh`).

4. **Open a PR on `flathub/com.pangoterm.desktop`** with the updated
   manifest + metainfo + sources. Flathub buildbot auto-builds; on
   merge, users get the update within ~30 minutes via `flatpak update`.

## Why this repo exists

- PangoTerm's source repo is private (proprietary stack — backend,
  admin dashboard, etc.)
- Flathub builds require a **public** source URL
- This repo is the narrowest possible public surface: just the
  packaging files + release tarballs. The source code isn't
  committed here; it arrives as `.tar.gz` release assets generated
  from the private repo on each tag.

## Privacy / security notes for Flathub reviewers

The Flatpak bundle is sandboxed. Declared permissions in
`com.pangoterm.desktop.yml`:

| Permission | Why |
|---|---|
| `--share=network` | SSH, SFTP, sync backend, Google Drive API |
| `--socket=wayland` + `--socket=fallback-x11` | Display |
| `--device=dri` | WebKit GPU acceleration |
| `--talk-name=org.freedesktop.secrets` | OS keychain for password storage (via tauri-plugin-keyring) |
| `--socket=ssh-auth` | ssh-agent integration |
| `--filesystem=~/.ssh:ro` | Read user's SSH config + keys |
| `--filesystem=xdg-download`, `--filesystem=xdg-documents` | SFTP transfers |
| `--talk-name=org.freedesktop.Notifications` | Connection status notifications |

No passwords, keys, terminal output, or file contents leave the user's
device except (a) SSH traffic to their own servers and (b) optional
end-to-end encrypted backups to the user's own Google Drive
`appDataFolder`.

## License

MIT. See [LICENSE](LICENSE).
