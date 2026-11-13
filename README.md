# UbuCon India 2026: Chisel workshop resources

This repo holds the resources for the [Chisel](https://github.com/canonical/chisel) workshop at UbuCon India 2026 ("Lean, Clean & Secure: A Hands-on Intro to Chisel").

More specifically, it contains two parts:

1. `session_content/`: the slides, actual slice definition files and examples for the workshop, so you can follow along and try things out on your own machine, and
2. `setup/`: instructions for setting up everything you need. This second part has been broken down below.

## Setup

As a participant, the process is made as simple as possible for you! At most, you'll need to perform 3 steps; in the best case, only one.

> **At the venue, everything is served from a local cache** on the conference
> network — much faster and more reliable than the Wi-Fi's internet link.
> The organizers will announce the cache server's address; wherever you see
> `http://<cache-ip>:8000` below, substitute that address.
> (Setting up ahead of the event, from home? See the note at the bottom.)

### Step 1: If you run Linux, ensure you have snapd installed and available.

If you use Ubuntu, you already have snapd available and ready. Proceed to Step 2.

If you use another Linux distribution, and you don't have snapd installed, please follow the instructions at https://snapcraft.io/docs/tutorials/install-the-daemon/.

(On Windows or macOS? Skip this step. You don't need snapd.)

### Step 2: Ensure you have Multipass installed and available.

We'll be using [Multipass](https://canonical.com/multipass/docs/stable/) to spin up an Ubuntu VM and work within this VM. This is to ensure that everyone has the same environment, and that you don't have to install anything (besides Multipass itself) on your host machine.

If you have Multipass already installed, you can skip this step — check with `multipass version`. Otherwise:

**Linux**

```bash
# From the venue cache:
curl -fLO http://<cache-ip>:8000/installers/linux/amd64/multipass.snap
curl -fLO http://<cache-ip>:8000/installers/linux/amd64/multipass.assert
sudo snap ack multipass.assert
sudo snap install multipass.snap

# (With internet access, simply: sudo snap install multipass)
```

**Windows**

Download `http://<cache-ip>:8000/installers/multipass-1.17.0+win-win64.msi` in your browser and run the installer (administrator rights required). Open a **new** terminal afterwards, so that `multipass` is on your `PATH`.
(With internet access, the same file is available on the [Multipass releases page](https://github.com/canonical/multipass/releases/latest).)

**macOS**

Download `http://<cache-ip>:8000/installers/multipass-1.17.0+mac-Darwin.pkg` in your browser and run the installer package.
(With internet access, the same file is available on the [Multipass releases page](https://github.com/canonical/multipass/releases/latest).)

Verify with `multipass version`, then proceed to Step 3.

### Step 3: Set up your dev environment with Multipass. We'll be working within this.

This has been made simple by using cloud-init (if you're curious, see [here](https://github.com/canonical/cloud-init)). Everything the VM needs (tools, repos and this workshop's session content itself) is installed automatically, straight from the local cache.

#### Step 3.1: First, download the VM image and the cloud-init file from the cache (quick, since it's all local):

**On Intel/AMD machines** (Windows, most Linux laptops, Intel Macs):

```bash
curl -fLO http://<cache-ip>:8000/images/ubuntu-26.04-server-cloudimg-amd64.img
```

**On Apple Silicon Macs:**

```bash
curl -fLO http://<cache-ip>:8000/images/ubuntu-26.04-server-cloudimg-arm64.img
```

**Everyone** (on Windows, a browser download works just as well):

```bash
curl -fLO http://<cache-ip>:8000/cloud-init-local-cache.yaml
```

#### Step 3.2: Then, from the same directory, launch the VM:

```bash
# Intel/AMD. Apple Silicon: use the arm64 .img filename you downloaded.
multipass launch file://$PWD/ubuntu-26.04-server-cloudimg-amd64.img \
  --memory 2G --disk 10G --name chisel-workshop \
  --cloud-init cloud-init-local-cache.yaml \
  --timeout 900
```

> Why download first? Multipass 1.17 rewrites `http://` URLs to `https://`
> when you pass them to `launch`, and a plain local HTTP server can't answer
> TLS. Launching from `file://` sidesteps that completely.
>
> **On Windows (PowerShell):** write the launch command on a single line
> (no `\` continuations), and point `file://` at the full path where you
> saved the image, e.g. `file://C:/Users/you/Downloads/ubuntu-26.04-....img`.

This kicks off the automatic setup. If Multipass reports a timeout, don't worry. The setup keeps running in the background.

**Wait for the setup to finish** (a few minutes), then shell into the VM:

```bash
multipass exec chisel-workshop -- cloud-init status --wait
multipass shell chisel-workshop
```

> With no internet access, `cloud-init status` may report `error` — that's
> the VM image's own optional apt step failing to reach the Ubuntu archive;
> it does **not** affect the workshop setup. The reliable readiness check is
> the marker file in the next step.

**Check that everything is ready** — inside the VM:

```bash
ls ~/.workshop-ready          # marker file left behind by the setup
snap list                     # should show lxd, rockcraft, docker and chisel
docker ps                     # should work without sudo
chisel --version
ls ~/chisel-releases ~/chisel-workshop-resources/session_content
```

You're all set — everything from here on happens inside this VM.

---


### Setting up before the event, or have solid internet anyway? For Step 3, use this online variant instead:

```bash
multipass launch 26.04 --memory 2G --disk 10G --name chisel-workshop \
  --cloud-init https://raw.githubusercontent.com/Soundarya03/chisel-workshop-resources/main/setup/cloud-init.yaml \
  --timeout 900
```

---

**Organizers**: see `setup/LOCAL-CACHE.md` for what resources to prep on the local cache and how.
