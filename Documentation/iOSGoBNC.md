# Using Rete on iOS with GoBNC

GoBNC is a single-user IRC bouncer. It stays connected to your IRC networks around the clock, keeps the
history, and hands it to whatever client you attach. Rete on iOS can attach to a GoBNC bouncer directly, so
your phone gets your networks, their history and your read positions without keeping an IRC connection of
its own.

This guide covers setting up the bouncer for Rete, adding it in the app, and what to expect once it is
there.

## How it works

The bouncer holds the IRC connections. Rete on the phone opens one secure WebSocket (`wss://`) to the
bouncer and uses it for every network on that bouncer. The connection is light: the bouncer sends the phone
the buffer list and the unread counts of every network you show on the device, and streams the lines of
only the buffer you are reading, plus the ones you have open. Everything else waits on the bouncer until
you look at it.

A phone can use a GoBNC bouncer and a ReteCloud account at the same time. Their networks sit together in
one buffer list, each network under its own collapsible header.

You can have more than one bouncer. Each is added separately.

## Setting up the bouncer

You need a GoBNC install with at least one network added. The first-time setup (installing GoBNC, setting
the bouncer password, adding networks) is in GoBNC's own README. This section covers what Rete needs on top
of that.

### Install the Rete connector

Rete on iOS connects through a separate program that ships with GoBNC, `gobnc-retecloud`. Building GoBNC
with `make build` produces it as `bin/gobnc-retecloud`; copy it to where you keep your binaries, for example
`/usr/local/bin/gobnc-retecloud`.

### Add it to `gobnc.json`

Add an entry to the `modules` list in `gobnc.json`:

```json
"modules": {
  "list": [
    {
      "name": "retecloud",
      "exec": "/usr/local/bin/gobnc-retecloud",
      "features": ["networks", "events", "history", "read_markers", "clients",
                   "send", "devices", "auth", "storage"],
      "config": {
        "listen_addr": "0.0.0.0:8443",
        "tls_cert": "/etc/gobnc/certs/server.crt",
        "tls_key": "/etc/gobnc/certs/server.key"
      }
    }
  ]
}
```

Keep the `features` list as shown. The keys in `config`:

| Key | Meaning |
|---|---|
| `listen_addr` | The address and port the phone connects to. The default, `127.0.0.1:8443`, accepts connections from the bouncer's own machine only, so set it (for example to `0.0.0.0:8443`) for a phone to reach it. |
| `tls_cert`, `tls_key` | The certificate and key the connector presents. Required. The `server.crt` and `server.key` that `make cert` creates for the bouncer work. |
| `max_connections` | How many phones and tabs may be connected at once. Default 16. A connection beyond that is refused. |
| `allowed_ips` | Optional. A list of addresses or networks (`"203.0.113.0/24"`) that may connect; empty means anyone. |

Then restart the bouncer's main process with `gobnc reload` so it starts the connector, and check with
`gobnc module list` that `retecloud` is running. Open the port (`8443` in the example) in your firewall.

### The address

The address you give the app is:

```
wss://bnc.example.org:8443/ws
```

That is the host you reach the bouncer at, the port from `listen_addr`, and the path `/ws`. It must start
with `wss://`; plain `ws://` is not accepted.

### Signing in

Rete can sign in two ways, with the same credentials you use for an IRC client:

- **Password.** The bouncer password you set with `gobnc auth set-password`. It is not a NickServ or SASL
  password.
- **Client certificate.** The phone presents a certificate, and the bouncer lets it in if the certificate's
  SHA-256 fingerprint is registered:

  ```bash
  gobnc auth add-fingerprint <sha256> [label]
  gobnc auth list-fingerprints
  gobnc auth delete-fingerprint <#N|sha256|prefix>
  ```

  The app shows the fingerprint of its certificate and the exact `add-fingerprint` command to run (see
  *Adding the bouncer*).

Five failed sign-ins from one address within a minute lock that address out for the rest of the minute. The
app shows when it will try again.

### Updating the connector

Install the new `gobnc-retecloud` binary at the path in `exec`, then run:

```bash
gobnc module reload retecloud
```

This is what applies the update. `gobnc reload` on its own restarts the bouncer's main process, but the
connector keeps running the old program. A reload hands the port to the new process; connected phones are
disconnected once and reconnect straight away, without showing an error. If the new program fails to start,
the old one keeps serving. `gobnc module reload retecloud` also applies a `config` change you made in
`gobnc.json` after `gobnc rehash`.

## Adding the bouncer in the app

On a fresh install, tap **Connect to a GoBNC Bouncer**. Otherwise, in the buffer list, open the **+** menu
and choose **Add GoBNC Bouncer**. Fill in:

- **Name.** Optional; it defaults to the host of the address.
- **Address.** The `wss://` address from above. The form tells you if it does not start with `wss://`.
- **Sign in with.** **Password** or **Client Certificate**.

With **Password**, type the bouncer password.

With **Client Certificate**, the phone needs a certificate. Tap **Generate Certificate** to make one, or
**Import .p12…** to use an existing PKCS#12 file (the app asks for its passphrase). The app then shows the
certificate's SHA-256 fingerprint, with a **Copy** button, and the command to register it:

```
gobnc auth add-fingerprint <fingerprint>
```

Run that on the bouncer. The phone uses the same certificate for every bouncer that signs in this way.

Tap **Connect**. The bouncer's networks appear in the buffer list.

### Trusting the bouncer's certificate

If the bouncer's certificate is signed by an authority your phone trusts, nothing more happens. A
self-signed certificate, such as the one `make cert` creates, is not, so the first time the app connects it
asks **Trust this server?** and shows the certificate's SHA-256 fingerprint. Compare it with the
fingerprint of the bouncer's `server.crt` (`openssl x509 -in server.crt -noout -fingerprint -sha256`)
and tap **Trust**. The app remembers the server's key and checks it on every later connection. Tap
**Cancel** and the bouncer stays off; the account row reads *Waiting for you to trust the server*.

If the bouncer later presents a different key, the app stops and asks **Server key changed**. Choose **Trust
New Key** only if you replaced the bouncer's certificate and key yourself. A renewed certificate that keeps
the same key does not ask again.

### Editing and removing the bouncer

To change a bouncer's name, address or sign-in, choose **Edit Bouncer**:

- in the network menu (long press a network's header in the buffer list),
- from the bouncer's status line above its networks, by swipe or long press, or
- in **Settings**, under **Accounts**, with **Edit**.

The form is the same as for adding. A saved password is kept when you leave the password field empty. The
app connects again when you change the address, the sign-in method or the password. The server key you
trusted is kept as long as the address names the same host and port.

To remove a bouncer, open **Settings**, then **Accounts**, and tap **Remove**. Its networks disappear from
the phone; the bouncer keeps them. Removing it also stops push notifications from that bouncer.

## Your networks

A GoBNC bouncer decides which networks exist. Add, change and delete networks on the bouncer with
`gobnc network add`, `gobnc network mod` and `gobnc network delete`; the app lists them under the bouncer
with the note *Networks are managed in the bouncer.* The phone cannot create or delete a network on a
bouncer, nor edit its server settings. A network a bouncer removes is gone from the phone's list the next
time the phone signs in.

What the phone does decide is what *this device* shows. Every network has a **Show on this device** switch
in Connections. Switch a network off and it leaves this phone's buffer list and stops streaming to it;
other devices, and the bouncer's connection to the network, are not affected. If the bouncer cannot be
reached when you flip the switch, the app says it could not change the network and shows what the bouncer
still has.

A network's long-press menu also offers **Connect** or **Disconnect** and **Reconnect**, which start,
stop and restart the bouncer's connection to that IRC network, and **Block list** on networks whose server
supports one.

## What differs from ReteCloud

- **No creating or deleting networks from the app.** Do that on the bouncer.
- **No Channel List.** Join channels with `/join` or the join button on a network's header.
- **Highlight rules are kept on the bouncer.** Choose **Highlight Rules…** in a network's long-press menu to
  edit them. Every device attached to that network uses the same rules, and the bouncer applies them when it
  counts highlights and decides what to notify you about.
- **A buffer's Highlight switch is shared too.** The **Highlight** switch in a buffer's long-press menu is
  stored on the bouncer and applies on every device. A buffer with it off counts no highlights and sends no
  notification. Lines in it still count as unread. The **Log** and **Activity** switches stay on the device.
- **Nick colours stay on the phone.** **Nick Colours…** in the network's menu edits rules that this
  device keeps for itself.
- **History comes from the bouncer.** Scrolling back reads the history the bouncer has stored.

## Push notifications

Rete on iOS can notify you of highlights and direct messages when the app is not open. For a GoBNC bouncer
this takes two things:

- You are signed in to ReteCloud in the app. The bouncer has no push service of its own, so it sends
  notifications through ReteCloud's relay. Without a ReteCloud sign-in the bouncer's networks work, but they
  do not push.
- Notifications are allowed for Rete in iOS Settings.

The bouncer needs to be able to reach ReteCloud over HTTPS. The text of each notification is encrypted on
the bouncer so only your phone can read it; ReteCloud and Apple relay it without seeing it.

A few rules to expect:

- Only networks you show on this device notify you.
- A buffer's **Highlight** switch and the network's highlight rules decide what counts.
- Nothing is sent while the app is open and connected to that bouncer.
- Tapping a notification opens the buffer at that line. Reading the buffer on another device removes the
  notification from this one.
- Signing out of ReteCloud, or removing the bouncer from the app, turns the bouncer's push off.

## Troubleshooting

**The app cannot connect, or says the connection was refused.**
Check that the connector is running (`gobnc module list`), that `listen_addr` is not left on the default
`127.0.0.1`, that the port is open in the firewall, and that the address ends in `/ws` and begins with
`wss://`. A bouncer that already has `max_connections` open refuses the next connection; raise the limit if
you use many devices.

**The app says the address must start with `wss://`.**
The connector only speaks TLS. Use `wss://`, not `ws://`.

**"The bouncer refused the password."**
That is the bouncer password from `gobnc auth set-password`, not a NickServ or SASL password. After five
failures the app waits before it tries again.

**"The bouncer does not recognise this device's certificate."**
Register the fingerprint the app shows with `gobnc auth add-fingerprint`. If you generated a new
certificate or imported a different one, the fingerprint changed; register the new one. The sign-in
methods you allow with `allow_password_auth` and `allow_cert_auth` in `gobnc.json` also apply here.

**A certificate warning, or "Server key changed".**
See *Trusting the bouncer's certificate*. If you replaced the bouncer's certificate, choose **Trust New
Key**. If you did not, do not.

**The connection dropped for a moment and came back.**
When you reload the connector, phones are disconnected once and reconnect immediately. Other drops
reconnect with a short, increasing delay, shown on the bouncer's row as *Reconnecting in N s*.

**"Could not change this network on the server. Try again when connected."**
The **Show on this device** switch asks the bouncer to change what it sends this device. Try again when the bouncer shows *Online*.

**Away, IRC operator and bot markers are missing in a channel's nick list.**
The bouncer learns who is away, an operator or a bot from the server's replies, and it does not know them
until it has seen a reply to a `WHO` for that channel. It asks for them in the background, a channel at a
time, so a busy bouncer can take a while to fill them in; running `/who #channel` in an IRC client attached
to the bouncer fills the channel in at once.

**Notifications do not arrive.**
Check that you are signed in to ReteCloud in the app, that notifications are allowed for Rete in iOS
Settings, that the network is shown on this device, and that the bouncer can reach ReteCloud over HTTPS.
