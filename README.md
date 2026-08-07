# VibeServer

Turn an SDR into a network receiver for the [VibeSDR](https://vibesdr.net) apps and any web
browser.

This repository is the **APT repository and download host**. The source lives in
[Stuey3D/VibeSDR](https://github.com/Stuey3D/VibeSDR).

**Current release: 2.0** &nbsp;·&nbsp; [apt.vibesdr.net](https://apt.vibesdr.net)

---

## Install on a Raspberry Pi (or any 64-bit ARM Debian / Ubuntu)

Paste the whole block. **You do not need to install any libraries yourself and you do not need to
build anything** — apt pulls in everything VibeServer needs.

```bash
curl -fsSL https://apt.vibesdr.net/KEY.gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/vibesdr.gpg

echo "deb [arch=arm64 signed-by=/usr/share/keyrings/vibesdr.gpg] https://apt.vibesdr.net stable main" \
  | sudo tee /etc/apt/sources.list.d/vibesdr.list

sudo apt update
sudo apt install vibeserver
```

> **`arch=arm64` is deliberate.** 64-bit Raspberry Pi OS enables multi-arch, so apt asks every
> repository for `armhf` as well — and ours is arm64 only, which produces a harmless but alarming
> `Skipping acquire of configured file 'main/binary-armhf/Packages'` on every update. Naming the
> architecture stops apt asking for one we do not publish.


The first two commands tell your machine to trust the packages and where to find them — you only
ever do that once. From then on VibeServer updates alongside everything else on the machine.

> **Use a 64-bit OS.** The DSP has a fast path that only exists on 64-bit ARM. A 32-bit system
> silently loses all of it and runs roughly **13× slower** on the same hardware.

### Then log out and back in

The install adds you to the `plugdev` group, which is what lets the software open a USB radio
without running as root — and group membership only takes effect on a **new login**. Over SSH,
disconnect and reconnect.

Skip this and the next step reports no radio, even with one plugged in.

### Set it up

```bash
vibeserver
```

A short wizard: a name for the receiver, roughly where it is, which radio, and an admin password.
When you finish **it starts the server for you** — there is no separate `systemctl start`. It
prints the address to use.

### Open it

```
http://<the address it printed>:48000
```

`http://vibeserver.local:48000` usually works on the same network. That is the receiver itself —
waterfall, audio, decoders — and the same server the VibeSDR apps connect to.

---

## ⚠ The waterfall looks quiet at first. That is deliberate.

A new install starts the radio at **minimum gain, in manual mode**.

An unknown antenna on an unknown band can overload a front end the moment it is switched on, and a
receiver that starts quiet is far easier to recover from than one that starts overloaded and
distorts everything.

Open the **menu** and raise the gain until the noise floor lifts and signals appear. Whatever you
set is remembered, and updates never change a gain you have set yourself.

---

## If you have an SDRplay RSP

`apt` cannot install SDRplay's driver, and that is not an oversight: SDRplay distribute it under
their own licence and it is not ours to redistribute. **RTL-SDR and Airspy HF+ need nothing
extra.**

On a headless Pi there is no browser, so fetch it from the command line:

```bash
curl -fLO https://www.sdrplay.com/software/SDRplay_RSP_API-Linux-3.15.2.run
chmod +x SDRplay_RSP_API-Linux-3.15.2.run
sudo ./SDRplay_RSP_API-Linux-3.15.2.run
```

The installer is **interactive** — it pages a licence (space to scroll), then asks twice for `y`.
One file covers every architecture including ARM64; there is no separate Pi download despite the
website's wording.

**Then reboot.** The installer asks for it and means it — the driver's service and the USB
permissions both need to come up cleanly. Replugging the radio is *not* enough.

```bash
sudo reboot
# once it is back:
systemctl status sdrplay        # should be active (running)
```

> The unit is **`sdrplay`**. Older API versions called it `sdrplay_apiService`.

You can install the driver before or after VibeServer. If you add an RSP later, just
`sudo systemctl restart vibeserver` — the driver is loaded at runtime.

Without it, VibeServer runs perfectly and reports no radio, which reads as broken hardware rather
than a missing driver — so the message it prints names the download.

---

## The admin page

Connect using the **admin password** — either on the opening screen, or unlock the protected
settings from the menu once connected. A **SERVER ADMIN** button then appears at the bottom of the
menu:

- CPU usage, temperature, memory and uptime, with an hour of history
- what the radio's front end is actually doing — gain, AGC, overload
- who is listening, from where, on what frequency — with disconnect and block
- blocking by address, by range, or by an entire network
- connection history, including **why** each one ended
- maintenance: check for updates, install updates, restart, reboot — each showing its output

Admin controls **re-lock after 30 minutes** with no interaction. The session, the audio and any
running decoder carry on; only the ability to *change* things is withdrawn, so a forgotten browser
tab cannot be used to meddle with your receiver.

There is deliberately **no web terminal**. The admin password's other job is guarding the gain
controls, and a shell behind it would be full control of the machine and the network around it — on
a receiver that may be publicly listed and served over plain HTTP. The buttons above cover what a
terminal was wanted for. For a real shell from a phone, put the machine on Tailscale or WireGuard.

### On privacy

Country flags and network names come from the regional internet registries' own published data and
[iptoasn.com](https://iptoasn.com) — both public domain, downloaded once and queried locally.
**No visitor's address is ever sent anywhere.**

---

## Updating

```bash
sudo apt update && sudo apt upgrade
```

Settings survive and the server restarts on the new version. The admin page's **Install updates**
button does the same thing and shows you exactly what it is doing.

## Uninstalling

```bash
sudo apt remove vibeserver      # keeps your settings
sudo apt purge  vibeserver      # removes settings, ban list and state too
```

## Troubleshooting

| Symptom | Try |
|---|---|
| Is it running? | `sudo systemctl status vibeserver` |
| What is it saying? | `journalctl -u vibeserver -f` |
| No radio found | `lsusb` — is it listed? `groups` — does it say `plugdev`? If not, log out and in |
| "could not open … (is another program using it?)" | Something else has the radio. Note that running `vibeserver` by hand opens the settings screen **and holds the radio** — quit it before starting the service |
| udev rules not applied | `sudo udevadm control --reload-rules && sudo udevadm trigger`, then replug |

## Where things go

| Path | Purpose |
|---|---|
| `/usr/bin/vibeserver` | the program |
| `/etc/vibeserver/` | your settings — never overwritten by an update |
| `/var/lib/vibeserver/` | ban list, connection history, spectrogram, station and country data |

---

Built from the [VibeSDR](https://github.com/Stuey3D/VibeSDR) tree — the same DSP engine, decoders
and protocol the apps use, which is why a fix in one reaches all of them.
