# Nuticast for Codex

Let Codex get an iPhone on a different network ready for Xcode. This plugin
connects Codex (CLI, IDE extension and desktop app) to
[Nuticast](https://www.nuticast.com), a Mac app that makes an iPhone on another
Wi-Fi show up in Xcode as a wireless debug device, over your own Tailscale,
ZeroTier or Netbird network.

Ask Codex to "run the tests on my iPhone" and it can list your saved devices,
start the bridge, wait until Xcode sees the phone, and build against the device
with the right `xcodebuild` destination.

## Requirements

- A Mac with the [Nuticast app](https://www.nuticast.com/download) installed in
  `/Applications` (the plugin starts the `nuticast` command inside it), version
  1.0.17 or later (Codex cannot open the Nuticast server when a session starts on
  1.0.16 or earlier). With Homebrew:
  `brew install --cask webwarrior06/tap/nuticast`
- A Nuticast license (the bridge needs one; reading devices does not)
- An iPhone saved in Nuticast and a mesh VPN (Tailscale, ZeroTier or Netbird)
  on both devices
- Codex on that Mac. This is a local server, so it does not run in Codex cloud
  tasks or on mobile.

## Install

Add this repository as a plugin marketplace, then install the plugin:

```
codex plugin marketplace add webwarrior06/nuticast-codex-plugin
codex plugin add nuticast@nuticast
```

You can also open `/plugins` inside Codex and install Nuticast from the
`nuticast` marketplace. Start a new session afterwards so Codex picks up the
skill and the server.

To use only the tools, without the skill:

```
codex mcp add nuticast -- /Applications/Nuticast.app/Contents/Helpers/nuticast mcp
```

## Set up the license once

The first time the license is read, macOS may ask for access to the Keychain
item. In your own Terminal run:

```
/Applications/Nuticast.app/Contents/Helpers/nuticast license activate
```

Type the key at the hidden prompt, and choose **Always Allow** if macOS asks.
On Nuticast older than 1.0.17 that command needs the key as an argument. Copy
the key and run this instead; it reads the key from stdin and keeps it out of
your shell history:

```
pbpaste | /Applications/Nuticast.app/Contents/Helpers/nuticast license activate -
```

Codex never sees the key and cannot activate or deactivate a license. If the
Keychain prompt is never answered, the license tools time out and Codex will
tell you to run `nuticast license` in Terminal.

## What it adds

A skill that tells Codex how to use the tools in the right order, and a local
MCP server (`nuticast mcp`, started by Codex over stdio) with these tools:

| Tool | What it does | Changes anything |
| --- | --- | --- |
| `nuticast_list_devices` | Lists saved iPhones and how each is reached | No |
| `nuticast_list_mesh_peers` | Asks Tailscale and Netbird for the devices on your network | No |
| `nuticast_scan_lan` | Finds iPhones advertising wireless debugging on this Wi-Fi | No |
| `nuticast_list_xcode_devices` | Lists the devices Xcode can see, with UDIDs | No |
| `nuticast_bridge_status` | Shows which devices are bridged by the command line | No |
| `nuticast_wait_for_bridge` | Waits for a bridge to come up | No |
| `nuticast_get_logs` | Reads the last lines of the Nuticast log (at most 200) | No |
| `nuticast_license_status` | Shows whether this Mac is licensed | No |
| `nuticast_add_device` | Saves an iPhone with its mesh VPN device (the iPhone must be on this Mac's Wi-Fi or plugged in) | Saves a device |
| `nuticast_diagnose` | Probes a device over the mesh; saves a moved port | Saves a port |
| `nuticast_start_bridge` | Starts a bridge for a saved device | Starts a process |
| `nuticast_ensure_device_ready` | Starts the bridge if needed and waits for Xcode | Starts a process |
| `nuticast_stop_bridge` | Stops a command-line bridge | Ends a debug session |

All 13 tools need Nuticast 1.0.17 or later. Earlier versions have no
`nuticast_add_device`; there, save the iPhone in the Nuticast app instead.

It cannot delete devices, activate or deactivate a license, or install apps.

## Approvals and timeouts

Every tool carries MCP annotations. Read-only tools are marked `readOnlyHint`,
`nuticast_stop_bridge` is marked `destructiveHint`, and the tools that start a
process or save a setting are marked as not read-only. To have Codex ask before
anything that is not read-only, add this to `~/.codex/config.toml`:

```toml
[plugins."nuticast@nuticast".mcp_servers.nuticast]
default_tools_approval_mode = "writes"
```

If you added the server with `codex mcp add` instead, put the same key under
`[mcp_servers.nuticast]`.

Codex stops a tool call after 60 seconds by default and a server start after 10
seconds. A cold `nuticast_ensure_device_ready` can take up to about three
minutes, so the plugin raises the limits to 30 seconds (start) and 300 seconds
(tool call). If you installed with `codex mcp add`, set them yourself:

```toml
[mcp_servers.nuticast]
command = "/Applications/Nuticast.app/Contents/Helpers/nuticast"
args = ["mcp"]
startup_timeout_sec = 30
tool_timeout_sec = 300
```

## What it runs and what it sends

The plugin contains no program of its own. It only starts the signed,
notarized `nuticast` command that ships inside Nuticast.app, and talks to it
over standard input and output. While tools run, that command may run
`tailscale`, `netbird`, `zerotier-cli`, `dns-sd` and `devicectl` on your Mac, as
the Nuticast app does.

The only network request Nuticast makes because of this plugin is the license
check against `nuticast.com`, when you start a bridge. It sends your license key,
the activation's instance ID and a hardware identifier of your Mac.

Tool results (device names, addresses, peer names, log lines) are returned to
Codex and become part of your conversation. Log lines and peer names can
contain mesh IP addresses and host names.

## Privacy Policy

Nuticast collects no usage data and has no account. The full policy, including
what the MCP tools return, what is stored and for how long, third parties, and
how to contact us, is at <https://www.nuticast.com/privacy>.

- Data collection: none by us from this plugin.
- Usage and storage: device profiles and the log stay on your Mac.
- Third-party sharing: tool output goes only to the MCP client you use; the
  license check goes only to our license service.
- Retention: we keep nothing from MCP calls. Your client keeps what it keeps.
- Contact: hello@nuticast.com

## License

The files in this repository are released under the MIT License. The Nuticast
app and the `nuticast` command are proprietary and licensed separately.
