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
| Windows installer (x86_64) | `multipass-1.17.1+win-win64.msi` | `https://github.com/canonical/multipass/releases/download/v1.17.1/multipass-1.17.1%2Bwin-win64.msi` | ~78 MB |
| macOS installer (universal pkg) | `multipass-1.17.1+mac-Darwin.pkg` | `https://github.com/canonical/multipass/releases/download/v1.17.1/multipass-1.17.1%2Bmac-Darwin.pkg` | ~162 MB |
| Linux snap (amd64) | `multipass.snap` + `multipass.assert` | `snap download multipass` (see §3.1) | see download |
| Linux snap (arm64) — only if ARM-Linux attendees are expected | same, run on arm64 | `snap download multipass` on arm64 | see download |

> Pin everyone to the same Multipass version (1.17.1 at time of writing).
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

### 1.4 Pre-baked rockcraft base instance image (per architecture)

`rockcraft pack` builds inside LXD containers. On first use per VM,
craft-providers resolves an Ubuntu buildd image **by alias** from a remote
(`https://cloud-images.ubuntu.com/buildd/releases`) and runs apt-heavy setup
inside the container. Verified by dry run: a fingerprint-identical
`lxc image import` is *ignored* — alias resolution always needs the remote,
and LXD 6 only allows **https** simplestreams remotes (impractical to mirror
on a LAN). What *is* reused with zero network access is an **existing base
instance**, provided it is STOPPED and marked `FINISHED`.

So the cache ships a **pre-baked base instance image**: the fully set-up
build environment, published from a golden VM (§3.3), plus the exact
instance name/fingerprint recorded in `manifest.txt`. The local cloud-init
instantiates it in the `rockcraft` LXD project with the right metadata,
getting `rockcraft pack` past instance creation with no network at all.
(~635 MB per arch.)

**Not covered here (pending organizer decision):** two further build-time
dependencies — the OCI base image pull (`public.ecr.aws`, for rocks with
`base: ubuntu@...`) and Ubuntu archive access (apt, also used by
`chisel cut`). Making builds 100% offline additionally needs an OCI registry
mirror and an apt proxy on the cache; the dry run (§5) shows exactly how far
a build gets without them.

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

* Source: `https://github.com/Soundarya03/chisel-workshop-resources.git`
  (see §3.4 for the mirror commands).
* **Refresh matters:** session content is still being committed as the event
  approaches. Re-sync this mirror shortly before the session (§3.4 shows
  how); if it changes mid-session, participants can simply `git pull` inside
  `~/chisel-workshop-resources` since its `origin` points at the LAN mirror.
* The edited `cloud-init-local-cache.yaml` (with `CACHE_BASE` filled in) is
  *additionally* copied to the cache root so participants can download just
  that one file onto their host before launching (§6).

---

## 2. Cache server layout

Serve one directory named `chisel-workshop-cache` (per-session name, since
organizers run caches for several sessions/speakers) that ends up like this:

```
chisel-workshop-cache/
├── installers/
│   ├── multipass-1.17.1+win-win64.msi
│   ├── multipass-1.17.1+mac-Darwin.pkg
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
│   ├── amd64/   (rockcraft-base-baked.tar.gz + manifest.txt)
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

**How to run this section:** everything here needs decent internet.

* **§3.1, §3.4 and §3.5** can run as-is on any Linux machine with `curl`,
  `git` and `snapd` (including the cache server itself). They create and
  populate `chisel-workshop-cache/` in the current directory.
* **§3.2 and §3.3** must each be run **once per architecture** (amd64 *and*
  arm64, if we want to support both kinds of participant devices) inside a **disposable VM** 
  — they install snaps and run `rockcraft pack` as part of the process, so don't run them 
  on a system you care about. A Multipass VM is ideal (`multipass launch 26.04 --name prep`;
  on an Apple Silicon Mac that gives you arm64 for free). Move the outputs
  into place afterwards: `snaps-<arch>.tar.gz` → `chisel-workshop-cache/snaps/`
  and `lxd-images/<arch>/` → `chisel-workshop-cache/lxd-images/`.
* Two finishing steps live outside this section: copy the
  `cloud-init-local-cache.yaml` **with `CACHE_BASE` replaced by the server's
  IP** into the cache root (see §2), and serve the directory (see §4).

### 3.1 Installers and VM images

```bash
mkdir -p chisel-workshop-cache/installers/linux/amd64 chisel-workshop-cache/images
cd chisel-workshop-cache/installers
curl -fLO "https://github.com/canonical/multipass/releases/download/v1.17.1/multipass-1.17.1%2Bwin-win64.msi"
curl -fLO "https://github.com/canonical/multipass/releases/download/v1.17.1/multipass-1.17.1%2Bmac-Darwin.pkg"

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

### 3.3 Pre-baked rockcraft base instance image

On a golden VM of the target architecture, with internet. The easiest golden
VM is one launched with the **online** cloud-init (`setup/cloud-init.yaml`) —
it installs the tools, runs `lxd init --auto` and applies the Docker/LXD
firewall fix, without which `rockcraft pack` fails (no network in LXD
containers). Then:

1. Run a minimal `rockcraft pack` once — this creates and sets up the base
   instance in the `rockcraft` LXD project:

   ```bash
   mkdir ~/base-bake && cd ~/base-bake
   cat > rockcraft.yaml <<'EOF'
   name: base-bake
   base: ubuntu@24.04
   version: "0.1"
   summary: base bake
   description: minimal rock to create the rockcraft base instance
   platforms:
     amd64:      # use arm64 on an arm64 machine
   parts:
     nothing:
       plugin: nil
   EOF
   rockcraft pack
   ```

2. Publish the fully set-up base instance as an image, and harvest the exact
   values (they vary with rockcraft/image versions — never hardcode):

   ```bash
   ARCH="$(dpkg --print-architecture)"
   mkdir -p "lxd-images/$ARCH"
   BASE_NAME="$(lxc list --project rockcraft -c n --format csv | grep '^base-instance-')"
   echo "base instance: $BASE_NAME"
   lxc publish "$BASE_NAME" --project rockcraft    # note the new fingerprint
   FP="$(lxc image list --project rockcraft -c f --format csv | head -1)"
   # If several images are listed, use the fingerprint of the NEWEST (the
   # freshly published base instance), not the plain buildd one.
   lxc image export "$FP" --project rockcraft "lxd-images/$ARCH/rockcraft-base-baked"
   # lxc appends .tar.gz itself; keep the name rockcraft-base-baked.tar.gz

   cat > "lxd-images/$ARCH/manifest.txt" <<EOF
   ROCKCRAFT_BASE_IMAGE_FILE="rockcraft-base-baked.tar.gz"
   ROCKCRAFT_BASE_IMAGE_FINGERPRINT="$FP"
   ROCKCRAFT_BASE_INSTANCE_NAME="$BASE_NAME"
   EOF
   ```

3. Repeat per architecture (bake once on amd64 and once on arm64).

`manifest.txt` is sourced as shell by the VM setup script — keep the
`KEY="value"` format exactly.

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

Re-run this after any late edits — notably the `CACHE_BASE` edit to
`cloud-init-local-cache.yaml`, which typically happens on the day itself.

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

## 5. Dry-run checklist

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
      `rockcraft pack` exercise inside the VM. Instance creation must work
      fully offline (baked base, §1.4); expect the build to stop only at the
      OCI base-image pull / apt archive unless those proxies are added later.
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
2. Fetch the image + cloud-init script, then launch **from the local files**
   (Multipass rewrites `http://` URLs to `https://`, which a plain local
   HTTP server can't answer — `file://` avoids that):
   ```bash
   # Intel/AMD hosts (arm64 hosts: fetch the arm64 .img instead)
   curl -fLO http://<cache-ip>:8000/images/ubuntu-26.04-server-cloudimg-amd64.img
   curl -fLO http://<cache-ip>:8000/cloud-init-local-cache.yaml

   multipass launch file://$PWD/ubuntu-26.04-server-cloudimg-amd64.img \
     --memory 2G --disk 10G --name chisel-dev \
     --cloud-init cloud-init-local-cache.yaml --timeout 900
   ```
3. Wait for setup and enter:
   ```bash
   multipass exec chisel-dev -- cloud-init status --wait
   multipass shell chisel-dev
   ```
4. Verify: `snap list`, `docker ps`, `lxc list`, `chisel --version`,
   `ls ~/chisel-releases`, and `ls ~/chisel-workshop-resources/session_content`.
5. Remind participants: `rockcraft pack` and `chisel cut` still need
   **internet at build time** (OCI base image from `public.ecr.aws`, plus
   the apt archive). Only instance creation is offline via the baked base
   (§1.4). Plan for proxies (OCI registry and apt cache) if you want it fully offline.

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
* **Multipass rewrites `http://` to `https://`** for image and `--cloud-init`
  URLs (observed on 1.17): a plain-HTTP cache server can't be used directly
  in `multipass launch`. Always download first (curl/browser) and launch
  from `file://`.
* **`cloud-init status` shows `error` on a WAN-less LAN** — the stock image's
  own apt/`pollinate` step fails offline; harmless and does not affect the
  setup. Judge readiness by `~/.workshop-ready`, and set participant
  expectations accordingly.
* **Firewall/permissions on the cache machine** — test from a *different*
  device, not just localhost.
* If the venue LAN does provide some WAN, keep the original
  `setup/cloud-init.yaml` (online version) handy as a fallback path for
  stragglers.
