---
layout: doc
title: Scripts Tutorial
permalink: /docs/scripts/
---
This document describes Rete's Tcl scripting API: how to bind to events, how return
values influence client behaviour, what globals are available, and which helper
commands you can call from Tcl.

### Command Namespace

All Rete-specific Tcl commands are registered in the `::rete::` namespace to avoid conflicts with standard Tcl commands and libraries. Commands can be called using the full namespace prefix (e.g., `::rete::join`) or by importing the namespace:

```tcl
namespace import ::rete::*
```

### Event binding

- **Bind a handler**:

```tcl
::rete::bind EVENT handlerProc
```

- **Unbind a handler**:

```tcl
::rete::unbind EVENT handlerProc
```

- Multiple handlers may be bound to the same `EVENT`; they run in bind order.
- Handler procs are called as:

```tcl
proc handlerProc {arg1 arg2 ...} {
    # return 0 to let Rete continue normal processing
    # return 1 to stop further processing of this event
}
```

#### Return convention

- **Return `0`** → Rete continues processing (other Tcl handlers, then its own logic).
- **Return `1`** → Rete stops processing this event in Swift; no further handlers
  are run and the built‑in behaviour is skipped.
- Any non‑integer or missing return value is treated like `0` (continue).

#### Event arguments

Each event passes a fixed list of string arguments to your handler.
Timestamps are passed as ISO‑8601 strings when available, otherwise `""`.
Optional values (like `account`, `reason`) are passed as `""` when absent.

**RAWIN**
```tcl
proc my_rawin {line} {
    # line: raw line from server before any parsing
    return 0
}
```

**RAW_OUT** (outgoing raw line)
```tcl
proc my_raw_out {line} {
    # line: raw line about to be sent to the server
    # return 0 to allow send, return 1 to block send
    return 0
}
```

**REGISTERED**
```tcl
proc my_registered {} {
    # No arguments
    # Fires on 001 RPL_WELCOME (protocol registration). Prefer ENDOFMOTD for
    # post-connect work such as perform/auto-commands; bouncers may emit 001 early.
    return 0
}
```

**ENDOFMOTD**
```tcl
proc my_endomotd {code} {
    # code: "376" (RPL_ENDOFMOTD) or "422" (ERR_NOMOTD)
    #
    # Fires once the server has finished the MOTD phase — either End of MOTD or
    # No MOTD. Despite the name, both numerics use this event: they are the
    # classic client cue that the connection banner is done and JOIN / perform
    # commands are safe to send. Prefer this over REGISTERED (001) or filtering RPL.
    return 0
}
```

**SILENCE**
```tcl
proc my_silence {action hostmask} {
    # action: "add" or "rem"
    # hostmask: the hostmask being added/removed
    return 0
}
```

**ERROR**
```tcl
proc my_error {text} {
    # text: server error message
    return 0
}
```

**RPL** (numeric replies)
```tcl
proc my_rpl {code text buffer serverTime} {
    # code: numeric code as string (e.g., "001", "005"), or ""
    # text: reply text
    # buffer: target buffer name (channel/DM) if applicable, or ""
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**SERVERNOTICE**
```tcl
proc my_servernotice {from text channel serverTime} {
    # from: sender prefix or ""
    # text: notice text
    # channel: associated channel if applicable, or ""
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**SASL_AUTH**
```tcl
proc my_sasl_handler {username password mechanism} {
    # username  - Current SASL username (may be empty, may contain spaces)
    # password  - Current SASL password (may be empty, may contain spaces)
    # mechanism - SASL mechanism name (e.g. "plain", "external")
    #
    # Return {} or "" to indicate "no override" and let Rete use the configured
    # credentials, or return a two-element list {newUsername newPassword} to
    # override what is sent to the server.

    # Example: override username and password
    set newUser "my sasl user"
    set newPass "my secret pass"
    return [list $newUser $newPass]
}

bind SASL_AUTH my_sasl_handler
```

**JOIN**
```tcl
proc my_join {channel nick user host account realname serverTime} {
    # channel: channel name
    # nick: nickname of person who joined
    # user: ident/username
    # host: hostname
    # account: services account name or ""
    # realname: realname/GECOS
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**PART**
```tcl
proc my_part {channel nick reason serverTime} {
    # channel: channel name
    # nick: nickname of person who left
    # reason: part reason or ""
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**QUIT**
```tcl
proc my_quit {nick message serverTime} {
    # nick: nickname of person who quit
    # message: quit message or ""
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**CHANMSG** (channel PRIVMSG)
```tcl
proc my_chanmsg {from channel text serverTime} {
    # from: sender nickname
    # channel: channel name
    # text: message text
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**DIRECTMSG** (direct PRIVMSG)
```tcl
proc my_directmsg {from target text serverTime} {
    # from: sender nickname
    # target: recipient (your nick or theirs)
    # text: message text
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**CHANNOTICE** (channel NOTICE)
```tcl
proc my_channotice {from channel target text serverTime} {
    # from: sender nickname or server prefix
    # channel: channel name
    # target: target nickname or ""
    # text: notice text
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**DIRECTNOTICE** (direct NOTICE)
```tcl
proc my_directnotice {from target text serverTime} {
    # from: sender nickname or server prefix
    # target: recipient nickname
    # text: notice text
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**NICK** (nick change)
```tcl
proc my_nick {oldNick newNick serverTime} {
    # oldNick: previous nickname
    # newNick: new nickname
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**TOPIC** (topic info/update)
```tcl
proc my_topic {channel topic who serverTime} {
    # channel: channel name
    # topic: topic text or ""
    # who: nickname who set topic, or "" for topic info (RPL 332/333)
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**MODE** (channel mode change)
```tcl
proc my_mode {channel setter modeString params serverTime} {
    # channel: channel name
    # setter: nickname who set the mode
    # modeString: mode string (e.g., "+o", "+nt-lk"), or ""
    # params: space-separated mode parameters, or ""
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**AWAY** (away status change)
```tcl
proc my_away {nick isAway message serverTime} {
    # nick: nickname
    # isAway: "1" if now away, "0" if back
    # message: away message or ""
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**ACCOUNT** (account change)
```tcl
proc my_account {nick account serverTime} {
    # nick: nickname
    # account: services account name, or "" if logged out
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**CHGHOST** (user/host change)
```tcl
proc my_chghost {nick user host serverTime} {
    # nick: nickname
    # user: new ident/username
    # host: new hostname
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**OPER** (oper status change)
```tcl
proc my_oper {nick isOper} {
    # nick: nickname
    # isOper: "1" if now oper, "0" if no longer oper
    # NOTE: No serverTime for OPER events
    return 0
}
```

**INVITE**
```tcl
proc my_invite {inviter target channel serverTime} {
    # inviter: nickname who sent invite
    # target: nickname who was invited (may be you)
    # channel: channel name
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**KICK**
```tcl
proc my_kick {channel kicker victim reason serverTime} {
    # channel: channel name
    # kicker: nickname who kicked, or ""
    # victim: nickname who was kicked
    # reason: kick reason or ""
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**WALLOPS**
```tcl
proc my_wallops {from text serverTime} {
    # from: sender nickname or server prefix
    # text: wallops message text
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**ACTION** (ACTION, `/me`)
```tcl
proc my_action {from target text serverTime} {
    # from: sender nickname
    # target: channel name or nickname (where action was sent)
    # text: action text (without the `/me` prefix)
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**CTCPREQ** (CTCP request)
```tcl
proc my_ctcpreq {from target command params serverTime} {
    # from: sender nickname
    # target: recipient (channel or nickname)
    # command: CTCP command name (uppercased, e.g., "VERSION", "PING")
    # params: command parameters or ""
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

**CTCPRPL** (CTCP reply)
```tcl
proc my_ctcprpl {from target command params serverTime} {
    # from: sender nickname
    # target: recipient (channel or nickname)
    # command: CTCP command name (uppercased, e.g., "VERSION", "PING")
    # params: reply parameters or ""
    # serverTime: ISO-8601 timestamp or ""
    return 0
}
```

### Globals

**Client target**:

- `rete-target` – which client app is running: `rete` (Rete GUI app) or `reterm` (reTerm terminal client). The value is fixed for the lifetime of the Tcl interpreter.

**Identity variables**:

- `mynick`   – your current nickname
- `myuser`   – your user/ident
- `myhost`   – your host
- `myaccount` – your services account name (or `""` if not authenticated)

**Server variables**:

- `server` – IRC server name from **004 RPL_MYINFO** (falls back to the 001 prefix until 004 arrives). Prefer this over `serveraddress` when matching networks (e.g. behind ZNC).
- `serveraddress` – configured connect host and port (`host:port`), i.e. the bouncer or direct endpoint you connected to
- `serverdaemon` – the detected IRC daemon type (e.g., `"ircu"`, `"ircd"`, `"ratbox"`, `"hybrid"`, `"bahamut"`, `"unrealircd"`, `"inspircd"`, `"solanum"`, `"snircd"`, or `"unknown"`)
- `server-online` – unixtime when connected to current server, or `"0"` if disconnected

**Version variables**:

- `version` – current Rete version string (e.g., `"1.0+pl100"`)
- `numversion` – numeric version in `M.NN.RR.PP` format (e.g., `"1.00.00.00"`)
- `uptime` – unixtime when the store/script engine was created

Access them from Tcl as normal variables. **Inside procedures**, you must either use `global` statements or the `::` namespace prefix:

```tcl
# At the top level of a script, use directly:
::rete::debug "I am $mynick!$myuser@$myhost (account=$myaccount)"

# Inside a procedure, use global:
proc my_handler {args} {
    global rete-target mynick myuser myhost myaccount server serveraddress
    ::rete::debug "I am $mynick!$myuser@$myhost"
}

# Or use the :: prefix to explicitly reference the global namespace:
proc my_handler {args} {
    ::rete::debug "Running on $::rete-target"
    ::rete::debug "I am $::mynick!$::myuser@$::myhost"
    ::rete::debug "Connected to $::server at $::serveraddress"
}
```

### Helper commands (getters)

- **Channel topic**

```tcl
::rete::topic #channel
```

Returns the current topic for `#channel` as a string, or `""` if none is known.

- **Channel modes**

```tcl
::rete::getchanmode #channel
```

Returns a human‑readable mode string for `#channel` if available, or `""` if
no cached information is available.

- **ISUPPORT**

```tcl
::rete::isupport_get KEY     ;# returns string or ""
::rete::isupport_isset KEY   ;# returns 1 if the key is present, 0 otherwise
```

These map directly onto the server's ISUPPORT tokens as seen in RPL 005.

- **IRC string comparison**

```tcl
::rete::rfcequal string1 string2
```

Checks if two strings are equal using IRC case-insensitive comparison (RFC 1459
matching). This uses the same comparison logic as Rete's internal `IRCCompare`
function, which respects the server's `CASEMAPPING` ISUPPORT token.

Returns `1` if the strings are equal (case-insensitive, with RFC 1459 character
equivalences: `{}|~` == `[]\^`), `0` otherwise.

Example:

```tcl
# These all return 1 (equal)
::rete::rfcequal "Nick" "nick"
::rete::rfcequal "Test" "test"
::rete::rfcequal "User[Name" "user{name"  ;# [ == { under RFC 1459
::rete::rfcequal "Chan^el" "chan~el"       ;# ^ == ~ under RFC 1459
```

- **Ban hostmask (`maskhost`)**

```tcl
::rete::maskhost nick!user@host ?masktype?
```

Returns a masked IRC hostmask string suitable for bans, using the same rules as
common Eggdrop-style `maskhost`. `masktype` is an integer from `0` to `39`.
If you omit it, Rete uses the **Ban mask type** preference (**Chat & Behaviour →
IRC** in the app), which defaults to `3` when unset.
Invalid `nick!user@host` or an out-of-range type makes the
command return a Tcl error.

**Types 0–9** (default wildcards on nick/user/host as shown):

| Type | Result pattern |
|------|----------------|
| 0 | `*!user@host` |
| 1 | `*!*user@host` (leading `~` on ident is stripped when a `*` prefix is added) |
| 2 | `*!*@host` |
| 3 | `*!*user@*.host` (see below) |
| 4 | `*!*@*.host` |
| 5 | `nick!user@host` |
| 6 | `nick!*user@host` |
| 7 | `nick!*@host` |
| 8 | `nick!*user@*.host` |
| 9 | `nick!*@*.host` |

For types **3, 4, 8,** and **9**, the host is widened when possible: IPv4 addresses
become `a.b.c.*`; hostnames with at least three dot-separated labels become
`*.last.two.labels`; **IPv6 literals** are normalized to a **`/64` CIDR prefix**
(e.g. `2001:db8:1:2:3:4:5:6` → `2001:db8:1:2::/64`), including forms with
brackets (`[...]`) or a zone id (`%…`), which are stripped before parsing.
Strings that look like hostnames but are not valid IPv6 are unchanged. Two-label
domains (e.g. `example.com`) are left unchanged.

**Types 10–19** — same structure as types **0–9**, but on hostnames that are not
handled by the IPv4 / multi-label / IPv6 widening above, **each digit** in the host
is replaced with `?`. When the widening rule applies (IPv4, `*.domain`, etc.),
behaviour matches the corresponding type **0–9**.

**Types 20–29** — same as **10–19**, but each run of digits in the host is replaced
with a single `*`.

**Types 30–39** — same nick and user wildcards as the corresponding type among
**0–9** (e.g. type 32 matches type 2), but the host is always `*`.

Example:

```tcl
::rete::maskhost "MrIron!~id@127.0.0.1"     ;# -> per Ban mask type pref (e.g. *!*id@127.0.0.* when pref is 3)
::rete::maskhost "MrIron!id@pretty.lame.tld" 3
# -> *!*id@*.lame.tld
::rete::maskhost "nick!user@2001:db8:1:2:3:4:5:6" 3
# -> *!*user@2001:db8:1:2::/64
```

- **Theme settings**

```tcl
::rete::theme_get <option>
```

Returns the value of a theme setting from the current theme. This allows scripts
to access theme configuration values such as colors, badge icons, and theme
information.

**Options:**

- **Badge icons** (returns SF Symbol names): `operatorBadge`, `awayBadge`,
  `secureBadge`, `botBadge`, `blockedBadge`, `joinBadge`, `partBadge`,
  `quitBadge`, `actionBadge`, `noticeBadge`, `infoBadge`, `topicBadge`

- **UI colors** (returns hex color strings): `background`, `foreground`, `accent`,
  `divider`, `topicBarBackground`, `topicBarForeground`, `badgeBackground`,
  `badgeForeground`, `sidebarSelectionBackground`, `sidebarSelectionForeground`,
  `highlightRowBackground`, `expandedGroupRowBackground`

- **Badge colors** (returns hex color strings): `operatorBadgeColor`,
  `awayBadgeColor`, `secureBadgeColor`, `botBadgeColor`, `blockedBadgeColor`

- **Theme information**: `theme` or `themename` (current theme name),
  `scriptEditorHighlightrTheme` (Highlightr theme name for script editor)

Returns a string value, or an empty string if the option is not found. Colors are
returned in hex format (e.g., `"#FF0000"`). The command automatically uses the
appropriate light/dark palette based on the current appearance mode.

Example:

```tcl
# Get current theme name
set themeName [::rete::theme_get theme]
::rete::debug "Current theme: $themeName"

# Get accent color
set accentColor [::rete::theme_get accent]
::rete::debug "Accent color: $accentColor"

# Get bot badge icon
set botBadge [::rete::theme_get botBadge]
::rete::debug "Bot badge: $botBadge"

# Respond with theme information
proc on_chanmsg_info {from channel text serverTime} {
    if {$text eq "!theme"} {
        set themeName [::rete::theme_get theme]
        set highlightTheme [::rete::theme_get scriptEditorHighlightrTheme]
        ::rete::msg $channel "Theme: $themeName, Highlight theme: $highlightTheme"
    }
    return 0
}
::rete::bind CHANMSG on_chanmsg_info
```

For complete documentation, see the "Accessing Theme Settings from TCL Scripts"
section in [Themes.md](Themes.md).

- **Channel status**

```tcl
::rete::isop nick #channel
::rete::ishalfop nick #channel
::rete::isvoice nick #channel
```

Each returns `1` if the nick has the corresponding privilege on the channel,
`0` otherwise. `nick` may be `""` to mean "me".

- **User status**

```tcl
::rete::isaway nick
::rete::isbot nick
::rete::isoper nick
::rete::issecure nick
::rete::getaccount nick
```

These query Rete's cached `Person` entry for `nick` and return:

- `isaway` / `isbot` / `isoper` / `issecure`: `1` or `0`.
- `getaccount`: account string or `""` if not known.

`issecure` checks if the user has a secure (TLS/SSL) connection to the server.

- **Channel list modes**

```tcl
::rete::ischanban ban #channel
::rete::ischanexempt exempt #channel
::rete::ischaninvite invite #channel
```

These commands check if a specific mask is present on the channel's ban list,
exception list, or invite-exempt list. Each returns `1` if found, `0` otherwise.

```tcl
::rete::chanbans #channel
::rete::chanexempts #channel
::rete::chaninvites #channel
```

These commands return Tcl lists of entries for the respective channel lists.
Each entry is a sublist of the form `{mask bywho age}`:

- `mask`: the ban/exempt/invite mask
- `bywho`: the nickname or server that set the entry (may be empty)
- `age`: seconds since the entry was set (may be empty if timestamp is unknown)

Example:

```tcl
# Check if a ban exists
if {[::rete::ischanban *!*@*.example.com #channel]} {
    ::rete::debug "User is banned"
}

# List all bans
set bans [::rete::chanbans #channel]
foreach entry $bans {
    set mask [lindex $entry 0]
    set bywho [lindex $entry 1]
    set age [lindex $entry 2]
    ::rete::debug "Ban: $mask (set by $bywho, age: $age seconds)"
}
```

### Helper commands (setters)

Scripts can also update certain fields on `Person` entries (Rete's cached
view of users on the current connection).

- **Color**

```tcl
::rete::setcolor nick #RRGGBB
```

Sets `color` for `nick` to the given hex color (normalized by Rete).
This affects how that nick is rendered in the UI (subject to theme rules).

- **Bot flag**

```tcl
::rete::setbot nick 0|1
```

Sets `Person.isBot` for `nick` to `1` (true) or `0` (false). This can be used
by themes or scripts to treat known bots differently.

### Helper commands (buffer manipulation)

- **Append message to buffer**

```tcl
::rete::appendmsg <buffer> <sender> <message> <category> ?tags...?
```

Appends a message to the specified buffer. This allows scripts to add
messages to buffers programmatically, which can be useful for custom logging,
notifications, or bot responses.

- **`buffer`**: Buffer identifier. This can be either:
  - A buffer name (channel name like `#channel`, DM peer name like `nickname`,
    or system/script/web buffer name), **or**
  - A buffer UUID string returned by [`getbuffer`](#getbuffer)
- **`sender`**: Sender nickname (use `""` for no sender, i.e., server/system messages)
- **`message`**: Message text content
- **`category`**: Message category (see below for valid categories)
- **`tags`**: Optional space-separated list of message tags (see below for valid tags)

**Valid categories:**

- `normal` – Regular channel or DM message
- `serverNotice` – Server notice message
- `serverReply` – Server numeric reply
- `info` – Informational message
- `error` – Error message
- `join` – User join event
- `part` – User part event
- `quit` – User quit event
- `topic` – Topic change/info
- `nick` – Nickname change
- `notice` – Notice message
- `mode` – Mode change
- `kick` – Kick event
- `action` – CTCP ACTION (`/me`)
- `ctcp` – CTCP message
- `invite` – Invite event

**Valid tags:**

- `server` – Server/system message
- `channel` – Channel message
- `dm` – Direct message
- `notice` – Notice message
- `reply` – Reply message
- `error` – Error message
- `info` – Info message
- `script` – Script-generated message
- `raw` – Raw message
- `in` – Incoming message
- `out` – Outgoing message
- `bold` – Bold text
- `highlight` – Highlighted message
- `silence` – Silenced user

Returns nothing on success. If the buffer is not found, an error message is
written to the Script Debug buffer.

- **Create buffer**

```tcl
::rete::createbuffer name type ?inputCallback? ?hidden?
```

Creates a new buffer with the specified name and type.

- **`name`**: Buffer name (string)
- **`type`**: Buffer type (one of: `channel`, `directMessage`, `web`, `script`)
- **`inputCallback`** (optional): For script buffers only - TCL function name to call when input is sent. Use empty string `""` to explicitly disable input, or omit to disable by default.
- **`hidden`** (optional): `"1"` or `"true"` to create buffer hidden, `"0"` or omitted for visible

**Buffer Types:**

- `channel` - Creates a channel buffer (equivalent to joining a channel)
- `directMessage` - Creates a direct message buffer (equivalent to opening a DM)
- `web` - Creates a web content buffer for displaying URLs
- `script` - Creates a script-controlled buffer with optional input callback

**Script Buffers:**

Script buffers are special buffers that can have an optional input callback function. When a callback function is provided and the user sends input in the buffer:

1. The callback function is called with two arguments:
   - `bufferName`: The name of the buffer
   - `inputText`: The text that was entered

2. If no callback is provided (or an empty string is passed), input is disabled for the buffer.

**Notes:**

- Buffers created by scripts are automatically removed when the script is unloaded
- On script reload (rehash), callback functions are validated - if a callback function no longer exists, input is automatically disabled for that buffer
- Only script buffers can have input callbacks; other buffer types use their standard input behavior

Example (create/append):

```tcl
# Append a normal message to a channel
::rete::appendmsg "#channel" "BotName" "Hello from script!" "normal" "channel"

# Append a server notice-style message (no sender)
::rete::appendmsg "#channel" "" "This is a system message" "serverNotice" "server"

# Append an error message
::rete::appendmsg "#channel" "" "Something went wrong!" "error" "error"

# Append a join event
::rete::appendmsg "#channel" "SomeUser" "has joined" "join" "channel"

# Append to a DM buffer
::rete::appendmsg "nickname" "BotName" "Private message" "normal" "dm"
```

```tcl
# Create a script buffer with input callback
proc my_buffer_input {bufferName text} {
    ::rete::msg #channel "Received: $text"
}
::rete::createbuffer "MyBuffer" script "my_buffer_input"

# Create a channel buffer
::rete::createbuffer "#test" channel

# Create a hidden script buffer without input
::rete::createbuffer "LogBuffer" script "" 1
```

- **Resolve buffer to UUID**

```tcl
::rete::getbuffer <name> <type>
```

Resolves a buffer by name and type and returns its UUID string. This is useful
to avoid name collisions (e.g. a DM and a script buffer both called "nick")
when using commands like `appendmsg`, `hidebuffer`, or `setbuffericon`.

- **`name`**: Buffer name (channel name like `#channel`, DM peer name like
  `nickname`, script buffer name, etc.)
- **`type`**: One of `channel`, `direct`, `dm`, `directMessage`, `web`,
  `script`, `system`, `raw`

Returns the buffer UUID as a string on success, or an empty string if no
matching buffer is found.

Example:

```tcl
# Get the UUID for a script buffer called "test"
set bufId [::rete::getbuffer "test" script]

if {$bufId ne ""} {
    # Append a message using the UUID
    ::rete::appendmsg $bufId "" "Hello from script buffer" "normal" "script"

    # Hide the buffer using the UUID
    ::rete::hidebuffer $bufId 1

    # Set a custom icon using the UUID
    ::rete::setbuffericon $bufId "star.fill"
}
```

- **Get server buffer UUID**

```tcl
::rete::getserverbuffer
```

Returns the server buffer UUID for the active session. This is a convenience
function that avoids needing to know the server buffer's name ("Server") and
type ("system").

Returns the buffer UUID as a string on success, or an empty string if there is
no active session.

Example:

```tcl
# Get the server buffer and append a message
set serverBufId [::rete::getserverbuffer]
if {$serverBufId ne ""} {
    ::rete::appendmsg $serverBufId "" "System message" "info" "server"
}
```

- **Set buffer icon**

```tcl
::rete::setbuffericon <buffer> <iconName>
```

Sets a custom SF Symbol icon for a buffer. If the icon name is invalid (SF Symbol doesn't exist) or empty, resets to the default icon based on buffer kind.

- **`buffer`**: Buffer identifier (name or UUID from `getbuffer`)
- **`iconName`**: SF Symbol name (e.g., `"star.fill"`, `"heart.circle"`). Use empty string `""` to reset to default.

If the specified SF Symbol doesn't exist, the command will log a warning and reset to the default icon for that buffer type.

Example:

```tcl
# Set a custom icon for a script buffer
::rete::setbuffericon "MyBuffer" "star.fill"

# Set a different icon
::rete::setbuffericon "MyBuffer" "heart.circle"

# Reset to default icon
::rete::setbuffericon "MyBuffer" ""
```

- **Hide/show buffer**

```tcl
::rete::hidebuffer <buffer> <hidden>
```

Hide or show a buffer in the sidebar. This works for any buffer type.

- **`buffer`**: Buffer identifier (name or UUID from `getbuffer`)
- **`hidden`**: `"1"` or `"true"` to hide, `"0"` or `"false"` to show

Example:

```tcl
# Hide a buffer
::rete::hidebuffer "MyBuffer" 1

# Show a buffer
::rete::hidebuffer "MyBuffer" 0

# Hide a channel
::rete::hidebuffer "#test" true

# Show a channel
::rete::hidebuffer "#test" false
```

- **Delete buffer**

```tcl
::rete::delbuffer <buffer>
```

Delete (close) a buffer. This will remove the buffer from the sidebar and
discard its messages from the current session.

- **`buffer`**: Buffer identifier (name or UUID from `getbuffer`)

**Notes:**

- `delbuffer` **cannot** delete `system` or `raw` buffers; attempts will log an
  error to the Script Debug buffer and do nothing.
- For `channel` and `directMessage` buffers this behaves like closing the tab:
  the channel/DM is closed in the UI but the server state (join/part) is not
  affected.
- For `web` buffers it removes the web tab.
- For `script` buffers it removes the script-controlled buffer and its
  messages; if the script is still loaded it can re-create the buffer.

### Persistent storage

Scripts can store persistent data that survives across sessions using the `store`
command. Data is organized by script namespace, ensuring that each script can
only access its own data.

**Important:** Scripts must have a `Namespace:` metadata field defined to use
storage commands. If a script attempts to use storage without a namespace, an
error will be returned.

**Storage types:**

- **String**: Store arbitrary string values (since Tcl is string-based, this includes numeric values)
- **List**: Store Tcl lists (arrays of strings)
- **Dict**: Store Tcl dictionaries (key-value mappings)

**String operations:**

```tcl
::rete::store setstring <key> <value>
::rete::store getstring <key>
```

- `::rete::store setstring` - Sets a string value for the given key in the script's namespace
- `::rete::store getstring` - Gets a string value for the given key. Returns empty string if not found.

**Note:** Since Tcl is string-based, numeric values should be stored as strings. Tcl will automatically convert strings to numbers when needed for arithmetic operations using `expr` or `incr`.

**List operations:**

```tcl
::rete::store setlist <key> <value1> <value2> ...
::rete::store getlist <key>
::rete::store lappend <key> <value> ...
::rete::store lremove <key> <value>
```

- `::rete::store setlist` - Sets a list value, replacing the entire list. Accepts multiple values.
- `::rete::store getlist` - Gets a list value. Returns a proper Tcl list object, or empty list if not found.
- `::rete::store lappend` - Appends one or more values to an existing list (like Tcl's `lappend`). Creates a new list if the key doesn't exist.
- `::rete::store lremove` - Removes all occurrences of a value from a list.

**Dict operations:**

```tcl
::rete::store setdict <key> <dictKey1> <dictValue1> <dictKey2> <dictValue2> ...
::rete::store getdict <key>
::rete::store dictset <key> <dictKey> <value>
::rete::store dictget <key> <dictKey>
::rete::store dictunset <key> <dictKey>
```

- `::rete::store setdict` - Sets a dict value, replacing the entire dict. Accepts key-value pairs as separate arguments.
- `::rete::store getdict` - Gets a dict value. Returns a proper Tcl dict object, or empty dict if not found.
- `::rete::store dictset` - Sets a value within a dict (like Tcl's `dict set`). Creates a new dict if the key doesn't exist.
- `::rete::store dictget` - Gets a value from a dict (like Tcl's `dict get`). Returns empty string if not found.
- `::rete::store dictunset` - Removes a key from a dict (like Tcl's `dict unset`).

**Notes:**

- All storage operations are scoped to the script's namespace (from the `Namespace:` metadata field)
- Data persists across sessions in a JSON file at `~/Library/Application Support/Rete/script_storage.json`
- Scripts can only access data in their own namespace - namespace isolation is enforced
- List operations return proper Tcl list objects that can be used with standard Tcl list commands
- Dict operations return proper Tcl dict objects that can be used with standard Tcl dict commands
- If you need to store a string with spaces, use `setstring`/`getstring`, not list operations

**Examples:**

```tcl
# Store a string
::rete::store setstring username "Alice"
set user [::rete::store getstring username]

# Store a numeric value as a string (Tcl will convert when needed)
::rete::store setstring message_count "42"
set count [::rete::store getstring message_count]
# Use in arithmetic: expr {$count + 1} or incr count

# Store a list (replace entire list)
::rete::store setlist favorites "item1" "item2" "item3"

# Get the list back
set favs [::rete::store getlist favorites]
# favs is now: {item1 item2 item3}

# Append to the list (like Tcl's lappend)
::rete::store lappend favorites "item4" "item5"
# favorites is now: {item1 item2 item3 item4 item5}

# Remove a value from the list
::rete::store lremove favorites "item2"
# favorites is now: {item1 item3 item4 item5}

# Store a string with spaces (not a list)
::rete::store setstring my_message "This is a string with spaces"
set msg [::rete::store getstring my_message]
# msg is: "This is a string with spaces"

# Store a dict (key-value pairs)
::rete::store setdict config "host" "irc.example.com" "port" "6667" "ssl" "1"

# Get the dict back
set cfg [::rete::store getdict config]
# cfg is now a proper Tcl dict: {host {irc.example.com} port 6667 ssl 1}

# Set a value in the dict
::rete::store dictset config "nick" "MyBot"

# Get a value from the dict
set host [::rete::store dictget config "host"]
# host is: "irc.example.com"

# Remove a key from the dict
::rete::store dictunset config "port"
```

**Complete example:**

```tcl
# Title: Example Script
# Namespace: example

# Track message count per channel
proc on_chanmsg {from channel text serverTime} {
    set count [::rete::store getstring "count_$channel"]
    if {$count eq ""} {
        set count 0
    } else {
        set count [expr {int($count)}]
    }
    incr count
    ::rete::store setstring "count_$channel" $count
    
    # Store recent messages
    ::rete::store lappend "recent_$channel" "$from: $text"
    
    # Keep only last 10 messages
    set recent [::rete::store getlist "recent_$channel"]
    if {[llength $recent] > 10} {
        set recent [lrange $recent end-9 end]
        ::rete::store setlist "recent_$channel" {*}$recent
    }
    
    return 0
}

::rete::bind CHANMSG on_chanmsg
```

- **Set web buffer URL**

```tcl
::rete::setweburl <buffer> <url>
```

Sets the URL for a **web buffer**.

- **`buffer`**: Either
  - A buffer name (web buffer name), **or**
  - A buffer UUID string returned by [`getbuffer`](#getbuffer)
- **`url`**: The website URL to load inside the web buffer (e.g. `https://example.com`).

If the buffer is not found or is not a web buffer, an error is logged to the
Script Debug buffer and the command is otherwise ignored.

Example:

```tcl
proc open_rete_website {} {
    # Create or find a web buffer
    ::rete::createbuffer "Rete Website" web

    # Point it to the desired URL
    ::rete::setweburl "Rete Website" "https://rete.chat/"
}
```

- **Add nicklist menu item**

```tcl
::rete::addnickmenu <label> <callback>
```

Adds a custom menu item to the nicklist context menu (right-click on nickname).

- **`label`**: Menu item label/text
- **`callback`**: Tcl function name to call when the menu item is clicked. The function must exist when `addnickmenu` is called.

The callback function will be called with **three arguments**:

```tcl
proc my_nick_action {bufferId bufferName nick} {
    # bufferId   - UUID of the buffer where the nick was right-clicked
    # bufferName - Name of that buffer (e.g. #channel)
    # nick       - The nickname that was right-clicked
}
```

**Notes:**

- Menu items are automatically removed when the script is unloaded
- On script reload (rehash), callback functions are validated - if a callback function no longer exists, the menu item is automatically removed
- Multiple scripts can add menu items; they will all appear in the context menu
- Menu items appear after the built-in menu items (Whois, Block/Unblock) and before the Control menu (if you're an op)

Example:

```tcl
# Define a callback function
proc my_nick_action {bufferId bufferName nick} {
    ::rete::msg $bufferName "Hello $nick from script! (buffer $bufferName / $bufferId)"
}

# Add the menu item
::rete::addnickmenu "Say Hello" "my_nick_action"
```

### Dynamic nicklist menu providers

For fully dynamic, per-nick context menus that are computed only when a nick is
right-clicked, you can register a **nickmenu provider**:

```tcl
::rete::setnickmenuprovider <callback>
```

- **`callback`**: Tcl function name to call when a nick is right-clicked. The
  function must exist when `setnickmenuprovider` is called.

The provider function will be called with three arguments:

```tcl
proc my_nickmenu_provider {bufferId bufferName nick} {
    # bufferId   - UUID of the buffer where the nick was right-clicked
    # bufferName - Name of that buffer (e.g. #channel)
    # nick       - The nickname that was right-clicked

    # Return a Tcl list of label/callback pairs:
    #   {label1 callback1 label2 callback2 ...}
    set items {}
    lappend items "Whois $nick" my_whois_proc
    lappend items "Greet $nick" my_greet_proc

    return $items
}

proc my_whois_proc {bufferId bufferName nick} {
    ::rete::putserv "WHOIS $nick"
}

proc my_greet_proc {bufferId bufferName nick} {
    ::rete::msg $bufferName "Hello $nick from dynamic menu (buffer $bufferName / $bufferId)"
}

# Register the provider
::rete::setnickmenuprovider my_nickmenu_provider
```

**Notes:**

- Multiple scripts may call `::rete::setnickmenuprovider`; all providers will be
  invoked and their menu items concatenated.
- Providers are called **only when the nicklist context menu is opened**, not
  on every UI re-render.
- Providers and their menu items are automatically cleaned up when the owning
  script is unloaded or rehashed.

### Store header, channel buffer, and nick buffer menus

In addition to nicklist entries, scripts can attach menu items to:

- The **store header** (server row in the sidebar).
- The **top‑level store menu** in the macOS menu bar (one menu per connection).
- **Channel buffers** (right-click on a channel buffer in the sidebar).
- **Nick/DM buffers** (right-click on a direct message buffer in the sidebar).

You can use simple static buttons (registered once at load time) or dynamic
providers that are evaluated each time the menu is opened.

- **Add store header button**

```tcl
::rete::addstoreheaderbutton <label> <callback>
```

Adds a custom menu item to the **store header** context menu.

- **`label`**: Menu item label/text.
- **`callback`**: Tcl function name to call when the menu item is clicked. The function must exist when `addstoreheaderbutton` is called.

The callback function will be called with **two arguments**:

```tcl
proc my_storeheader_action {storeId serverName} {
    # storeId    - UUID of the IRC store/connection
    # serverName - Human-readable server name/address
}
```

Example:

```tcl
proc my_storeheader_action {storeId serverName} {
    ::rete::echo "Clicked store header for $serverName ($storeId)"
}

::rete::addstoreheaderbutton "My Store Action" my_storeheader_action
```

- **Add channel buffer menu item**

```tcl
::rete::addchannelbuffermenu <label> <callback>
```

Adds a custom menu item to **channel buffer** context menus (right-click on a
channel buffer in the sidebar).

- **`label`**: Menu item label/text.
- **`callback`**: Tcl function name to call when the menu item is clicked.

The callback will be invoked with **two arguments**:

```tcl
proc my_channelbuffer_action {bufferId bufferName} {
    # bufferId   - UUID of the channel buffer
    # bufferName - Channel name (e.g. #channel)
}
```

Example:

```tcl
proc my_channelbuffer_action {bufferId bufferName} {
    ::rete::msg $bufferName "Channel buffer menu clicked for $bufferName ($bufferId)"
}

::rete::addchannelbuffermenu "Channel Script Action" my_channelbuffer_action
```

- **Add nick buffer (DM) menu item**

```tcl
::rete::addnickbuffermenu <label> <callback>
```

Adds a custom menu item to **nick/DM buffer** context menus (right-click on a
direct message buffer in the sidebar).

- **`label`**: Menu item label/text.
- **`callback`**: Tcl function name to call when the menu item is clicked.

The callback will be invoked with **three arguments**:

```tcl
proc my_nickbuffer_action {bufferId bufferName nick} {
    # bufferId   - UUID of the DM buffer
    # bufferName - Name of the buffer (usually the nick)
    # nick       - Nickname for this DM
}
```

Example:

```tcl
proc my_nickbuffer_action {bufferId bufferName nick} {
    ::rete::msg $bufferName "DM menu clicked for $nick in $bufferName ($bufferId)"
}

::rete::addnickbuffermenu "DM Script Action" my_nickbuffer_action
```

### Dynamic store header / buffer menu providers

For fully dynamic menus that are computed only when the context menu is opened,
you can register providers similar to nicklist and hover providers:

- **Store header menu provider**

```tcl
::rete::setstoreheadermenuprovider <callback>
```

The provider is called with:

```tcl
proc my_storeheader_menu_provider {storeId serverName} {
    # Return {label1 callback1 label2 callback2 ...}
    set items {}
    lappend items "Reconnect $serverName" my_reconnect_proc
    return $items
}
```

### Top‑level store menu buttons

For the macOS menu bar, Rete exposes a **top‑level store menu** for each
connection (the menu whose title is the current store/connection name). Scripts
can add custom items to this menu:

- **Add top‑level store menu button**

```tcl
::rete::addmenubutton <label> <callback>
```

Adds a custom menu item to the **top‑level store menu** for the current
connection in the macOS menu bar.

- **`label`**: Menu item label/text.
- **`callback`**: Tcl function name to call when the menu item is clicked. The
  function must exist when `addmenubutton` is called.

The callback function will be called with **two arguments**, identical to
`addstoreheaderbutton`:

```tcl
proc my_storemenu_action {storeId serverName} {
    # storeId    - UUID of the IRC store/connection
    # serverName - Human-readable server name/address
}
```

Example:

```tcl
proc my_storemenu_action {storeId serverName} {
    ::rete::echo "Clicked top-level store menu for $serverName ($storeId)"
}

::rete::addmenubutton "My Store Menu Action" my_storemenu_action
```

- **Channel buffer menu provider**

```tcl
::rete::setchannelbuffermenuprovider <callback>
```

The provider is called with:

```tcl
proc my_channelbuffer_menu_provider {bufferId bufferName} {
    set items {}
    lappend items "Say hi in $bufferName" my_channel_hi_proc
    return $items
}
```

- **Nick buffer (DM) menu provider**

```tcl
::rete::setnickbuffermenuprovider <callback>
```

The provider is called with:

```tcl
proc my_nickbuffer_menu_provider {bufferId bufferName nick} {
    set items {}
    lappend items "Note about $nick" my_dm_note_proc
    return $items
}
```

**Notes for these providers:**

- Providers are called **only when the corresponding context menu is opened**.
- Multiple scripts can register providers; all providers are called and their
  menu items concatenated.
- Static buttons added via `addstoreheaderbutton`, `addchannelbuffermenu`, and
  `addnickbuffermenu` are combined with dynamic provider items.
- Items and providers are automatically cleaned up when the owning script is
  unloaded or rehashed.

### Dynamic hover-card (user info tooltip) buttons

You can also add buttons to the **hover card** that appears when hovering over
nicknames (user info tooltip). These are computed dynamically when the tooltip
is shown, similar to dynamic nicklist menu providers:

```tcl
::rete::setnickhoverprovider <callback>
```

- **`callback`**: Tcl function name to call when the hover card is shown. The
  function must exist when `setnickhoverprovider` is called.

The provider function will be called with three arguments:

```tcl
proc my_nickhover_provider {bufferId bufferName nick} {
    # bufferId   - UUID of the buffer context (channel or DM), or "" if unknown
    # bufferName - Name of that buffer (e.g. #channel or nick), or "" if unknown
    # nick       - The nickname being hovered

    # Return a Tcl list of label/callback pairs:
    #   {label1 callback1 label2 callback2 ...}
    set items {}
    lappend items "Whois" my_whois_proc
    lappend items "Greet" my_greet_proc

    return $items
}

proc my_whois_proc {bufferId bufferName nick} {
    ::rete::putserv "WHOIS $nick"
}

proc my_greet_proc {bufferId bufferName nick} {
    # Send greeting to the current buffer if known, or fall back to using the nick
    if {$bufferName ne ""} {
        ::rete::msg $bufferName "Hello $nick from hover card (buffer $bufferName / $bufferId)"
    } else {
        ::rete::msg $nick "Hello from hover card"
    }
}

# Register the provider
::rete::setnickhoverprovider my_nickhover_provider
```

**Notes:**

- Providers are called **only when the hover card is visible**, not for every
  mouse-move event.
- For nicklist hovers, `bufferId`/`bufferName` will be the channel buffer.
- For message hovers:
  - If the message is in a channel buffer, `bufferId`/`bufferName` refer to that channel.
  - If the message is in a direct-message buffer with this nick, they refer to that DM.
  - Otherwise, both may be empty strings (`""`), so scripts should handle that case.
- Providers and their buttons are automatically cleaned up when the owning
  script is unloaded or rehashed.

### Helper commands (sending commands)

In addition to getters, Tcl scripts can send commands to the IRC server via
the following helpers. All of them operate on the current connection
(`IRCStore.transport`).

- **Raw lines**

```tcl
::rete::putserv RAWLINE...
```

Example:

```tcl
::rete::putserv PRIVMSG #channel :Hello from Tcl
```

- **Basic commands**

```tcl
::rete::join #channel ?key?
::rete::part #channel ?reason...?
::rete::msg target text...
::rete::notice target text...
::rete::ctcp target command ?params...?
::rete::action target text...
::rete::topic_set #channel ?newTopic...?
::rete::nick newNick
::rete::quit ?message...?
::rete::mode target modes ?params...?
::rete::kick #channel nick ?reason...?
::rete::silence ?input...?
```

These are thin wrappers around Rete's internal command/transport layer and
behave like the corresponding `/join`, `/part`, `/msg`, `/notice`, `/ctcp`,
`/me`, `/topic`, `/nick`, `/quit`, `/mode`, `/kick` and `/silence` commands
typed in the client, with `putserv` exposing raw access to the underlying
connection.

### Custom commands

Scripts can register custom commands that will be called when the user types
`/command` in the input field. Custom commands can override native commands.

- **Register a custom command**:

```tcl
::rete::addcommand command callback
```

- **`command`**: The command name (case-insensitive). When a user types `/command`,
  this callback will be invoked.
- **`callback`**: Tcl function name to call when the command is executed. The
  function must exist when `addcommand` is called.

The callback function will be called with **three arguments**:

```tcl
proc my_custom_command {params bufferId bufferName} {
    # params     - Everything after the command name (the remaining text)
    # bufferId   - UUID of the buffer where the command was executed
    # bufferName - Name of that buffer (e.g. #channel, nickname, or system buffer name)
}
```

**Notes:**

- Custom commands are automatically removed when the script is unloaded
- On script reload (rehash), custom commands are automatically removed and must be
  re-registered if the script still needs them
- If a custom command is registered with the same name as a native command, the
  custom command will override the native command
- Custom commands appear in tab autocomplete alongside native commands
- The callback function receives the full remaining text after the command name as
  a single string, allowing the script to parse it however it needs

Example:

```tcl
# Define a callback function
proc my_greet {params bufferId bufferName} {
    if {$params eq ""} {
        ::rete::msg $bufferName "Hello! Usage: /greet <name>"
    } else {
        ::rete::msg $bufferName "Hello, $params!"
    }
}

# Register the command
::rete::addcommand greet my_greet

# Now /greet Alice will call my_greet with params="Alice"
```

```tcl
# Override the native /join command
proc my_custom_join {params bufferId bufferName} {
    # Parse the channel name from params
    set channel [lindex [split $params] 0]
    if {$channel eq ""} {
        ::rete::debug "Usage: /join #channel"
        return
    }
    
    # Add a custom message before joining
    ::rete::debug "Joining $channel with custom handler"
    
    # Call the native join command
    ::rete::join $channel
}

# Register to override native /join
::rete::addcommand join my_custom_join
```

- **CAP capability management**

```tcl
::rete::cap ls
::rete::cap values
::rete::cap values <capability>
::rete::cap enabled
::rete::cap req "cap1 cap2 ..."
::rete::cap raw "CAP SUBCOMMAND ..."
```

Manages IRCv3 capabilities (CAP):

- `cap ls` – Returns a list of capabilities the server supports (from CAP LS).
- `cap values` – Returns a Tcl dict of all capabilities and their associated
  values (CAP 302 format). Each entry is `{capability {value1 value2 ...}}`.
- `cap values <capability>` – Returns a list of values for the specified
  capability, or empty list if the capability has no values or doesn't exist.
- `cap enabled` – Returns a list of capabilities currently enabled/negotiated
  with the server.
- `cap req "cap1 cap2 ..."` – Requests the specified capabilities from the
  server. The argument should be a space-separated string of capability names.
- `cap raw "CAP SUBCOMMAND ..."` – Sends a raw CAP command to the server.
  For example: `cap raw "REQ :foo bar"`.

Examples:

```tcl
# List server capabilities
set caps [::rete::cap ls]
::rete::debug "Server supports: $caps"

# Check if SASL is enabled
set enabled [::rete::cap enabled]
if {"sasl" in $enabled} {
    ::rete::debug "SASL is enabled"
}

# Get SASL mechanisms
set mechs [::rete::cap values sasl]
::rete::debug "SASL mechanisms: $mechs"

# Request a capability
::rete::cap req "cap-notify"

# Send raw CAP command
::rete::cap raw "LS 302"
```

### Timer commands

Rete provides timer functionality similar to Eggdrop's `timer` and `utimer`
commands, allowing scripts to schedule Tcl commands to execute at regular
intervals.

- **Minute-based timers**

```tcl
::rete::timer minutes commandName ?count? ?timerName?
```

Creates a timer that fires every `minutes` minutes, aligned to the top of the
minute. The first fire occurs at the next minute boundary plus `minutes`.

- `minutes`: positive number (can be fractional, e.g., `0.5` for 30 seconds)
- `commandName`: Tcl command or procedure name to execute
- `count` (optional): number of times to fire (default: infinite). Use `0` for
  infinite repeats.
- `timerName` (optional): unique identifier for this timer. If omitted, Rete
  generates one automatically.

Returns the timer name (useful when auto-generated).

Example:

```tcl
# Fire every 5 minutes, 10 times
::rete::timer 5 my_periodic_task 10 mytask

# Fire every 30 seconds, infinite
::rete::timer 0.5 check_status 0
```

- **Second-based timers (utimer)**

```tcl
::rete::utimer seconds commandName ?count? ?timerName?
```

Creates a timer that fires every `seconds` seconds, starting immediately
(now + seconds).

- `seconds`: positive number
- `commandName`: Tcl command or procedure name to execute
- `count` (optional): number of times to fire (default: infinite). Use `0` for
  infinite repeats.
- `timerName` (optional): unique identifier for this timer

Returns the timer name.

Example:

```tcl
# Fire every 10 seconds, 5 times
::rete::utimer 10 check_connection 5 conncheck

# Fire once after 30 seconds
::rete::utimer 30 delayed_action 1
```

- **List active timers**

```tcl
::rete::timers
::rete::utimers
```

Returns a space-separated list of active timers. Each entry contains:
`timerName intervalSeconds count nextFireAt command`

- `timerName`: the timer's identifier
- `intervalSeconds`: the interval in seconds
- `count`: remaining fires (`infinite` if unlimited)
- `nextFireAt`: ISO-8601 timestamp of next fire
- `command`: the command that will be executed

- **Kill timers**

```tcl
::rete::killtimer timerName
::rete::killutimer timerName
```

Cancels and removes the specified timer. Returns nothing on success, or an
error if the timer doesn't exist.

Example:

```tcl
set t [::rete::timer 5 my_task 0]
# ... later ...
::rete::killtimer $t
```

**Notes:**

- Timers are automatically cleaned up when the interpreter is destroyed (e.g.,
  when the IRCStore is deallocated).
- Timer commands execute in the Tcl interpreter context. Errors in the command
  are logged to the Script Debug buffer but do not stop the timer.
- For `::rete::timer` (minute-based), the first fire is aligned to the top of the
  minute. For example, if you create a `::rete::timer 5` at 12:34:20, it will first
  fire at 12:35:00, then every 5 minutes thereafter.
- For `::rete::utimer` (second-based), the first fire occurs `seconds` after creation,
  then repeats every `seconds` thereafter.

### The `debug` command

Scripts can write to the per‑store **Script Debug** buffer using:

```tcl
::rete::debug ?level? text...
```

- **`level`** (optional): one of `error`, `warning`, `info`, `debug`.
  - Default is `info` if no level or an unknown level is given.`
- **`text...`**: free‑form text; all remaining arguments are joined with spaces.

Rete maps debug verbosity levels with `script` as the theme `category` and `error`, `warning`, `debug` and `info`as theme `tags`.

### Script UI API (`ui::` namespace)

Scripts can build script-driven windows and forms using the `::ui::` namespace.

**View:**

- **Load a view from JSON** (async; view handle is passed to the callback):

```tcl
::ui::view load <jsonPath> ?-callback <proc>?
```

- **Destroy a view:**

```tcl
::ui::view destroy <viewHandle>
```

- **Set view title:**

```tcl
::ui::view set <viewHandle> -title <text>
```

- **Resolve element handle** (for use with label, text, table, input, bind):

```tcl
::ui::view elem <viewHandle> <localId>
```

Returns the element handle (e.g. for `localId` `lblToken` in the JSON).

**View JSON layout — how to build the JSON**

The view file is a single JSON object with two top-level keys:

- **`title`** (optional string) — window title.
- **`widgets`** (optional array) — list of root widgets. Widgets are laid out in order, top to bottom. Containers group children vertically (`container` / `vbox`) or horizontally (`hbox`).

**Widget object**

Each widget is an object with:

- **`type`** (required string) — one of: `label`, `text`, `button`, `input`, `table`, `container`, `vbox`, `hbox`.
- **`id`** (optional string) — local identifier. **Required** for any widget you reference from Tcl (`::ui::view elem`, `::ui::bind`, or `copyFromId`). Use stable names like `lblTitle`, `btnSave`, `inputSeed`, `tblScores`. Containers do not need an `id` unless used for structure only.
- **`props`** (optional object) — string-to-string map for initial state and styling. Keys depend on widget type (see below).
- **`children`** (optional array) — only for containers. List of child widget objects. Order is layout order (vertical or horizontal).

**Widget types and props**

- **`label`** — single-line text.  
  Props: `text` (initial text), `size` (`large` | `small` | omit for default), `font` (`monospaced` or `mono`), `alignment` (`center` | `trailing` | `right` | `leading`), `selectable` (`true` | `false`; default false).

- **`text`** — multi-line, scrollable text (append/clear from Tcl).  
  Props: `text` (initial content).

- **`button`** — clickable button; bind `onClick`.  
  Props: `title` (button label).

- **`input`** — single-line text field; Enter fires `onSubmit`, use `::ui::input get` to read value.  
  Props: `text` (initial value).

- **`table`** — rows keyed by rowId; bind `onSelect`, `onDoubleClick`. Set columns and rows from Tcl.  
  Props: `columns` — comma-separated column names (e.g. `"Player,Score,Level"`). Initial rows can be added from Tcl after load.

- **`container`** / **`vbox`** — stacks `children` vertically.  
  Props: `alignment` (`center` | `trailing` | `leading`), `padding` (number, or use `paddingVertical` / `paddingHorizontal`), `background` (e.g. `green`, `gray`, `blue`, `red`, `orange`, `accent`), `backgroundOpacity` (0–1, default 0.12), `borderColor`, `borderWidth`, `cornerRadius`, `divider` (`true` adds a divider above the content). Optional copy-on-click: `copyOnClick` `true` and `copyFromId` set to the **local id** of a label whose text is copied to the clipboard when the container is clicked.

- **`hbox`** — lays out `children` in a horizontal row (e.g. Save and Close buttons side by side).  
  Same styling props as `container` if you need padding/background on the row.

**Minimal example**

```json
{
  "title": "My Window",
  "widgets": [
    { "type": "label", "id": "lblTitle", "props": { "text": "Hello" } },
    { "type": "button", "id": "btnGo", "props": { "title": "Go" } }
  ]
}
```

**Example with containers and hbox**

```json
{
  "title": "Form",
  "widgets": [
    {
      "type": "container",
      "props": { "divider": "true" },
      "children": [
        { "type": "label", "id": "lblTarget", "props": { "text": "Target:" } },
        { "type": "input", "id": "inputTarget", "props": { "text": "" } },
        {
          "type": "hbox",
          "children": [
            { "type": "button", "id": "btnSave", "props": { "title": "Save" } },
            { "type": "button", "id": "btnClose", "props": { "title": "Close" } }
          ]
        }
      ]
    }
  ]
}
```

Use **pretty-printed** JSON (newlines and indentation) so the file is easy to edit. The app accepts any valid JSON for the structure above.

**Widgets:**

- **Label:**

```tcl
::ui::label set <elemHandle> -text <text>
```

- **Text** (multi-line; append or clear):

```tcl
::ui::text append <elemHandle> <text>
::ui::text clear <elemHandle>
```

- **Input** (single-line text field; Enter fires `onSubmit` with `::ui(event) data value` set to the current text):

```tcl
::ui::input set <elemHandle> -text <value>
::ui::input get <elemHandle>
```

- `ui::input get` returns the current value of the input (e.g. for a Save button that reads multiple fields).

- **Table** (rowId-keyed):

```tcl
::ui::table setcolumns <elemHandle> <column1> <column2> ...
::ui::table add <elemHandle> -id <rowId> -data <dict>
::ui::table update <elemHandle> -id <rowId> -data <dict>
::ui::table remove <elemHandle> <rowId>
::ui::table clear <elemHandle>
```

**Binding:**

- **Bind an event** (returns a token for unbind):

```tcl
::ui::bind <elemHandle> <eventName> <scriptBody>
```

- **Unbind:**

```tcl
::ui::unbind <bindingToken>
```

- **Error/info alert** (modal pop-up; does not block the script):

```tcl
::ui::alert -message <text> ?-title <text>? ?-style error|warning?
```

Shows a modal dialog with a badge on the left and the message text on the right. **-style** sets the badge: `error` (red, default) or `warning` (orange). Default **-title** is `Error`; use `-title` to override (e.g. `-title "Validation"`).

**Events:** `button` → `onClick`; view window → `onClose`; `table` → `onSelect`, `onDoubleClick`; `input` → `onSubmit` (Enter key; `::ui(event) data value` is the field text).

When an event fires, the script runs on the serial queue with `::ui(event)` set to a dict: `type`, `target`, `view`, `data`.

**Namespacing:** Element handles are generated as `<connectionId>::<scriptNamespace>::<viewInstance>::<localElementId>`. Scripts use local ids from the JSON (e.g. `tblScores`, `btnSave`); the app applies the prefix.

Example:

```tcl
set viewHandle ""   ;# set by callback
proc on_view_loaded {vh} {
    global viewHandle
    set viewHandle $vh
    set lbl [::ui::view elem $vh lblTitle]
    set btn [::ui::view elem $vh btnGo]
    ::ui::label set $lbl -text "Ready"
    set tok [::ui::bind $btn onClick {
        set n [::rete::get_current_nick]
        ::rete::debug "Clicked by $n"
    }]
}
::ui::view load "my_view.json" -callback on_view_loaded
```

Use a view JSON path that your script can resolve (e.g. a file next to the script via `[file join [file dirname [info script]] my_view.json]`, or a path under the app’s script directory).

**Behaviour:** UI events run your Tcl on the connection serial queue; UI updates are applied on the main thread. The app does not block the serial queue waiting for the main thread, so your handlers can safely call UI commands.
