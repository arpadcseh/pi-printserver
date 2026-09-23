# Raspberry Pi print server (Ansible)

Turns a Raspberry Pi with a USB-attached **Brother HL-L2312D** into a print
server that Apple devices see as AirPrint and Android devices see as Mopria.

Reproduces a setup built and verified by hand on Debian 13 (trixie) / Pi 3B+,
including the two failures that setup hit.

## Quick start

    pip install ansible-core
    ansible-playbook site.yml --ask-become-pass

`--ask-become-pass` is required: Raspberry Pi OS images written by the Pi
Imager do **not** grant passwordless sudo.

> Do not put `ansible_become_pass` in plain `group_vars`. If you want to stop
> typing the password, use an `ansible-vault` encrypted file — `.gitignore`
> already excludes the usual vault filenames.

Check what would change without touching anything:

    ansible-playbook site.yml --check --ask-become-pass

Re-run only the post-setup checks:

    ansible-playbook site.yml --tags verify --ask-become-pass

## What it does

| Step | Why |
|---|---|
| Installs `cups`, `cups-filters`, `printer-driver-brlaser`, `avahi-daemon`, `iw` | print server, raster pipeline, driver, mDNS, wifi tuning |
| **Blacklists `usblp`** | see below — without this nothing prints |
| Enables sharing, remote admin, web UI, `ServerAlias *` | makes CUPS reachable and manageable from the LAN |
| Auto-discovers the USB URI and the matching brlaser PPD | no serial numbers hard-coded; works on a replacement printer |
| Creates the queue, shares it, makes it the default | the thing phones actually find |
| Applies print defaults (size, resolution, toner save, duplex) | the only lever phones have — see below |
| Installs a `wifi-powersave-off` systemd unit | stops the Pi sleeping through mDNS queries |
| Verifies services, mDNS, usblp, queue state | a run that reports success actually is one |

## The two non-obvious failures this encodes

**1. `usblp` silently breaks USB printing.** The `usblp` kernel module claims
the printer's USB interface and fights CUPS's libusb backend for it. Jobs hang
forever at *"Sending data to printer"*, nothing is logged as an error, and
`dmesg` shows `usblp` registering and removing the device in a loop. The role
blacklists it and unloads it immediately (the blacklist alone only applies at
next boot). **If printing ever hangs like that, check `lsmod | grep usblp`
first.**

**2. Wifi power saving breaks discovery intermittently.** It defaults to on,
which makes the Pi slow to answer mDNS, so phones sporadically "can't find"
the printer. A NetworkManager `conf.d` drop-in is **not** enough — a
netplan-generated connection profile overrides it. Hence the systemd unit.

## Printer defaults are the only lever your phones have

AirPrint and Mopria cannot set vendor-specific options. This printer also
advertises `print-quality-supported = normal` — a single value — so the
draft/normal/best selector on a phone does nothing. Whatever is in
`printserver_print_defaults` is what every phone gets.

Lighter printing / less toner — uncomment in `group_vars/printservers.yml`:

    printserver_print_defaults:
      brlaserEconomode: "True"    # toner save
      Resolution: 600dpi          # lighter and faster

There is no toner-density slider: brlaser does not implement one, and
Brother's own driver (which has one) is x86-only.

## Variables

All tunables live in `roles/printserver/defaults/main.yml`; override them in
`group_vars/printservers.yml`. The ones you are most likely to touch:

| Variable | Default | Notes |
|---|---|---|
| `printserver_queue` | `Brother_HL-L2312D` | CUPS queue name |
| `printserver_device_uri` | `""` | empty = auto-discover; set to pin one of several printers |
| `printserver_ppd_match` | `Brother HL-L2310D series` | the HL-L2312D enumerates as an L2310D |
| `printserver_print_defaults` | A4 / 1200dpi / no toner save | what phones get |
| `printserver_disable_wifi_powersave` | `true` | set `false` on an ethernet-only Pi |
| `printserver_test_page` | `false` | `true` prints a page during verify |

## Adapting to a different printer

1. Plug it in, then on the Pi: `lpinfo -v` (URI) and `lpinfo -m | grep -i brlaser` (drivers).
2. Point `printserver_ppd_match` at the model string you saw, and change
   `printserver_usb_uri_match` / `printserver_usb_vendor_id` if it is not a Brother.
3. `printserver_print_defaults` keys are PPD option names — confirm them with
   `lpoptions -p <queue> -l`, since they differ per driver.

## Notes

- The queue accepts jobs from anyone on the LAN with no authentication, which
  is the normal CUPS home-server posture. Tighten `printserver_cupsctl_args`
  if you want otherwise.
- Web UI: `http://<host>:631`. Admin actions force TLS and need a member of
  the `lpadmin` group: `https://<host>:631/admin/`. The certificate is
  self-signed, so expect a browser warning.
- This driver reports no toner level, so nothing will warn you when the
  cartridge runs low.

## License

MIT — see [LICENSE](LICENSE). Use it, change it, redistribute it, at home or
anywhere else. No warranty.
