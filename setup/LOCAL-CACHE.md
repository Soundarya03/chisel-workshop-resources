# Chisel workshop — local cache guide (for conference organizers)

**Goal:** participants must be able to complete the entire workshop setup
(Multipass install → VM launch → tool install → repo clones) using **only the
conference LAN**, without touching the WAN.

**How:** one machine on the LAN (wired Ethernet strongly preferred) serves a
directory of pre-downloaded artifacts over plain HTTP. Participants fetch the
Multipass installer, the VM image and a cloud-init file from it; the VM's
cloud-init then fetches everything else (snaps, LXD images, git repos) from
the same server.

**Where this lives:** this guide is `setup/LOCAL-CACHE.md` in the
`chisel-workshop-resources` workshop repo, which is structured as:

```
chisel-workshop-resources/
├── setup/                      # everything about provisioning
│   ├── cloud-init.yaml             # one-step setup, online version
│   ├── cloud-init-local-cache.yaml # one-step setup, LAN-cache version
│   └── LOCAL-CACHE.md              # this guide
└── session_content/            # exercises, slice definitions, etc.
```

The whole repo is itself one of the cached artifacts (§1.6): it is mirrored
onto the cache server and cloned into each participant's VM by cloud-init.

**Companion file:** `setup/cloud-init-local-cache.yaml` is the cloud-init the
VM consumes. Edit its `CACHE_BASE` placeholder once the cache server's IP is
known, and host the edited copy on the server itself (cache root, see §2).

---

## 1. Artifact checklist — everything to download

### 1.1 Multipass installers (for participants' host OSes)

| Artifact | File | Source (canonical URL) | Approx. size |
|---|---|---|---|
| Windows installer (x86_64) | `multipass-1.17.0+win-win64.msi` | `https://github.com/canonical/multipass/releases/download/v1.17.0/multipass-1.17.0%2Bwin-win64.msi` | ~78 MB |
| macOS installer (universal pkg) | `multipass-1.17.0+mac-Darwin.pkg` | `https://github.com/canonical/multipass/releases/download/v1.17.0/multipass-1.17.0%2Bmac-Darwin.pkg` | ~162 MB |
| Linux snap (amd64) | `multipass.snap` + `multipass.assert` | `snap download multipass` (see §3.1) | see download |
| Linux snap (arm64) — only if ARM-Linux attendees are expected | same, run on arm64 | `snap download multipass` on arm64 | see download |

> Pin everyone to the same Multipass version (1.17.0 at time of writing).
> `multipass launch` from a custom image URL and `--cloud-init` are exercised
> heavily here; a single known-good version removes a whole class of
> "works on my machine" issues.

### 1.2 Ubuntu 26.04 cloud images (one per host architecture)

From `https://cloud-images.ubuntu.com/releases/26.04/release/`:

| File | Approx. size | For |
|---|---|---|
| `ubuntu-26.04-server-cloudimg-amd64.img` | ~819 MB | Intel/AMD Linux, Windows, Intel Macs |
| `ubuntu-26.04-server-cloudimg-arm64.img` | ~897 MB | Apple Silicon Macs, ARM Linux |
| `SHA256SUMS` (same directory) | small | verification |

> **Do not skip the arm64 image.** On Apple Silicon, Multipass can only run
> arm64 VMs; an amd64-only cache leaves those participants stranded. (If you
> also want to cover the newer x86-64-v3 variant, `...-amd64v3.img` exists,
> but the standard `amd64` image is sufficient and safest.)

### 1.3 Snaps (per VM architecture: amd64 and arm64)

The workshop needs `lxd`, `rockcraft` (classic), `docker` and `chisel`, plus
whatever **base snaps** those pull in (`snapd`, `bare`, `coreNN...` — the
exact set changes over time, which is why §3.2 enumerates it from a live
system instead of hardcoding it).

Each snap is needed as **two files**: `<name>.snap` and `<name>.assert`
(the assertion lets snapd install offline *without* `--dangerous`).

Bundled as: `snaps-amd64.tar.gz` and `snaps-arm64.tar.gz` — see §3.2 for the
script that produces them.

### 1.4 LXD base image(s) for rockcraft (per architecture)

`rockcraft pack` builds inside LXD containers and pulls a base image
(e.g. an Ubuntu buildd image) from a remote on first use — per VM, that is a
few hundred MB over the network. Cache it:

* On a golden VM, run the workshop's first `rockcraft pack` exercise once
  with internet, then `lxc image list` to see what was pulled.
* Export each such image: `lxc image export <fingerprint> <name>.tar.gz`.
* Record `filename` + `alias` pairs in `manifest.txt` (see §3.3).

The local cloud-init imports these so first builds start fast.
**Verify in the dry run (§5) that a fully-offline `rockcraft pack` works** —
depending on how rockcraft resolves its remote, it may still try to check the
remote for updates; the dry run tells you definitively. If it does insist on
the remote, the fallback is simply that the first build per VM needs WAN once
(document it for participants), or pre-seed and ensure exercises reference
the locally imported alias.

### 1.5 `chisel-releases` git mirror

A `--mirror` clone of `https://github.com/canonical/chisel-releases` with
`git update-server-info` run inside it, so it can be cloned over plain HTTP
("dumb" protocol). A few MB. See §3.4.

### 1.6 The workshop repo itself (`chisel-workshop-resources`)

Participants need more than the cloud-init file: the repo carries the actual
**session content** (exercises, slice definitions under `session_content/`)
as well as the setup files. So the repo is also mirrored to the cache as a
second bare git repo, and the local cloud-init clones it into the VM at
`~/chisel-workshop-resources`.

* Source: the public URL the speaker publishes the repo to (placeholder
  `https://github.com/Soundarya03/chisel-workshop-resources.git` in §3.4 — get
  the real one from the speaker).
* **Refresh matters:** session content is still being committed as the event
  approaches. Re-sync this mirror shortly before the session (§3.4 shows
  how); if it changes mid-session, participants can simply `git pull` inside
  `~/chisel-workshop-resources` since its `origin` points at the LAN mirror.
* The edited `cloud-init-local-cache.yaml` (with `CACHE_BASE` filled in) is
  *additionally* copied to the cache root so participants can download just
  that one file onto their host before launching (§6).

### 1.7 Conditional: Docker Hub images

If any exercise does `docker pull <image>`, those images should be cached
too (`docker pull` + `docker save -o name.tar <image>` on a connected
machine; participants `docker load -i name.tar`, or you run a `registry:2`
mirror container). Check with the final exercise list; skip if unused.

---

## 2. Cache server layout

Serve one directory named `chisel-workshop-cache` (per-session name, since
organizers run caches for several sessions/speakers) that ends up like this:

```
chisel-workshop-cache/
├── installers/
│   ├── multipass-1.17.0+win-win64.msi
│   ├── multipass-1.17.0+mac-Darwin.pkg
│   └── linux/
│       ├── amd64/  (multipass.snap + multipass.assert)
│       └── arm64/  (same, if needed)
├── images/
│   ├── ubuntu-26.04-server-cloudimg-amd64.img
│   ├── ubuntu-26.04-server-cloudimg-arm64.img
│   └── SHA256SUMS
├── snaps/
│   ├── snaps-amd64.tar.gz
│   └── snaps-arm64.tar.gz
├── lxd-images/
│   ├── amd64/   (<name>.tar.gz files + manifest.txt)
│   └── arm64/   (same)
├── git/
│   ├── chisel-releases.git/          # bare mirror, update-server-info run
│   └── chisel-workshop-resources.git/   # bare mirror, update-server-info run
├── cloud-init-local-cache.yaml       # with CACHE_BASE filled in
└── SHA256SUMS                        # covering all of the above
```

Total size: roughly **4–5 GB** for both architectures plus all installers.

---

## 3. Preparation (run ahead of the event, with good internet)

### 3.1 Installers and VM images

```bash
mkdir -p chisel-workshop-cache/installers/linux/amd64 chisel-workshop-cache/images
cd chisel-workshop-cache/installers
curl -fLO "https://github.com/canonical/multipass/releases/download/v1.17.0/multipass-1.17.0%2Bwin-win64.msi"
curl -fLO "https://github.com/canonical/multipass/releases/download/v1.17.0/multipass-1.17.0%2Bmac-Darwin.pkg"

cd linux/amd64        # repeat in an arm64 dir on an arm64 machine if needed
snap download multipass
mv multipass_*.snap multipass.snap && mv multipass_*.assert multipass.assert

cd ../../../images
curl -fLO "https://cloud-images.ubuntu.com/releases/26.04/release/ubuntu-26.04-server-cloudimg-amd64.img"
curl -fLO "https://cloud-images.ubuntu.com/releases/26.04/release/ubuntu-26.04-server-cloudimg-arm64.img"
curl -fLO "https://cloud-images.ubuntu.com/releases/26.04/release/SHA256SUMS"
grep '\.img$' SHA256SUMS | sha256sum --check --ignore-missing
```

### 3.2 Snaps — "golden VM" method (run once per architecture)

Run this on a machine **of the target architecture** with internet. It
installs the tools exactly as the workshop would (so the correct base snaps
get pulled automatically), records the resulting revisions for
reproducibility, downloads every snap + assertion, and bundles them.

```bash
#!/usr/bin/env bash
# prep-snaps.sh — produces snaps-$(dpkg --print-architecture).tar.gz
set -euo pipefail

ARCH="$(dpkg --print-architecture)"
OUT="snaps-$ARCH"
mkdir -p "$OUT"

# 1. Install the workshop tools (pulls in the right base snaps too).
sudo snap install lxd
sudo snap install rockcraft --classic
sudo snap install docker
sudo snap install chisel

# 2. Record what's installed (audit trail / reproducibility).
snap list | tee "snap-revisions-$ARCH.txt"

# 3. Download every installed snap + assertion, renamed to <name>.{snap,assert}.
cd "$OUT"
snap list | awk 'NR>1 {print $1}' | while read -r s; do
  snap download "$s"
  mv "${s}"_*.snap   "${s}.snap"
  mv "${s}"_*.assert "${s}.assert"
done
cd ..

# 4. Bundle flat (files at the tarball root — the VM script expects that).
tar -czf "$OUT.tar.gz" -C "$OUT" .
echo "Done: $OUT.tar.gz — upload it to the cache server under snaps/"
```

Notes:

* **arm64:** run the same script on any arm64 Linux machine — e.g. an
  Apple-Silicon Mac (`multipass launch 26.04 --name arm64-prep` gives you an
  arm64 VM for free) or an arm64 cloud instance.
* Run this **close to the event date** so revisions match what online users
  would get; channels move.
* The tarballs extract to `<name>.snap`/`<name>.assert` pairs — exactly what
  `cloud-init-local-cache.yaml` consumes (`snap ack` + ordered
  `snap install`).

### 3.3 LXD images for rockcraft

On the golden VM (§3.2), after running the workshop's first `rockcraft pack`
once:

```bash
ARCH="$(dpkg --print-architecture)"
mkdir -p "lxd-images/$ARCH"
lxc image list
# For each image the exercise used (note fingerprint + alias from the output):
lxc image export <FINGERPRINT> "lxd-images/$ARCH/<sensible-name>.tar.gz"
echo "<sensible-name>.tar.gz <alias-from-lxc-image-list>" \
  >> "lxd-images/$ARCH/manifest.txt"
```

Repeat per architecture if both are needed (build the exercise once on each).

### 3.4 Git mirrors (two repos)

```bash
mkdir -p chisel-workshop-cache/git

# Upstream workshop material:
git clone --mirror https://github.com/canonical/chisel-releases.git \
  chisel-workshop-cache/git/chisel-releases.git
git -C chisel-workshop-cache/git/chisel-releases.git update-server-info

# The workshop repo itself (setup files + session_content/).
git clone --mirror https://github.com/Soundarya03/chisel-workshop-resources.git \
  chisel-workshop-cache/git/chisel-workshop-resources.git
git -C chisel-workshop-cache/git/chisel-workshop-resources.git update-server-info
```

`update-server-info` is **required** for cloning over plain HTTP.

To refresh a mirror later (important for the workshop repo as session
content keeps landing — re-run right before the session):

```bash
git -C chisel-workshop-cache/git/chisel-workshop-resources.git remote update
git -C chisel-workshop-cache/git/chisel-workshop-resources.git update-server-info
```

### 3.5 Checksums over everything

```bash
cd chisel-workshop-cache
find . -type f ! -name SHA256SUMS -print0 | sort -z | xargs -0 sha256sum > SHA256SUMS
```

---

## 4. Serving the cache on the day

```bash
cd ~/chisel-workshop-cache
python3 -m http.server 8000 --bind 0.0.0.0
```

(`python3 -m http.server` is threaded on modern Python and adequate for a
workshop room; for 100+ simultaneous participants use nginx or Caddy instead.)

Practicalities:

* Put the cache machine on **wired Ethernet** if at all possible; Wi-Fi
  serving 50 laptops is the bottleneck you're trying to remove.
* Note its LAN IP, e.g. `192.168.1.50`. Participants will use
  `http://192.168.1.50:8000`. Don't rely on mDNS/hostnames on a conference
  network.
* Open the port in the machine's firewall; disable sleep/suspend.
* Smoke-test from another machine: `curl -I http://<ip>:8000/images/`,
  `git ls-remote http://<ip>:8000/git/chisel-releases.git` and
  `git ls-remote http://<ip>:8000/git/chisel-workshop-resources.git`.

---

## 5. Dry-run checklist (organizers + speaker, ~1 week before)

- [ ] Every artifact in §2 fetched once through the cache from a client
      machine; `sha256sum --check` passes.
- [ ] Multipass installed from the cache on **Windows, macOS (Apple Silicon)
      and Linux** test machines.
- [ ] `cloud-init-local-cache.yaml` edited with the real `CACHE_BASE`.
- [ ] The **workshop-repo mirror is current** — it contains the latest
      `session_content/` commits (re-run §3.4 refresh if not).
- [ ] VM launched from the **cache-hosted image URL** on both amd64 and arm64
      hosts; `cloud-init status --wait` reaches `done`;
      `~/.workshop-ready` present; `snap list` shows lxd/rockcraft/docker/
      chisel; `docker ps` works as the ubuntu user; `chisel --version` runs;
      `~/chisel-releases` and `~/chisel-workshop-resources` both cloned.
- [ ] **With the host's WAN disabled**, run the workshop's first
      `rockcraft pack` exercise inside the VM end-to-end. This is the
      definitive test of the LXD-image seeding (§1.4).
- [ ] Fresh clones from both LAN git mirrors work.
- [ ] Full setup timed (target: under ~10 minutes per participant on LAN).

---

## 6. Participant quick-start (offline path) — for the workshop slides

1. Install Multipass from the cache:
   * **Windows/macOS:** download the `.msi`/`.pkg` from
     `http://<cache-ip>:8000/installers/` (browser) and install as usual.
   * **Linux:**
     ```bash
     curl -fLO http://<cache-ip>:8000/installers/linux/amd64/multipass.snap
     curl -fLO http://<cache-ip>:8000/installers/linux/amd64/multipass.assert
     sudo snap ack multipass.assert
     sudo snap install multipass.snap
     ```
2. Fetch the cloud-init and launch the VM **from the cached image**:
   ```bash
   curl -fLO http://<cache-ip>:8000/cloud-init-local-cache.yaml

   # Intel/AMD hosts:
   multipass launch http://<cache-ip>:8000/images/ubuntu-26.04-server-cloudimg-amd64.img \
     --memory 2G --disk 10G --name chisel-dev \
     --cloud-init cloud-init-local-cache.yaml --timeout 900

   # Apple Silicon / arm64 hosts: use ubuntu-26.04-server-cloudimg-arm64.img
   ```
3. Wait for setup and enter:
   ```bash
   multipass exec chisel-dev -- cloud-init status --wait
   multipass shell chisel-dev
   ```
4. Verify: `snap list`, `docker ps`, `lxc list`, `chisel --version`,
   `ls ~/chisel-releases`, and `ls ~/chisel-workshop-resources/session_content`.

---

## 7. Gotchas

* **Architecture mismatch** is the #1 failure mode: amd64 snaps/image on an
  Apple-Silicon host (or vice versa) fails in confusing ways. Keep the
  `amd64`/`arm64` split explicit everywhere, as above.
* **Missing `.assert` files** → offline `snap install` fails signature
  checks. Every `.snap` must ship with its `.assert` (the §3.2 script
  guarantees this pairing).
* **Install order matters** offline: `snapd` first, then base snaps
  (`bare`, `core*`), then the tools. The cloud-init script enforces this.
* **`rockcraft` needs `--classic`** when installed from a local file — the
  cloud-init script already passes it.
* **`git update-server-info` forgotten** → cloning the mirrors over HTTP
  fails. It must be run for **both** mirrors (§3.4), and **re-run after every
  refresh** of a mirror.
* **Stale workshop-repo mirror:** session content lands late. Re-sync the
  `chisel-workshop-resources.git` mirror right before the session (§3.4); tell
  participants they can `git pull` in `~/chisel-workshop-resources` anytime —
  its `origin` points at the LAN mirror.
* **Version drift:** re-run §3.2 close to the event; don't reuse months-old
  snap bundles.
* **Firewall/permissions on the cache machine** — test from a *different*
  device, not just localhost.
* If the venue LAN does provide some WAN, keep the original
  `setup/cloud-init.yaml` (online version) handy as a fallback path for
  stragglers.
