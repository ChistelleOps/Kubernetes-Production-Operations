# 01 — GKE authentication plugin not found on macOS

**Category:** Developer workstation / GKE access  
**Environment:** macOS with Zsh, Homebrew Cask Google Cloud CLI, GKE Standard  
**Status:** Resolved in the lab  
**Symptom:** `gke-gcloud-auth-plugin` was not found or not executable when obtaining GKE credentials.

## What we were trying to do

After provisioning our GKE cluster with Terraform, we needed to connect from the laptop using `kubectl`:

```bash
gcloud container clusters get-credentials \
  gcp-platform-lab-2026-gke-cluster \
  --zone us-central1-b \
  --project gcp-platform-lab-2026
```

The command wrote kubeconfig details but warned that the authentication plugin was missing or not executable. Kubernetes access from the laptop depends on **both** kubeconfig and a working authentication mechanism.

## The evidence we collected

```bash
gcloud components install gke-gcloud-auth-plugin
# All components are up to date

which gcloud
# /usr/local/bin/gcloud

which gke-gcloud-auth-plugin
# not found

readlink /usr/local/bin/gcloud
# /usr/local/Caskroom/gcloud-cli/587.0.0/google-cloud-sdk/bin/gcloud

ls -l /usr/local/Caskroom/gcloud-cli/587.0.0/google-cloud-sdk/bin/ | grep gke
# -rwxr-xr-x@ ... gke-gcloud-auth-plugin
```

The plugin binary **existed** and had executable permissions, but the shell could not locate it by name. This pointed to the shell's `PATH`, not a missing installation.

## Why this happens — Unix/Linux fundamentals

When a command has no slash, e.g. `gke-gcloud-auth-plugin`, the shell searches the directories in its `PATH` environment variable in order:

```bash
printf '%s\n' "$PATH" | tr ':' '\n'
```

`/usr/local/bin/gcloud` was a **symbolic link** (symlink) to the actual executable inside the versioned Homebrew Cask directory. A symlink is a filesystem entry that points to another path. Having a symlink for `gcloud` does not automatically provide a symlink for every other command in the same directory.

The plugin's permission string was `-rwxr-xr-x`:

- Initial `-`: regular file (not a directory).
- Owner `rwx`: read, write, execute.
- Group `r-x`: read, execute.
- Others `r-x`: read, execute.

Therefore, the binary was executable; its directory simply was not available in the shell's command search path.

### What `.zshrc` and `source` do

Zsh reads `~/.zshrc` when starting an interactive Zsh shell. Our machine initially had no `~/.zshrc`, which is valid: the shell doesn't require one. `touch` creates an empty file if it does not exist. `source` executes shell commands in a file **in the current shell**, so changes like `export PATH=...` take effect immediately without reopening the terminal.

## How we fixed it

```bash
# Create the config if absent
touch ~/.zshrc

# Derive the SDK bin directory from the gcloud symlink.
# The grep guard makes this safe to re-run (it adds the line only once).
grep -q 'readlink /usr/local/bin/gcloud' ~/.zshrc || \
  echo 'export PATH="$(dirname "$(readlink /usr/local/bin/gcloud)"):$PATH"' >> ~/.zshrc

# Apply the new config to the current terminal
source ~/.zshrc

# Check that the binary can now be resolved and executed
command -v gke-gcloud-auth-plugin
gke-gcloud-auth-plugin --version

# Verify Kubernetes access
kubectl get nodes -o wide
```

**Observed result:** The GKE node was `Ready`. The node reported internal IP `10.100.100.5`, no external IP, Container-Optimized OS, and containerd.

> **Portability note:** Homebrew installs to `/usr/local` on Intel Macs but `/opt/homebrew` on Apple Silicon, so the `/usr/local/bin/gcloud` path above may differ on your machine (check with `command -v gcloud`). This fix reflects our specific `/usr/local/bin/gcloud` **symlink**. If Google Cloud CLI is installed differently or `readlink` returns nothing, find the actual SDK directory with `gcloud info --format='value(installation.sdk_root)'` and adjust `PATH` accordingly. Avoid hardcoding the version `587.0.0`, which may change after an upgrade.

## A safe diagnostic sequence for next time

```bash
command -v gcloud
command -v gke-gcloud-auth-plugin
gcloud info --format='value(installation.sdk_root)'
ls -l /usr/local/bin/gcloud
printf '%s\n' "$PATH" | tr ':' '\n'
kubectl config current-context
kubectl get nodes
```

1. **Does the binary exist?** Investigate installation if not.
2. **Is it executable?** Inspect permissions with `ls -l`.
3. **Can the shell find it?** Inspect `PATH` and symlinks.
4. **Can it run?** Invoke `--version`.
5. **Can we reach the cluster?** Run `kubectl get nodes`. A different error here may indicate kubeconfig, IAM, or network issues.

## Outcome and prevention

The failure was a **local command discovery issue**, not a GKE cluster outage. We resolved it by placing the SDK's binary directory in the interactive shell's `PATH`. Keep bootstrap instructions in the repository README, and prefer version-independent SDK paths.

**Security:** Do not commit personal kubeconfig files, tokens, passwords, or cloud credentials.
