# Commands

Everything you type into the input line that starts with `/` is a command. This is the reference for all of
them: what each takes, what it does, and which apps have it.

The apps are written **Mac** (Rete for Mac), **iOS** (Rete for iOS), **reTerm** and **Linux** (Rete for Linux).

- Commands that talk to the IRC server (`/join`, `/msg`, `/kick` …) are **the same in every app**.
- A few commands belong to the app you are in. They are listed under *Commands of the app*, with the apps
  that have them. A command an app does not have answers `Unknown command: /name` and sends nothing.
- `/set` is its own document: [Settings](Settings.md) lists every key.
- Commands that scripts add are described in [Scripts](Scripts.md).

## At a glance

| Command group | Mac | iOS | reTerm | Linux |
|---|:-:|:-:|:-:|:-:|
| IRC commands (below) | yes | yes | yes | yes |
| `/buffer` (Log, Highlight, Activity) | yes | yes | yes | yes |
| `/script` | yes | says it is unavailable | yes | yes |
| `/set` | yes | no | yes | yes |
| `/help` | yes | no | yes | no |
| `/rowaudit` | yes | yes | yes | no |
| `/buffer` (switch, close, move, hide) | no | no | yes | no |
| `/network`, `/server`, `/exit`, `/mouse`, `/control`, `/log`, `/colour`, `/highlight`, `/fold`, `/raw` | no | no | yes | no |

## How a line is read

- The first word after the `/` is the command, in any case: `/JOIN` and `/join` are the same.
- A command that is not in this document, and that no script has added, is **sent to the server as typed**
  with the command word in capitals: `/knock #chan please` goes out as `KNOCK #chan please`. This is how
  you reach a server's own commands, and its services, such as `/ns identify …` or `/cs …`. A line that
  carries a password is shown masked in the raw buffer and is not kept in your input history.
- Two things are never sent: a command word that is not made of letters and digits (`/#chan`, `/@x`), and
  the name of a command that belongs to one app, typed in an app that does not have it. Both answer
  `Unknown command: /name`.
- A script can register a command of the same name as an IRC command below; the script's command then runs
  in its place.
- Tab completes command names and, for many commands, their arguments. As you type, the line shows the
  rest of the command's usage and a one-line description.
- Where a command says `[#channel]`, you can leave it out when you type the command in that channel's buffer.
- In a usage line, `<angle brackets>` are required, `[square brackets]` optional, and `a|b` means one of.

## IRC commands

All of these are in every app.

### Channels

| Command | What it does |
|---|---|
| `/join <#channel>[,<#channel>…] [key]` | Joins a channel, or several separated by commas. `key` is the channel's password. Also `/j`. |
| `/part [#channel] [reason]` | Leaves a channel (the current one if you name none), with the reason you type, or your part message (the `part_message` setting) when you type none. Also `/leave` and `/p`. |
| `/cycle [#channel]` | Leaves a channel and joins it again at once. |
| `/topic [#channel] [new topic]` | With no text, asks for the channel's topic. With text, sets it. Spacing inside the text is sent as you typed it. |
| `/topic [#channel] -clear` | Clears the topic. `-delete` does the same. |
| `/invite [#channel] <nick>` | Invites a nick to a channel. |
| `/list [#channel[,#channel]] [server]` | Lists channels and their topics; with no argument, all of them. |
| `/names [#channel[,#channel]] [server]` | Lists who is on a channel. |

```
/join #rete
/join #private secretkey
/topic #rete Release night - be kind
/topic -clear
```

`/topic` outside a channel needs the channel: `/topic #rete`.

### Messages

| Command | What it does |
|---|---|
| `/msg <target> <message>` | Sends a message to a nick or a channel. |
| `/notice <target> <message>` | Sends a notice. |
| `/me <action>` | Sends an action to the current buffer (`/me waves`). Not available in the server buffer. |
| `/smsg <target> <text>` | Sends a message like `/msg`, but Rete masks the text everywhere it would otherwise show or store it: the buffer, the raw buffer, logs and AI access. The server still receives it in full. A connection that cannot mask refuses and sends nothing. The text is not kept in your input history. |
| `/query <nick>` | Opens a conversation with a nick, without sending anything. A channel name is refused. |
| `/ctcp <target> <command> [text]` | Sends a CTCP request, for example `VERSION`, `TIME` or `USERINFO`. |
| `/ping <nick>` | Sends a CTCP PING to measure the round trip. |
| `/away [message]` | Marks you away with that message; with no message, marks you back. |
| `/wallops <message>` | Sends a message to everyone who has the wallops mode set, usually operators. |

### Nicks and information

| Command | What it does |
|---|---|
| `/nick <newnick>` | Changes your nick. |
| `/whois <nick> [nick…]` | Asks for information about one or more nicks. |
| `/whowas <nick> [count] [server]` | Looks up a nick that has left the network. |
| `/who [mask] [o]` | Lists users matching a mask. A trailing `o` limits it to operators. |
| `/userip <nick>` | Asks for a nick's IP address. Servers usually allow this to operators only. |
| `/silence [+mask\|-mask]` | The server-side ignore list: with no argument shows it, `+mask` adds, `-mask` removes. |

### Modes and channel operators

| Command | What it does |
|---|---|
| `/mode [target] <modes> [params]` | Views or changes modes. In a channel, `/mode +m` changes that channel; in the server buffer, `/mode +i` changes your own modes. Name the target to reach another: `/mode #rete +t`, `/mode mynick +i`. |
| `/op [#channel] <nick> [nick…]` | Gives channel operator status (`+o`). |
| `/deop [#channel] <nick> [nick…]` | Takes it away (`-o`). |
| `/voice [#channel] <nick> [nick…]` | Gives voice (`+v`), so a nick may speak in a moderated channel. |
| `/devoice [#channel] <nick> [nick…]` | Takes voice away (`-v`). |
| `/kick [#channel] <nick> [reason]` | Removes a nick from a channel. They may rejoin. |
| `/kickban <nick> [reason]` | Bans a nick and then kicks them. Only inside a channel buffer. |
| `/ban [#channel] <mask> [mask…]` | Adds a ban (`+b`). A mask can be a nick, a hostmask or an extban. |
| `/unban [#channel] <mask> [mask…]` | Removes a ban (`-b`). The mask must match the one that was set. |

```
/op alice bob
/kick #rete spammer flooding
/ban #rete *!*@bad.example.com
```

### The server and the network

Each of these is sent to the server as typed, and the server's answer appears in the server buffer. What a
server accepts, and what it replies, varies between servers.

| Command | What it asks |
|---|---|
| `/motd [server]` | The server's message of the day. |
| `/lusers [mask] [server]` | How many users and servers the network has. |
| `/map` | The network's server tree. Not every server has it. |
| `/links [server mask]` | The servers linked into the network. |
| `/stats [query] [server]` | Server statistics. The letters accepted vary by server. |
| `/admin [server]` | Who administers a server. |
| `/version [server]` | What software a server runs. |
| `/time [server]` | The server's local time. |

### Operators

| Command | What it does |
|---|---|
| `/oper <name> <password>` | Authenticates you as an IRC operator. The password is masked in everything Rete shows and is not kept in your input history. |
| `/kill <nick> <reason>` | Disconnects a nick from the network. |
| `/connect <server> <port> [remote server]` | Links one server to another. This is the operator command, not a way to connect Rete to a network. |
| `/squit <server> <comment>` | Breaks a server link. |
| `/rehash` | Makes the server re-read its configuration. |
| `/restart` | Restarts the server. |
| `/die` | Shuts the server down. |

### Connection

| Command | What it does |
|---|---|
| `/quit [message]` | Leaves IRC, with that quit message or your `quit_message` setting. On a ReteCloud session, takes the session off IRC while this app stays linked. It does not close the app. |
| `/disconnect [message]` | Closes this app's link and does not reconnect by itself. On a ReteCloud session, the session stays on IRC. |
| `/reconnect [message]` | Disconnects and connects again. |

### Raw lines, ZNC and bouncers

| Command | What it does |
|---|---|
| `/quote <raw line>` | Sends the line to the server exactly as typed, with no interpretation. |
| `/znc <params>` | Sends a `ZNC` command to the server: `/znc *status help` goes out as `ZNC *status help`. |
| `/bnc <params>` | Sends a `BNC` command to the server in the same way. |

### Diagnostics

`/perf [on|off|status|reset|dump]` switches Rete's performance recording and prints or saves what it
recorded. It is for troubleshooting with the developers and has no effect on chatting.

## Commands of the app

### `/buffer`

A buffer is a channel, a conversation or a server window. The part that every app has:

| Form | What it does |
|---|---|
| `/buffer` | Says the current buffer's Log, Highlight and Activity settings. |
| `/buffer log [on\|off] [name]` | Turns logging on or off for a buffer; with neither, says which. |
| `/buffer highlight [on\|off] [name]` | Turns highlights on or off for a buffer. |
| `/buffer activity [on\|off] [name]` | Turns activity on or off. With activity off, the buffer's unread count is hidden; a highlight still shows. |

With no `name`, the command acts on the buffer you typed it in. These are the same switches as in a buffer's
menu, saved per network. `logging` and `highlights` are accepted for `log` and `highlight`.

**Mac, iOS, reTerm, Linux** have the forms above. **reTerm** adds the rest:

| Form | What it does |
|---|---|
| `/buffer <name>` | Switches to a channel, conversation or other buffer by name. |
| `/buffer <N>` | Switches by the number shown beside the buffer in the list. |
| `/buffer close [name]` | Closes the current channel or conversation, or the one named. A channel you are on is left first. |
| `/buffer move up\|down\|top\|bottom\|<N> [name]` | Moves a channel or conversation within its own list. `<N>` is a position, from 1. The order is saved per network. |
| `/buffer hide [name]`, `/buffer show <name>` | Hides a channel or conversation from the list without leaving it, and brings it back. |

One of the words above wins over a buffer of that name; use `/buffer <N>` to reach one.

### `/set`

**Mac, reTerm, Linux.** Reads and changes settings: `/set` lists them, `/set <key> <value>` changes one.
Every key and value is in [Settings](Settings.md).

### `/help`

**Mac, reTerm.**

| Form | What it does |
|---|---|
| `/help` | An introduction and the list of help topics. |
| `/help commands` | Every IRC command with a one-line description. |
| `/help <command>` | One command's usage and description, for example `/help kick`. |
| `/help <topic>` | A topic of the app, such as `/help buffer`. |

reTerm's topics are `network`, `server`, `exit` (also `quit`), `script`, `set`, `rowaudit`, `control`,
`mouse` (also `select`, `copy`), `colour`, `highlight`, `keys` (also `keyboard`), `fold`, `log` and
`buffer`. Mac's are `script`, `set` and `rowaudit`.

### `/script`

**Mac, reTerm, Linux.** Manages Tcl script bundles from the input line: list, inspect, fetch from the
catalogue, load, unload, rehash, debug, and start a new bundle. **iOS** has no scripting and says so.
The subcommands and their answers are in [Scripts](Scripts.md#the-script-command).

### `/rowaudit`

**Mac, iOS, reTerm.** A check that every message the app received is on screen. Switch recording on with
`/rowaudit on`, then `/rowaudit` (or `/rowaudit run [settle-ms] [max-wait-s]`) waits for the message stream
to go quiet and prints a report naming any message that went missing. `/rowaudit status` shows what has been
seen, `/rowaudit reset` starts the window again, and `/rowaudit off` stops recording. It is off by default
and costs nothing while off.

## reTerm

These commands exist in **reTerm only**. They work in any buffer, with or without a connected network,
unless a command says otherwise.

### `/network` and `/server`

reTerm has no connection window, so networks are added and edited here.

```
/network add libera irc.libera.chat 6697 --tls true
/network add mynet ws://irc.example.com:8097
/server libera
```

| Command | What it does |
|---|---|
| `/network list` | Lists the networks. |
| `/network add <name> <host> <port> [flags]` | Adds a network. Use the positional name, host and port (or a `ws://` or `wss://` address in place of host and port) and not `--name`, `--host` or `--port`, which are for `mod`. |
| `/network mod <name> [flags]` | Changes a network. |
| `/network del <name>` | Removes a network. |
| `/network help` | Prints the flags below in the app. |
| `/server` | Lists the configured networks. |
| `/server <network>` | Connects to a network. The network can be given by its name, its server, its ReteCloud session id, its identifier or a unique prefix. |

Flags for `/network add` and `/network mod`. A boolean takes `true`, `false`, `yes`, `no`, `on`, `off`, `1` or
`0`. Where a flag shows `|none`, passing `none` clears the field.

| Flag | Meaning |
|---|---|
| `--retecloud <bool>` | A ReteCloud-backed session rather than direct IRC. |
| `-tls`, `--tls <bool>` | Use TLS. `-tls` alone turns it on. |
| `--accept-self-signed <bool>` | Accept a self-signed certificate. |
| `--display-name <name>`, `--name <name>` | The name shown (`mod`). |
| `--server <host>`, `--host <host>` | The server address (`mod`). |
| `--port <1-65535>` | The port (`mod`). |
| `--nick <nick>`, `--alt-nick <nick\|none>` | Your nick and a fallback. |
| `--username <user>`, `--real-name <text>` | Your username and real name. |
| `-password <pass>`, `--password <pass\|none>` | The server password. |
| `--user-modes <modes\|none>` | User modes to set on connect. |
| `--auto <bool>`, `--auto-connect <bool>` | Connect when reTerm starts. |
| `--bouncer <bool>`, `--bouncer-integration <bool>` | Treat the server as a bouncer. |
| `--sasl-disable <bool>` | Switch SASL off. |
| `--sasl-mechanism <plain\|external\|scram-sha-256\|none>` | The SASL mechanism; `none` or `off` disables SASL. |
| `--sasl-user <user\|none>`, `--sasl-password <pass\|none>` | SASL credentials. |

Lines that carry a password are not kept in your input history.

### `/exit`

`/exit [message]` leaves reTerm. It sends QUIT to every connected network (with the message, or your
`quit_message` setting), waits for the goodbyes to go out, saves the logs and exits. A second `/exit` while
that is happening leaves at once. `/quit` is the IRC command: it disconnects the current network and
leaves reTerm running. Ctrl+C does nothing on purpose; closing the terminal leaves the same way `/exit` does.

### `/mouse`

`/mouse on|off|toggle` is a short form of `/set mouse on|off|toggle`. By default reTerm takes the mouse:
click a buffer in the list, a link to open it, a channel to join it, a nick to put it in the input line, or a
folded line to open it; the wheel scrolls the pane under the pointer. Hold Shift as you drag to select text
and copy it (Option in iTerm2, Fn in Terminal.app). With the mouse off, your terminal handles selection and
you scroll with PageUp and PageDown.

### `/colour`

Colours nicks in the nick list and the transcript, on the network of the current buffer. The first rule that
matches a user gives the colour. These are the rules Rete for Mac edits as Coloured nicks.

| Command | What it does |
|---|---|
| `/colour add <kind> [pattern] <colour>` | Adds a rule. |
| `/colour list` | Lists the rules, numbered, in the order they are tried. |
| `/colour del <n>` | Removes rule `n`. |

Kinds: `me`, `oper`, `secure`, `bot`, and `nick <nick>`, `account <account>` and `hostmask <mask>`, which
take a pattern. A colour is `#RRGGBB` or one of `red`, `orange`, `yellow`, `gold`, `lime`, `green`, `olive`,
`teal`, `cyan`, `blue`, `navy`, `purple`, `magenta`, `pink`, `brown`, `maroon`, `white`, `silver`, `gray`
and `black`.

```
/colour add nick alice orange
/colour add oper #FF8800
/colour del 2
```

### `/highlight`

Chooses which messages are highlighted, on the network of the current buffer. A message that matches any
enabled rule is highlighted. With no rules, messages that contain your nick are. These are the rules Rete for
Mac edits as Highlight Rules.

| Command | What it does |
|---|---|
| `/highlight add <kind> [text]` | Adds a rule. |
| `/highlight list` | Lists the rules, numbered. |
| `/highlight del <n>` | Removes rule `n`. |

| Kind | Highlights a message when |
|---|---|
| `mynick` | it contains your nick |
| `text <text>` | it contains the text |
| `sender <nick>`, `notsender <nick>` | it is sent by, or not sent by, that nick |
| `loggedin`, `notloggedin` | the sender is, or is not, logged in to an account |
| `common`, `notcommon` | the sender is, or is not, on a channel with you |
| `opped`, `voiced` | the sender is opped, or voiced, on a channel with you |
| `channel <#channel>`, `notchannel <#channel>` | it is, or is not, in that channel |

```
/highlight add text release
/highlight add sender alice
```

### `/fold`

Join, part and quit lines can be folded into one line (`/set collapse_activity on`); a channel limit change
(`+l`) among them folds in too, and the folded line shows the new limit. `/fold` opens the most
recent folded line in the current channel, or closes it if it is open. `/fold all` opens every fold in the
channel, or closes them all when every one is already open. It works in a channel only.

### `/log`

`/log export [-codes] <buffer> <path>` writes the log of a buffer to a file, oldest first, one message per
line. Run it from one of the network's buffers: the log exported is that network's. A path ending in
`.jsonl` is written as JSON Lines; any other is plain text. An existing file is replaced, and a relative
path is relative to the directory reTerm was started in. The text has its IRC formatting codes stripped,
unless you give `-codes`.

```
/log export #rete ~/rete.txt
/log export -codes #rete ~/rete.jsonl
```

### `/control`

AI access, described in [Control](Control.md). `/control status` says whether it is on, its debug tools and
sending. `/control token` shows the AI access token, and `/control token regenerate` makes a new one.
`/control log` prints the AI access log.

### `/raw`

`/raw on` shows the raw buffer, where every line to and from the server is listed, and `/raw off` hides it.

## Keyboard (reTerm)

`/help keys` prints this list in the app.

| Key | Does |
|---|---|
| Left, Right, Ctrl+F | Move back or forward a character. |
| Home, End, Ctrl+A, Ctrl+E | Move to the start or end of the line. |
| Backspace, Delete, Ctrl+D | Delete the character before or under the cursor. |
| Ctrl+U, Ctrl+K | Delete to the start or end of the line. |
| Ctrl+W | Delete the word before the cursor. |
| Esc | Clear the line. |
| Ctrl+L | Redraw the screen. |
| Up, Down | Earlier and later lines you sent. |
| Tab | Complete what you are typing. |
| Shift+Up, Shift+Down | The previous or next buffer. |
| Alt+Up, Alt+Down | Move the current buffer up or down the list. |
| Alt+1 to Alt+9 | The buffer with that number. |
| PageUp, PageDown | Scroll the transcript; PageUp at the top loads older messages. |

Formatting codes, typed into the line:

| Key | Code |
|---|---|
| Ctrl+B | Bold |
| Ctrl+_ | Underline |
| Ctrl+R, Ctrl+V | Reverse |
| Ctrl+] | Italics |
| Ctrl+^ | Strikethrough |
| Ctrl+Q | Monospace |
| Ctrl+O | Plain text |

Other control characters are not typed into the line.
