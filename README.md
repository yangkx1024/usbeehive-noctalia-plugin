# USB Beehive (Noctalia plugin)

Shows connected USB devices and USB-C charging wattage in the Noctalia v5 bar, using
[usbeehive](https://github.com/abrauchli/usbeehive).

- **Bar widget**: `⚡ 45W` while a USB-C port is charging (`↗` while supplying power), otherwise
  a USB icon with the device count. The icon turns `tertiary` when usbeehive flags a cable or charger
  bottleneck. Left click opens the panel; right click refreshes.
- **Panel**: power summary, charging diagnostics, and an expandable device list (IDs, driver, link
  speed, cable e-marker, charger PDOs with the active profile marked).
- **Notifications**: device connect/disconnect, and new charging bottlenecks.

## Requirements

- `usbeehive` (`cargo install usbeehive`, needs libudev and pkg-config)
- `jq`: usbeehive pretty-prints `--watch --json` across many lines, and jq compacts each snapshot
  into one line so the plugin can read the stream line by line.

## How it works

`service.luau` owns everything that touches usbeehive:

1. `usbeehive --watch --json | jq -c --unbuffered .` as a long-lived stream that emits a snapshot
   at start and one per hotplug event. If the pipeline dies, the shell wrapper reports usbeehive's
   or jq's own error text, and the service retries.
2. While a port is charging, it polls `usbeehive --json` every `poll_interval_s` seconds. The
   watcher only listens to `usb`/`typec` uevents, so a PD renegotiation would otherwise go unnoticed.

The service publishes a pre-digested snapshot (`lib/model.luau`) on `noctalia.state("snapshot")`.
`widget.luau` and `panel.luau` only render it.

The wattage shown is usbeehive's `powerInMW`. That is the device's negotiated operating point
(UCSI voltage × current) when the kernel exposes it, otherwise the charger's active PD profile.
Either way it is a negotiated ceiling, not a metered reading.
Desktops without USB-C PD never show wattage. Their widget stays in device-count mode.

## Development

```sh
# Install as a local drop-in (outranks every source), then enable it once
ln -s ~/usbeehive-noctalia ~/.local/share/noctalia/plugins/usbeehive
noctalia msg plugins enable yangkx/usbeehive
noctalia plugins lint .
```

`.luau` edits hot-reload. Changes to `plugin.toml` are picked up on the next config reload.
Add the widget to a bar:

```toml
[widget.usb]
type = "yangkx/usbeehive:usb"
# then put "usb" in a [bar.<name>] start/center/end list
```

Useful IPC commands:

```sh
noctalia msg plugin yangkx/usbeehive:monitor all refresh
noctalia msg panel-toggle yangkx/usbeehive:devices
```

### Testing the wattage UI without USB-C PD

`dev/fixtures/charging` is a captured sysfs tree: a 100W USB-C PD charger, a 3A passive cable,
and the sink drawing 45W. Set the plugin's **usbeehive binary** setting to the absolute path of
`dev/usbeehive-fixture`, which runs usbeehive with `--sysfs-root` pointed at that tree.
