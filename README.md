# VibeServer

Turn an SDR into a network receiver for the [VibeSDR](https://vibesdr.net) apps and any web
browser.

This repository is the **APT repository and download host**. The source lives in
[Stuey3D/VibeSDR](https://github.com/Stuey3D/VibeSDR).

**Current release: 3.0** &nbsp;·&nbsp; [apt.vibesdr.net](https://apt.vibesdr.net)

### 📻 **[→ See one running: demo.vibesdr.net](https://demo.vibesdr.net)**

A live VibeServer in England with three receivers behind one address — an RTL-SDR, an SDRplay
RSP1B and an Airspy HF+. Open it in a browser and tune one. That is this software, unmodified,
on a Raspberry Pi on a home broadband line, so please be gentle with it.

> **New in 3.0:** several radios on one machine behind **one address and one port**, per-radio
> frequency **allow / block lists** with an ITU-region-aware band plan, an admin overview of the
> **whole machine** rather than one radio, admin sign-in **from the opening screen**, a safe
> **RTL serial-number editor** in the browser, **releasing a radio when nobody is listening** so
> another program can borrow it, and support for running **behind a reverse proxy**.

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

### Which systems this runs on

**Debian 13 (trixie) or newer, or Ubuntu 24.04 or newer** — including the current Raspberry Pi OS.

`apt` will tell you if your system is too old, naming `libc6` or `libstdc++6`. **If it does, do not
upgrade those libraries and do not upgrade your OS to get around it.** They sit underneath
everything else installed on the machine, and dragging them forward on a distribution that was not
built for them breaks other software — it has already cost one person a working OpenWebRX. Nothing
about VibeServer is worth that. Tell us instead: the fix belongs in our package, not on your
machine.

Older systems (Debian 12 bookworm, Ubuntu 22.04) are simply not built for yet.

> **Use a 64-bit OS.** The DSP has a fast path that only exists on 64-bit ARM. A 32-bit system
> silently loses all of it and runs roughly **13× slower** on the same hardware.

### On a Raspberry Pi 5, the install reboots your USB power budget

Nothing for you to do — this is here so it is not a surprise.

A Pi 5 ships with a conservative cap on how much current the USB ports may draw in total, and two
or three SDRs together exceed it. What that looks like is not a power warning: radios drop off the
bus under load, which reads exactly like a faulty cable, a bad hub, or "too much USB bandwidth".

So the install adds `usb_max_current_enable=1` to `/boot/firmware/config.txt` (keeping a backup
alongside it) on Pi hardware only, and only if it is not already there. **It takes effect at the
next reboot** — if you plan to run more than one radio, reboot before you go looking for trouble.
Use a power supply that can actually deliver it: the official 27 W one, or equivalent.

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

## More than one radio

Plug in a second or third SDR, open the **setup page** (the ⚙ on the opening screen, or
`http://<address>:48000/` before anything is configured), and each one gets its own tab.

**You still publish one address and one port.** Port 48000 becomes a front door: it shows the
opening screen listing every radio, and hands a visitor through to whichever one they pick. The
radios themselves run behind it and are never contacted directly, so **there is still exactly one
port to forward** on your router, and one address to give out.

Each radio is configured independently, and the first question on its tab is the one everything
else follows from:

- **Shared** — the radio sits on a band you choose and many people listen at once, each with their
  own tuning, mode, filters and decoders inside that window. This is the mode for a public
  receiver.
- **Single user** — one person at a time gets the whole radio, tuned anywhere it can reach. Others
  wait in a queue and are told where they are in it.

A radio in single-user mode has no time limit unless you set one, and it is the mode where the
frequency lists below matter most.

> **Identity comes from the serial number**, not the USB port. Unplug a radio, move it to another
> socket, reboot in a different order — its settings follow the device, not the position.
>
> Two radios that report the **same** serial cannot be told apart, which is the usual state of
> cheap RTL-SDR dongles straight from the factory. See the serial-number editor below.

### Copying settings between radios

If several radios share an antenna and a purpose, set one up and copy to the others. The copy
deliberately carries **only the frequency allow and block lists** — not the serial, not the port,
not the gain or the ppm correction, all of which belong to one specific piece of hardware.

---

## Deciding where people may tune

Per radio, you can give an **allow list**, a **block list**, or both.

- An **empty allow list means everything the radio can reach** — never nothing. An owner who only
  wanted to keep people off the airband must not lose their receiver by saying so.
- Entries can be typed as ranges (`87.5MHz-108MHz`, `530k-1710k`, bare numbers are Hz) or as
  **names** — `fm`, `mw`, `air`, `40m`, `2m` and the rest.
- **The band plan follows your location's ITU region.** A receiver in the United States is offered
  40 m ending at 7.300 and 2 m ending at 148; one in Europe gets 7.200 and 146. Same names,
  correct edges, taken from the location you gave the wizard.
- **The hardware always wins.** Allowing 40 MHz on an Airspy HF+ does not invent coverage it does
  not have; the setup page says *range not supported by hardware* rather than pretending.

A listener who tunes outside the permitted set is moved to the nearest edge of it, and told **why**
— *access restricted by server operator* when it is your rule, *range not supported by hardware*
when it is physics. Two different problems should never read as the same one.

---

## Sharing a radio with another program

If you also run OpenWebRX, `rtl_433`, a decoder, or anything else that wants the same dongle, turn
on **release the radio when nobody is listening** on that radio's tab.

When the last listener leaves, VibeServer waits out a short **grace period** — instant, 30 seconds,
1 minute, 5 minutes, your choice, so a browser refresh or a dropped Wi-Fi packet does not hand the
radio away — and then closes the device completely. The other program can open it. When a listener
next arrives, VibeServer takes it back.

Off by default: it is a deliberate choice, and a receiver that quietly lets go of its radio is not
what most people want.

> Radios that are **not** shared this way are still parked when idle — the front end stops
> streaming IQ rather than heating the room for nobody. A radio producing the landing page's
> spectrogram is the exception: it keeps running, because waking it and letting its AGC settle
> costs more than it saves.

---

## Changing an RTL-SDR's serial number

Two RTL dongles from the same factory usually report the **same** serial, and identical serials
cannot be told apart — not by VibeServer, not by anything else. So the setup page will do it for
you, without a terminal.

Each RTL radio's tab shows its current serial with a **Change** button. The dialog opens
**pre-filled with the serial it has now**, so you can put the cursor at the end and change one
digit rather than counting zeros; you type the new value twice and they must match.

Saving becomes a **reboot** rather than a restart, because the RTL2832U reads its EEPROM only at
power-on and only a reboot drops power to the USB ports. The page waits for the machine to come
back, **keeps you signed in**, and then reads the serial off the bus to confirm it actually took.

---

## Behind a reverse proxy

If VibeServer sits behind nginx, Caddy or similar, tell it which proxies to trust on the **Server**
tab. It then reads the real client address from `X-Forwarded-For` instead of logging every visitor
as `127.0.0.1` — which matters, because that address is what the ban list and the country lookup
work on.

**It is opt-in and empty by default, deliberately.** A server that believes `X-Forwarded-For`
without being told to is a server where any visitor can claim any address they like, walk straight
through a ban, and pin it on someone else. Only hops you list are trusted; the address is taken
from the last one you did not.

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

Enter the **admin password** on the opening screen and the whole machine unlocks: the title says
**ADMIN MODE**, radios that are busy become available to you anyway, and **ADMIN CONTROLS** and
**SETUP** appear on that screen — so you can look after the server without tuning a radio first.
Taking over a busy single-user radio evicts whoever is on it, and says so in the log.

The admin page reports the **whole machine, not the radio you happen to be on**. There is no point
sitting on the RTL and being blind to what the RSP is doing.

- CPU usage, clock speed, temperature, memory and uptime, with an hour of history
- every radio's front end — gain, AGC, overload — side by side
- **everyone connected to the server**: which radio, frequency, mode, decoder, bitrate and the CPU
  their own audio pipeline is costing, plus who is sitting on the opening screen and who is queued
- disconnect, and blocking by address, by range, or by an entire network
- connection history, paged, including **why** each one ended and what client it used
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

**Upgrading from 2.x changes nothing you have set.** A machine with one radio keeps its port, its
service and its behaviour; the multi-radio front door only appears once you configure a second one.

## Uninstalling

```bash
sudo apt remove vibeserver      # keeps your settings
sudo apt purge  vibeserver      # removes settings, ban list and state too
```

## Troubleshooting

| Symptom | Try |
|---|---|
| Is it running? | `sudo systemctl status vibeserver` |
| And the other radios? | `systemctl status 'vibeserver@*'` — one unit per extra radio, named after its serial |
| What is it saying? | `journalctl -u vibeserver -f` |
| No radio found | `lsusb` — is it listed? `groups` — does it say `plugdev`? If not, log out and in |
| A radio drops out under load on a Pi 5 | Reboot once after installing — the USB current budget needs it — and check the power supply |
| Two radios show as one | They report the same serial. Change one with the serial editor on its setup tab |
| "could not open … (is another program using it?)" | Something else has the radio. Note that running `vibeserver` by hand opens the settings screen **and holds the radio** — quit it before starting the service |
| Every visitor logs as `127.0.0.1` | You are behind a proxy — list it under trusted proxies on the Server tab |
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
