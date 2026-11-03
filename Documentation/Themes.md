---
layout: doc
title: Themes Tutorial
permalink: /docs/themes/
---
Rete supports custom themes defined as JSON files. Themes provide UI colors and message colors for both Light and Dark variants. Users can import themes from Preferences or by placing files in the Themes folder.

## Where to put theme files

The themes folder is always `Themes/` inside Rete's per-user configuration directory. Where
*that* is depends on the platform, and on macOS on whether the program is sandboxed:

| You are running | Themes folder |
|---|---|
| **Rete.app** on macOS (sandboxed) | `~/Library/Containers/masterbofh.Rete/Data/Library/Application Support/Rete/Themes` |
| **reTerm** on macOS (not sandboxed) | `~/Library/Application Support/Rete/Themes` |
| **reTerm** on Linux | `~/.rete/Themes` |
| **Rete** on iOS | `Library/Application Support/Rete/Themes` inside the app's own data container |

- The app creates this directory on first import or load attempt.
- On macOS you can also use Preferences → Appearance → “Import theme…” to copy a .json file into
  this directory, which saves you finding it.
- In Rete on Linux, **Settings → Preferences → Theme** has the same **Import theme…**
  (it offers `.json` files, copies the one you choose into `~/.rete/Themes` and switches to it) and
  **Delete selected theme** (it removes the theme's file; the built-in `System` cannot be deleted,
  and deleting the theme in use switches to `System`). Both follow macOS's rules: a file that is not
  a theme is refused with nothing copied, and an imported file replaces one of the same name.
- **A `rete-theme://host/path` link** — clicked in a channel, or opened from a browser — is
  `https://host/path`. Rete (macOS, and Linux where the desktop hands it the link) **asks before it
  downloads anything**: the question names the file, the host and the full address, and says that
  importing saves the theme and switches to it. Cancel is the default; nothing is fetched or written
  unless you choose Import Theme. A theme is colours, fonts and symbol names — it runs nothing —
  but anyone in a channel can post a link. Preferences' own "Import from URL" field does not ask: you typed the address.

Notes:
- The last two path components are always “Rete/Themes”, and they do not depend on the bundle
  identifier. What changes between the rows above is the directory they hang off — Rete.app's
  sandbox container, your home directory's `Library/Application Support`, `~/.rete`, or the iOS
  app's own data container.
- **`~/Library/Application Support/Rete/Themes` is reTerm's folder, not Rete.app's.** The two
  macOS programs do not share a themes folder, because Rete.app is sandboxed and reTerm is not.
  Copying a theme into one does not put it in the other.
- Files must have the .json extension.
- **What an import calls the file it writes:**
  - Importing from a **URL** normally keeps the URL's own filename, so
    `https://example.com/themes/solarized.json` lands as `solarized.json`.
  - If the URL does not end in something Rete can use as a filename, the file is named after the
    theme's own `name` field instead — `"<name>.json"`, the name the picker shows. A URL is
    unusable when its last path component is empty, is `.` or `..`, contains a `/` (which a URL
    *can* do — a `%2F` decodes into it), contains a control character, is longer than the
    filesystem allows, or does not end in `.json`.
  - **If the theme's `name` is not usable as a filename either, the import refuses and says so**,
    naming both. Rete will not invent a name for you and will not silently shorten one: a file
    under a name you cannot predict is a theme you cannot find again.
- **What “Save As” is allowed to call a theme:**
  - Saving a new theme writes a file called `"<name>.json"`, so the theme's **name** has to be
    usable as a filename — the same test an import applies, and the same list: it may not be
    empty, may not be `.` or `..`, may not contain a `/` or a control character, and must be short
    enough for the filesystem once `.json` is on the end. **If it is not, “Save As” refuses and
    names the theme**, rather than writing the file somewhere you did not ask for.
  - A dot inside a name is fine — `Solarized 1.2` saves as `Solarized 1.2.json` — and so is a
    leading one (`.hidden`). What is refused is a name that is a *path* rather than a
    name, and a name that is **only** dots (`.`, `..`), because `".json"` and `"...json"` read as
    hidden files with no extension and the picker lists themes by their `.json` extension.
  - **Plain “Save” (overwrite)** is not held to this, including for a theme already on disk whose
    name is one of the above: it writes back to the file the theme was loaded from, which is a file inside
    the themes folder by construction. To save such a theme under a new file you have to rename it
    first.
- **An import never destroys anything until the replacement is known to be good.** Importing a
  file that turns out not to be a theme leaves the theme already under that filename as it was.
  Importing a valid theme over an existing one replaces it.
- **If a *folder* is sitting where the theme file would go, the import refuses** and tells you the
  path — *“…” is a folder, not a theme file, so Rete did not replace it.* The folder is not
  touched.
- **If the themes folder's path exists but is not a folder, saving and importing a theme say so**, in the
  same words — *“…” is a file, not a folder, so Rete cannot use it as your themes folder.* That
  covers Save, Save As, “Import theme…”, importing from a URL, and importing a folder of themes.
- **If that path exists but is not a folder, Rete says so** and falls back to the built-in
  `System` theme rather than behaving as though you had no themes. The same is true if the
  folder cannot be listed for any other reason. This is the same message on macOS, iOS, Linux
  and reTerm.

## The theme editor

Rete.app and Rete for Linux have a theme editor with the same rows. It covers the keys below,
including the `invite` category, `ui.secondary`, the date colour (`categories.dateDivider`) and
bold on or off for the `timestamp` and `literal` segments. It saves only what you change, so a
colour you leave unset stays unset and keeps following the default. An unset row shows the colour
the app draws in its place: for a category, `ui.foreground` (or the colour of a contextual tag the
line carries); for a segment, its fallback from *How a segment's colour is resolved*. An unset tag
colours nothing.

## JSON schema (overview)


Top-level object (`Theme`):
- `name`: `String` (unique theme name; used in the picker)
- `description`: `String` (optional; short human-readable description of the theme)
- `author`: `String` (optional)
- `version`: `String` (optional)
- `operatorBadge`: `String` (optional; SF Symbol name for IRC operator badge; `""` = no icon. Default: `"shield.checkered"`)
- `awayBadge`: `String` (optional; SF Symbol name for away user badge; `""` = no icon. Default: `"moon.fill"`)
- `secureBadge`: `String` (optional; SF Symbol name for secure connection badge; `""` = no icon. Default: `"lock.fill"`)
- `botBadge`: `String` (optional; SF Symbol name for bot user badge; `""` = no icon. Default: `"cpu"`)
- `blockedBadge`: `String` (optional; SF Symbol name for blocked user badge; `""` = no icon. Default: `"hand.raised.fill"`)

  **On Linux (Rete, the GTK app)** SF Symbols do not exist, so each of these five is translated to the nearest
  freedesktop symbolic icon in the desktop's icon theme — for example `shield.checkered` → `security-high-symbolic`,
  `moon.fill` → `weather-clear-night-symbolic`, `lock.fill` → `channel-secure-symbolic`, `cpu` →
  `computer-symbolic`, `hand.raised.fill` → `action-unavailable-symbolic`; `star.fill`, `gearshape`, `clock`,
  `person`, `crown` and their outline or `.fill` forms are known too. This is the **same table** Rete on Linux
  uses for the message badges, the buffer-list icons and a script's toolbar button (`addtoolbarbutton`), so the symbols known there — `bell` →
  `preferences-system-notifications-symbolic`, `trash` → `user-trash-symbolic`, `heart` →
  `emblem-favorite-symbolic`, `book`, `doc`, `globe`, `bolt`, `envelope` and the rest — draw their icon on a
  badge as well. A symbol Rete does not know draws the badge's default icon. The icon takes the badge's colour (`operatorBadgeColor` and the rest). To choose the
  Linux icon yourself, write the field as `{ "sfSymbol": …, "linux": [ … ] }` — see *Icon fields: an SF
  Symbol, and the Linux icon beside it*.
- `light`: `ThemePalette`
- `dark`: `ThemePalette`

`ThemePalette`:
- `ui`: `UI`
- `messageColors`: `MessageColors`
- `fonts`: `Fonts` (optional)

`Fonts` (optional):
- `messageFontName`: `String` (optional; custom font name, e.g., `"SF Mono"`, `"Menlo"`, `"Courier New"`; surrounding spaces are ignored)
- `messageFontDesign`: `String` (optional; one of: `"default"`, `"monospaced"`, `"serif"`, `"rounded"`)
- `messageFontSize`: `String` (optional; one of: `"small"`, `"body"`, `"large"`; scales the message font size set in Rete's settings — see *Font Configuration*)

`UI` (colours are `HexColor` objects; the badge, icon and Highlightr fields are strings, and
`bufferListNotifyNameBold` is a boolean):
- `background` — also what per-nick colours are made readable against; see
  "Unique nick colours and `ui.background`" below. reTerm keeps your terminal's own background
  and reads `background` only to blend translucent colours (see *How alpha renders*).
- `foreground`. On reTerm, the input line is a band of `foreground` at 20 % over `background`
  (10 % beside it), with its text in `foreground`.
- `secondary` (optional): a muted colour (the built-in palettes use grey) for text that should
  recede. It is the fallback for the `hostmask` segment (after `categories.nick`) and the
  `account` segment — see *How a segment's colour is resolved*. **On reTerm** it is also the
  colour of the timestamp column, the status bar, the nick list's user count, the buffer numbers
  in the buffer list, the input line's suggestion, the sender's mode prefix (`@`, `+`) when
  `senderPrefixColor` is not set, and segment braces when `segmentBraceColor` is not set. When
  omitted or unparseable, those uses fall back to `ui.foreground`. macOS and iOS draw braces and the
  mode prefix in the system's secondary label colour when their own keys are unset, not in
  `ui.secondary`.
- `accent`: channel names, the unread divider, and the tint of buttons and other controls on
  Rete.app and iOS.
- `divider`: the lines between parts of the window (on reTerm, the line between the buffer list
  and the chat). When it will not parse, the secondary colour at 20 % opacity.
- `topicBarBackground`: the bar above the chat with the channel's modes and topic (on reTerm, the
  header line above the transcript). When it will not parse, the secondary colour at 8 % opacity.
- `topicBarForeground`: the text on that bar: the topic, the topic editor, and the user-info bar
  (see *What every app draws the same way*). When it will not parse, `foreground`.
- `badgeBackground`: the capsule behind a nick list row's mode badge (`@`, `+`) and behind the
  channel's modes in the topic bar. When it will not parse, the accent at 15 % opacity. reTerm
  draws it as the badge cell's background, composited over `ui.background` (the channel's modes:
  over the topic bar).
- `badgeForeground`: the mode badge's text, and the channel's modes in the topic bar. When it will
  not parse, `accent`.
- `sidebarSelectionBackground`: the selected row in the buffer list, and in the log viewer's lists.
  When it will not parse, the accent at 15 % opacity.
- `sidebarSelectionForeground`: the selected row's name in the buffer list (and the log viewer), in place of the name's
  own colour (`ui.foreground`, `bufferListActiveBufferNameColor` or
  `bufferListHighlightBufferNameColor`); the name keeps its weight, bold when the buffer has
  something new. Every app draws it. When it will not parse, `accent`.
- `highlightRowBackground` (optional): Background color for highlighted message rows. Default: the
  system red at ~15% opacity (`#FF3B3026` in light, `#FF453A26` in dark). Not drawn in direct messages, the raw buffer or the script
  debug buffer. Every app fades it out over 30 seconds from when the line is first shown.
- `expandedGroupRowBackground` (optional): Background color for the member rows of an opened group of join/part lines; the group's headline is not painted. Default: the theme's `accent` at ~8% opacity. Every app draws it; in reTerm, click a folded line (`▸`) to open it (`▾`) and click again to close it.
- `operatorBadgeColor` (optional): `HexColor` for operator badge icon color. Default: `secondary` (then `foreground`). On reTerm, the IRCop flag in the header line and the `*` marker in the nick list.
- `awayBadgeColor` (optional): `HexColor` for away badge icon color. Default: `secondary` (then `foreground`). On reTerm, the Away flag in the header line (an away nick in reTerm's nick list is shown dimmed, with no marker).
- `secureBadgeColor` (optional): `HexColor` for secure connection badge icon color. Default: the system green (`#34C759` in light, `#30D158` in dark). On reTerm, the Secure flag in the header line.
- `botBadgeColor` (optional): `HexColor` for bot badge icon color. Default: `secondary` (then `foreground`). On reTerm, the Bot flag in the header line and the `b` marker in the nick list.
- `blockedBadgeColor` (optional): `HexColor` for blocked badge icon color. Default: the system red (`#FF3B30` in light, `#FF453A` in dark). On reTerm, the `x` marker in the nick list.
- `bufferListHighlightBadgeBackground` (optional): Capsule fill for the **highlight** count badge in the buffer list (sidebar, split pane headers, topic bar). Default: the system red at ~80% opacity. On reTerm, the highlight count's text colour (opacity ignored).
- `bufferListHighlightBadgeForeground` (optional): Text colour on that highlight badge. Default: white. Not used on reTerm.
- `bufferListUnreadBadgeBackground` (optional): Capsule fill for the **unread** count badge (shown when there are unreads and no highlight badge). Default: grey (`#8E8E93`) at ~60% opacity. On reTerm, the unread count's text colour (opacity ignored).
- `bufferListUnreadBadgeForeground` (optional): Text colour on the unread badge. Default: white. Not used on reTerm.
- `bufferListHighlightBufferNameColor` (optional): Colour for the buffer **title** in the sidebar (and related rows) when the buffer has a **highlight** (mention / rules / red badge state). When omitted, the title uses normal `foreground` (default: no separate tint, bold still follows `bufferListNotifyNameBold`).
- `bufferListActiveBufferNameColor` (optional): Colour for the buffer title when the row is **active** only in the sense of **unread** and/or **channel activity** (grey badge / activity strip), and not in the highlight state above. When omitted, uses normal `foreground`.
- `bufferListNotifyNameBold` (optional): Boolean; when `false`, buffer names do **not** switch to bold when emphasized. When `true` or omitted, emphasized names are bold (default: bold).
- `joinBadge` (optional): `String` (SF Symbol name for join message badge. Default: `"arrow.right.circle.fill"`)
- `partBadge` (optional): `String` (SF Symbol name for part message badge. Default: `"arrow.left.circle.fill"`)
- `quitBadge` (optional): `String` (SF Symbol name for quit message badge. Default: `"arrow.left.circle.fill"`)
- `actionBadge` (optional): `String` (SF Symbol name for action (/me) message badge. Default: `"star.fill"`)
- `noticeBadge` (optional): `String` (SF Symbol name for notice message badge. Default: `"exclamationmark.bubble.fill"`)
- `infoBadge` (optional): `String` (SF Symbol name for info message badge. Default: `"info.circle.fill"`)
- `topicBadge` (optional): `String` (SF Symbol name for topic message badge. Default: `"text.quote"`)
- `bufferListChannelIcon` (optional): `String` (SF Symbol for channel rows in the buffer list; omit or `null` for the default; `""` = no icon. Default: `"number"`)
- `bufferListDirectMessageIcon` (optional): `String` (SF Symbol for direct message / nick buffer rows; `""` = no icon. Default: `"person"`)
- `bufferListSystemIcon` (optional): `String` (SF Symbol for server / system buffer rows; `""` = no icon. Default: `"gearshape"`)
- `bufferListRawIcon` (optional): `String` (SF Symbol for the raw traffic buffer; `""` = no icon. Default: `"waveform.path"`)
- `bufferListWebIcon` (optional): `String` (SF Symbol for web content buffers; `""` = no icon. Default: `"globe"`)
- `bufferListScriptIcon` (optional): `String` (SF Symbol for script buffers; `""` = no icon. Default: `"terminal.fill"`)
- `bufferListScriptDebugIcon` (optional): `String` (SF Symbol for the script debug buffer; `""` = no icon. Default: `"terminal"`)
- `scriptEditorHighlightrTheme` (optional): `String` (Highlightr theme name for script editor syntax highlighting. Default: `"github"` in the light palette, `"github-dark"` in the dark one)

`MessageColors`:
- `categories`: `{ String: HexColor }`
  - Keys are the message categories (the primary message type classification):
    - `normal`, `serverNotice`, `serverReply`, `info`, `error`, `join`, `part`, `quit`, `topic`, `nick`, `notice`, `mode`, `kick`, `action`, `ctcp`, `invite`
  - One more key, `dateDivider`, colours the **date** shown where the day changes between messages. The line
    drawn beside the date keeps its own colour (`ui.divider`, faded). When `dateDivider` is not
    set, every app draws the date muted, in the secondary colour (the system's secondary label
    colour on macOS and iOS, `ui.secondary` on Rete for Linux and reTerm). A value that does not
    parse draws it in the text colour.
  - **Categories are the primary way to color messages** - each message has exactly one category
  - `error` also colours reTerm's own error messages; when it is not set, they are in `ui.foreground`.
- `tags`: `{ String: HexColor }`
  - Keys are message tags. A tag colour is used **only when the line's category
    has no colour in `categories`**, and only these eleven tags are consulted — the first one
    in this order that the theme colours wins:
    - `highlight`, `bold` (formatting)
    - `server`, `channel`, `dm` (context - where the message appears)
    - `raw`, `in`, `out`, `script` (technical metadata)
    - `do` (TCL command input/output in script debug buffer)
    - `silence` (silence/block related messages)
  - **Every other tag is ignored for colouring**, on every frontend — a key for one of these is accepted and does nothing: `notice`, `reply`, `error`, `info`, `join`, `part`, `quit`, `nick`, `topic`, `topicInfo`, `action`, `ctcp`, `mode`, `kick`.
  - Colour those messages by **category** instead: `categories.notice` / `serverNotice`,
    `serverReply` (numeric server replies), `error`, `info`, `join`, `part`, `quit`, `nick`,
    `topic`, `action`, `ctcp`, `mode`, `kick`.
- `segmentColors` (optional): `{ String: HexColor }`
  - Keys are segment kind names. Used to color specific parts of join/part/quit/topic/mode/kick/account/CTCP etc. messages, and boolean literals via the `true` / `false` segment kinds in settings-style rows.
  - Supported keys: `literal`, `channel`, `person`, `topic`, `quitmessage`, `partmessage`, `kickmessage`, `hostmask`, `account`, `timestamp`, `mode`, `modeParam`, `command`, `value`, `true`, `false` — one per segment kind; see *Segment kinds* below. A quit, part or kick **reason** is coloured by `quitmessage`, `partmessage` or `kickmessage`; there is no key named "reason". (`literal` and `timestamp` entries are accepted and ignored — see rule 1 under *How a segment's colour is resolved*.)
  - If omitted, semantic fallbacks keep parts distinct from the line — e.g. `ui.accent` for topic and channel, `categories.nick` then `ui.secondary` for hostmask, `ui.secondary` for account, the line's category for a quit/part/kick reason. The full table is under *How a segment's colour is resolved*.
- `segmentBold` (optional): `[ String ]`
  - List of segment kind names that should be rendered bold (e.g. `["person", "timestamp", "command"]`)
  - When omitted or empty, the app still bolds `person`, `timestamp`, and `command` by default
- `segmentBraceColor` (optional): `HexColor`
  - Color for parentheses and brackets around segments, e.g. `(reason)` and `[account]` in part/quit messages
  - When omitted, the secondary colour, so braces don’t contrast with the content: the system's secondary label colour on macOS and iOS, `ui.secondary` (then `ui.foreground`) on Rete for Linux and reTerm
- `senderPrefixColor` (optional): `HexColor`
  - Color for sender mode prefix symbols in the nickname column, e.g. `@nick`, `+nick`, `%nick` (the prefix glyph itself)
  - When omitted, the secondary colour: the system's secondary label colour on macOS and iOS, `ui.secondary` (then `ui.foreground`) on Rete for Linux and reTerm


`HexColor`:
- Object with a single field:
  - `hex`: `"#RRGGBB"`

- Two forms are accepted: `"#RRGGBB"` and `"#RRGGBBAA"`, where `AA` is alpha (`00` fully
  transparent, `FF` fully opaque). Case does not matter, and the leading `#` is optional —
  `"0A84FF"` and `"#0a84ff"` name the same colour as `"#0A84FF"`.
- **Three-digit shorthand (`"#ABC"`) is expanded to six** when the file is read: `"#0af"`
  becomes `"#00AAFF"`. It works with or without the `#`, and it works everywhere a `HexColor`
  appears. Rete saves the expanded form.
- Alpha is most useful for the keys that tint something behind them — `highlightRowBackground`,
  `expandedGroupRowBackground`, `sidebarSelectionBackground`, `badgeBackground`, `divider`.
  The built-in theme uses it for exactly those.
- **How alpha renders on each frontend.** Rete.app, iOS and Rete on Linux draw the colour
  translucently over whatever is behind it. A terminal cell has no alpha channel, so **reTerm
  composites the colour against the theme's own `ui.background`**, which it assumes is behind it,
  and draws the result — the same arithmetic. One theme file therefore looks the same on every
  app. A colour
  reTerm draws over its topic bar (the channel's modes badge) is composited over the bar's colour
  instead.
- **reTerm keeps text readable on what it paints.** Text on `topicBarBackground`,
  `sidebarSelectionBackground`, `highlightRowBackground`, a badge, or the input line whose
  contrast with that colour is below 3:1 is drawn in the bar's text colour, or in black or white,
  instead.

Example `HexColor`:
`{ "hex": "#0A84FF" }`

### Icon fields: an SF Symbol, and the Linux icon beside it

The nineteen icon fields named under *reTerm ignores the SF-Symbol fields* below are listed in
the schema as `String`. Each of them accepts **either** that string **or** an object:

```json
"operatorBadge": { "sfSymbol": "crown.fill", "linux": ["emblem-ok-symbolic", "starred-symbolic"] }
```

- **`sfSymbol`** (optional `String`) is the SF Symbol name, and means exactly what the plain
  string means for that field: `"sfSymbol": ""` is what `""` is (no icon), and a missing or `null` `sfSymbol` is what omitting the field is (the app's default).
- **`linux`** (optional array of `String`) names freedesktop icons from the desktop's icon
  theme, best first. Use the `-symbolic` names: a symbolic icon takes the badge's colour.
  Surrounding spaces are trimmed and empty names skipped.
- Any other key in the object is ignored. Anything that is neither a string nor this object
  (a number, an array, `"linux": "x"`) makes the theme fail to load, as a wrong type in any
  other field does.

What each frontend draws:

| Frontend | Draws |
|---|---|
| **Rete.app (macOS) and iOS** | `sfSymbol`, exactly as the plain string. `linux` is never read. |
| **Rete on Linux (the GTK app)** | the **first** `linux` name the desktop's icon theme has; if it has none of them, `sfSymbol`'s translation (the table under the badge fields above); then the field's default icon; and if the icon theme has none of those, a user badge's word, a message badge's character (`→`, `←`, `★`, `‼`, `ℹ`, `❝`, `•`), and no icon in the buffer list. |
| **reTerm** | nothing — it ignores both halves, as it ignores the string. |

So a theme that only needs to look right on macOS keeps writing strings, and a theme author
who wants a particular icon on Linux adds `linux` without changing what macOS draws.

**Rete.app's theme editor keeps the object form.** A field read as an object is written back as
an object — with its `linux` list — when the theme is saved, including when you change that
field's SF Symbol in the editor; a field read as a string is written back as a string. The
editor validates `sfSymbol` exactly as it validates the string (where it validates the string
at all — the seven `bufferList…Icon` fields), and does not check the `linux` names: they
are not SF Symbols, and a Mac has no freedesktop icon theme to check them against.

Rete on Linux draws all nineteen: the five user badges, the seven message badges (beside join,
part, quit, notice, info, topic and action lines, where a chat line shows its nick) and the
seven buffer-list icons. A message badge takes its line's colour, as on macOS; a buffer-list icon
is drawn in `ui.foreground`, faded. As on macOS, `""` in a user badge or message badge field
draws no badge and in a `bufferList…Icon` field no icon. The translation table knows Rete.app's defaults for all
of them — for example `arrow.right.circle.fill` → `go-next-symbolic`, `arrow.left.circle.fill` →
`go-previous-symbolic`, `exclamationmark.bubble.fill` → `emblem-important-symbolic`, `text.quote`
→ `format-justify-left-symbolic`, `number` → `system-users-symbolic`, `waveform.path` →
`network-transmit-receive-symbolic`, `terminal` and `terminal.fill` → `utilities-terminal-symbolic`.

### reTerm ignores the SF-Symbol fields and the script-editor theme

A terminal has no SF Symbols, so **reTerm never reads any of the nineteen SF-Symbol-valued
fields** — setting, changing or omitting one makes no difference to what reTerm draws. They are:

- on the theme: `operatorBadge`, `awayBadge`, `secureBadge`, `botBadge`, `blockedBadge`;
- on each palette's `ui`: `joinBadge`, `partBadge`, `quitBadge`, `actionBadge`, `noticeBadge`,
  `infoBadge`, `topicBadge`;
- on each palette's `ui`: `bufferListChannelIcon`, `bufferListDirectMessageIcon`,
  `bufferListSystemIcon`, `bufferListRawIcon`, `bufferListWebIcon`, `bufferListScriptIcon`,
  `bufferListScriptDebugIcon`.

The same is true of **`scriptEditorHighlightrTheme`**, which names a theme for Rete.app's
macOS script editor; reTerm has no script editor. The **colour** fields that go with the
badges (`operatorBadgeColor` and the rest) are not SF Symbols, and are covered by their own
entries above.

A theme that sets these fields is still valid for reTerm — they are read and kept, and a
script that asks for one with `::rete::theme_get` gets an answer on reTerm exactly as it does
in Rete.app (see `Scripts.md`). What they do not do on reTerm is change anything on screen.

## Fallback behavior

### A colour that will not parse is not the same as a colour you did not set

**Three cases, not two.** A hex value is checked before it is drawn, and a value
that cannot be drawn takes **the system default for Dark/Light** — not the fallback the key
would have had if you had left it out.

| The theme file | What is drawn |
|---|---|
| names no colour for the key | the fallback for that key, below |
| names a colour that **will not parse** | **the system default for Dark/Light** |
| names a valid colour | that colour |

This applies to the message colours — categories, tags and segment colours — and to
`ui.background`, `ui.foreground` and `ui.accent`. The point is that a typo should look like a
typo, not like a colour you left out.

- On `Rete.app` and iOS the system default is the system's adaptive label colour (for
  `ui.background` and `ui.accent`, the system window background and accent colour), following
  the current appearance.
- On Rete for Linux it is the same system colours, for the desktop's light or dark setting;
  a message colour that will not parse is drawn in the foreground.
- On `reTerm`, which always uses the `dark` palette, a message colour that will not parse is
  drawn in the theme's foreground, and an unparseable `ui.background`, `ui.foreground` or
  `ui.accent` is `#323232`, `#FFFFFF` or `#0A84FF`, the system's dark colours.

The other `ui` colours take a fallback of their own when they will not parse, the one their entry
in the schema names: the selection colours the accent at 15 % and the accent, a buffer-name colour
(`bufferListHighlightBufferNameColor`, `bufferListActiveBufferNameColor`) the text colour, and
`ui.secondary` the foreground.

**What counts as parseable is one rule on every frontend**, and it is the same for a colour
typed into a theme file and one set in the theme editor: `#RRGGBB` or `#RRGGBBAA`, exactly six
or eight hex digits, with or without the leading `#`. A three-digit value is expanded before
this check. Anything else — `#ABCDEFG`, `#FF0000ZZ`, `##FF0000`, `#FF`, `nope` — will not parse.

If a color is missing:
- UI background falls back to the system window background (macOS) / system background (iOS/iPadOS).
- Foreground falls back to the system label colour.
- Accent falls back to the system accent colour.
- Divider, topic bar, badge, sidebar selection, highlight-row, expanded-row and badge-icon colours fall back to the defaults given for each key in the schema above. Every app uses the same defaults. Where a default is a system colour (red, green), reTerm uses its dark-appearance value, as it uses the `dark` palette.

Message color resolution:
1. **Category color** is checked first (the line's category) - this is the primary classification
2. **Tag colors** are checked only for the eleven contextual tags, in this order: highlight, bold, server, channel, dm, raw, in, out, script, do, silence. Every other tag is ignored (see `tags` in the schema)
3. If neither category nor tag color is defined, the UI foreground color is used
4. Final fallback is the system label colour

**Important**: Categories and tags do not overlap. Categories handle message types (join, part, mode, etc.), while tags only provide contextual metadata (where the message appears, formatting, etc.).

What every app draws the same way, whatever the categories and tags say:

- **The raw traffic buffer's text is always `ui.foreground`.** `categories.info` and
  `tags.raw`, `tags.in` and `tags.out` do not colour it; they still colour the line's badge
  icon where an app draws one. Each line is shown exactly as it was sent or received: mIRC
  colours and formatting are not applied, and URLs, channel names and nicks in it are not picked
  out. A line longer than 24,000 characters is cut short, with a note saying so.
- **Timestamps** are `ui.foreground` at 65 % opacity.
- **URLs in messages** are drawn in the platform's link colour, underlined; no theme key sets
  it. On Rete for Linux that is the desktop GTK theme's link colour. reTerm draws a URL as
  ordinary text in the line's colour; your terminal may underline or link it itself. Channel
  names in messages are `ui.accent` and bold.
- **Nicks** — the sender, and a known nick inside a message — take the network's nick-colour
  rules first, then the unique nick colour when that setting is on, then `categories.nick`,
  then `ui.foreground`. The sender is not bold; a known nick inside a message is.
  `segmentColors.person` does not colour them: it colours a nick in a state message (join,
  part, quit, mode and the like).
- **The nick list** draws a nick in `ui.foreground` (at 60 % when the person is away), or in the
  network's nick-colour rule or the person's assigned colour.
- **A folded group's headline** is coloured section by section: joins and "Netsplit over" in the
  join colour, parts in the part colour, quits and netsplits in the quit colour, anything else
  in `ui.foreground`.
- **The unread divider**, label and rule, is `ui.accent` (reTerm draws none).
- **The topic bar's topic** is `topicBarForeground`, with channel names in `ui.accent` (keeping
  their bold, italic and underline) and URLs in the link colour, as in messages; both are
  clickable. With mIRC colours turned on, the topic's mIRC colours, bold, italic and underline are
  drawn; its mIRC background colours are not. reTerm draws a URL in the topic unstyled.
  **"No topic set"** is `ui.foreground` at half strength (reTerm shows nothing).
- **The user-info bar** — your own details in the server window, the other person's in a query —
  is `topicBarForeground` at half strength on the topic bar's background.
- **Selected text** is the system's selection colour (on Linux, the desktop's); no theme key
  sets it.
- **Link previews** (Rete.app and Rete on Linux) take no theme colours.

## Segment-based message styling

State messages (join, part, quit, topic, mode, kick, account, CTCP, nick change, etc.) are built from **segments**: each part (nick, channel, reason, topic text, user@host, account, timestamp, command name, etc.) is tagged with a **segment kind**. The theme can then color and bold those parts independently via `segmentColors` and `segmentBold`, and style the parentheses/brackets around them via `segmentBraceColor`.

### Segment kinds

| Key | Used for |
|-----|----------|
| `literal` | Plain text (e.g. " has joined ", " has left ") |
| `channel` | Channel names (e.g. #foo); also styled by accent for clickability |
| `person` | A nick in a state message (join, part, quit, mode, …); bold by default |
| `topic` | Topic text (e.g. after "Topic for #ch: ") |
| `quitmessage` | Quit reason text (default e.g. burgundy on red quit line) |
| `partmessage` | Part reason inside `( )` (default colour complements part line) |
| `kickmessage` | Kick reason inside `( )` (default colour complements kick line) |
| `hostmask` | user@host inside `( )` in join/part/quit |
| `account` | Account name inside `[ ]` |
| `timestamp` | Dates/times (e.g. "channel created on", "topic set on") |
| `mode`, `modeParam` | Mode string and parameters in mode messages |
| `command` | CTCP command and "request"/"reply" in CTCP messages |
| `value` | Other values (e.g. batch params in `( )`) |
| `true` | Boolean `true` values in settings/status style rows |
| `false` | Boolean `false` values in settings/status style rows |

### How a segment's colour is resolved

`segmentColors` and `categories` are **not independent**. A segment with no colour of its
own borrows a *category* colour, so setting `categories.join` also changes the colour of
channel names in state messages unless `segmentColors.channel` says otherwise. The order,
top to bottom, first match wins:

1. **`literal` and `timestamp` always take the line's category colour**, and a
   `segmentColors.literal` / `segmentColors.timestamp` entry is ignored. These are the glue
   text between segments; colouring them separately makes every state message a ransom note.
2. **`topic` takes `ui.accent`** when `segmentColors.topic` is absent, so topic text stands
   out from the "Topic for #ch: " it follows. Set `segmentColors.topic` to override it.
   **Absent means absent**: a `segmentColors.topic` that will not parse is a colour you set,
   so it takes the system default rather than the accent (see *Fallback behavior*).
2b. **`channel` takes `ui.accent`** on the same terms, and is **always bold** — see the note
   under the borrowing table.
3. **`segmentColors[<kind>]`**, if the theme sets it.
4. **The semantic fallback for that kind**, from the table below.
5. **The line's category colour** (what a message of this category is coloured), and
   finally `ui.foreground`.

| Segment kind | Borrows, in order |
|---|---|
| `person` | `categories.nick` → `ui.foreground` |
| `channel` | — (rule 2b always applies first) |
| `hostmask` | `categories.nick` → `ui.secondary` → `ui.foreground` |
| `account` | `ui.secondary` → `ui.foreground` |
| `quitmessage` | `categories.quit` → `ui.foreground` |
| `partmessage` | `categories.part` → `ui.foreground` |
| `kickmessage` | `categories.kick` → `ui.foreground` |
| `mode`, `modeParam` | `categories.mode` → `ui.foreground` |
| `command` | `categories.ctcp` → `ui.foreground` |
| `value` | `categories.info` → `ui.foreground` |
| `true`, `false` | `ui.foreground` |
| `topic` | — (rule 2 always applies first) |
| `literal`, `timestamp` | — (rule 1 always applies first) |

> **`channel` works like `topic`, and is always bold.** It takes `ui.accent` when
> `segmentColors.channel` is absent and that colour when it is set — on every app.
> **Bold is not negotiable**: the accent marks a channel name as *clickable*, so `segmentBold`
> cannot switch it off. A theme may recolour a channel name; it may not make it stop reading as
> a control. The built-in themes set no `segmentColors.channel`, so their channel names follow
> `ui.accent`.

### Which segments are bold

`segmentBold` is a **replacement, not an addition**, and an omitted key is not the same as
an empty list only in the sense that JSON cannot tell them apart:

- **Omitted** — `person`, `timestamp` and `command` are bold.
- **`"segmentBold": []`** — the same three are bold. An absent key and an empty array decode
  identically, so **there is no way for a theme to say "bold nothing"**.
- **`"segmentBold": ["channel"]`** — `channel` is bold and `person`, `timestamp` and
  `command` are **not**. Naming any kind replaces the default set rather than adding to it,
  so a theme that wants a fourth bold kind must list all four.

### Brace color

`segmentBraceColor` applies to the characters `(`, `)`, `[`, `]` that wrap segments (e.g. reason in parentheses, account in brackets). Default is a subtle grey so braces don’t contrast with the content; you can set it to match your palette (e.g. same as divider or a muted foreground).

### Example: custom segment and brace colors

```json
{
  "messageColors": {
    "categories": { "join": { "hex": "#34C759" }, "part": { "hex": "#FF9F0A" }, ... },
    "tags": { ... },
    "segmentColors": {
      "topic": { "hex": "#007AFF" },
      "quitmessage": { "hex": "#800020" },
      "partmessage": { "hex": "#8B6914" },
      "kickmessage": { "hex": "#8B2500" },
      "hostmask": { "hex": "#6B7280" },
      "account": { "hex": "#00CED1" }
    },
    "segmentBold": ["person", "timestamp", "command"],
    "segmentBraceColor": { "hex": "#8E8E93" },
    "senderPrefixColor": { "hex": "#8E8E93" }
  }
}
```

If you omit `segmentColors`, `segmentBold`, `segmentBraceColor`, or `senderPrefixColor`, the app uses semantic defaults so state messages and sender prefixes still look distinct and readable.

## Appearance mode

A theme does not choose between its light and dark palettes; the app does. On macOS and iOS,
Rete's appearance setting picks the palette:
- Automatic: follows the system appearance
- Light: uses the theme's `light` palette
- Dark: uses the theme's `dark` palette

Rete on Linux follows the desktop's light or dark setting, and reTerm always uses `dark`. The theme
must define both light and dark palettes. A `preferDark` field in a theme file is ignored.

## Example: Minimal valid theme

This is a tiny example that uses a light gray background, dark text, and a blue accent. It sets only a few message colors (others fall back to foreground).

```json
{
  "name": "Minimal Example",
  "description": "A simple light theme with blue accents.",
  "author": "You",
  "version": "1.0",
  "operatorBadge": "shield.checkered",
  "awayBadge": "moon.fill",
  "secureBadge": "lock.fill",
  "botBadge": "cpu",
  "blockedBadge": "hand.raised.fill",
  "light": {
    "ui": {
      "background": { "hex": "#FFFFFF" },
      "foreground": { "hex": "#000000" },
      "accent": { "hex": "#0A84FF" },
      "divider": { "hex": "#D1D1D6" },
      "topicBarBackground": { "hex": "#F2F2F7" },
      "topicBarForeground": { "hex": "#000000" },
      "badgeBackground": { "hex": "#D6E8FF" },
      "badgeForeground": { "hex": "#0A84FF" },
      "sidebarSelectionBackground": { "hex": "#D6E8FF" },
      "sidebarSelectionForeground": { "hex": "#0A84FF" },
      "highlightRowBackground": { "hex": "#FF000026" },
      "expandedGroupRowBackground": { "hex": "#0A84FF14" },
      "operatorBadgeColor": { "hex": "#6B7280" },
      "awayBadgeColor": { "hex": "#6B7280" },
      "secureBadgeColor": { "hex": "#34C759" },
      "botBadgeColor": { "hex": "#6B7280" },
      "blockedBadgeColor": { "hex": "#FF3B30" },
      "scriptEditorHighlightrTheme": "github",
      "joinBadge": "arrow.right.circle.fill",
      "partBadge": "arrow.left.circle.fill",
      "quitBadge": "arrow.left.circle.fill",
      "actionBadge": "star.fill",
      "noticeBadge": "exclamationmark.bubble.fill",
      "infoBadge": "info.circle.fill",
      "topicBadge": "text.quote"
    },
    "messageColors": {
      "categories": {
        "normal": { "hex": "#000000" },
        "error": { "hex": "#FF3B30" }
      },
      "tags": {
        "do": { "hex": "#000000" }
      }
    }
  },
  "dark": {
    "ui": {
      "background": { "hex": "#000000" },
      "foreground": { "hex": "#FFFFFF" },
      "accent": { "hex": "#0A84FF" },
      "divider": { "hex": "#3C3C43" },
      "topicBarBackground": { "hex": "#1C1C1E" },
      "topicBarForeground": { "hex": "#FFFFFF" },
      "badgeBackground": { "hex": "#1B2A40" },
      "badgeForeground": { "hex": "#0A84FF" },
      "sidebarSelectionBackground": { "hex": "#1B2A40" },
      "sidebarSelectionForeground": { "hex": "#0A84FF" },
      "highlightRowBackground": { "hex": "#FF000026" },
      "expandedGroupRowBackground": { "hex": "#0A84FF14" },
      "operatorBadgeColor": { "hex": "#9CA3AF" },
      "awayBadgeColor": { "hex": "#9CA3AF" },
      "secureBadgeColor": { "hex": "#30D158" },
      "botBadgeColor": { "hex": "#9CA3AF" },
      "blockedBadgeColor": { "hex": "#FF453A" },
      "scriptEditorHighlightrTheme": "github-dark",
      "joinBadge": "arrow.right.circle.fill",
      "partBadge": "arrow.left.circle.fill",
      "quitBadge": "arrow.left.circle.fill",
      "actionBadge": "star.fill",
      "noticeBadge": "exclamationmark.bubble.fill",
      "infoBadge": "info.circle.fill",
      "topicBadge": "text.quote"
    },
    "messageColors": {
      "categories": {
        "normal": { "hex": "#FFFFFF" },
        "error": { "hex": "#FF453A" },
        "notice": { "hex": "#FF6B81" },
        "ctcp": { "hex": "#FF453A" }
      },
      "tags": {
        "do": { "hex": "#FFFFFF" }
      }
    }
  }
}
```

## Troubleshooting

- Theme not appearing in the picker
  - Ensure the file extension is .json.
  - Place it in the themes folder for the program you are running — see **Where to put theme
    files** above. It is not the same folder for `Rete.app` and `reTerm`.
  - Validate JSON syntax and schema. **Rete tells you when a .json file in that folder could
    not be read**, and names it — one message per load covering every file that failed, not one
    per file. A file that fails is skipped; the themes that do decode still load, and so does
    the built-in `System` theme.
  - The debug log carries the decoding error in full — the field and its path in the file,
    which the message does not — so it is the place to look for *why* a file failed rather than
    *which*.
  - A file with no `.json` extension is not a candidate at all and is not reported; it is not a
    theme file as far as Rete is concerned.
- Decoding error mentioning “Expected to decode Dictionary<String, Any> but found a string”
  - You likely used a plain string for a color. Use an object: "background": { "hex": "#RRGGBB" }.
- Alpha/transparency in hex
  - Supported: use `"#RRGGBBAA"`. Rete.app, iOS and Rete on Linux draw it translucently; reTerm
    composites it against the theme's `ui.background` and draws the result, so every app agrees.
  - A colour looks right in the app and wrong in reTerm
    - Check whether it is drawn over something other than `ui.background` — a badge inside
      the topic bar, for instance. reTerm composites against the background, so a tint over a
      different backdrop is an approximation there.
- A colour is ignored entirely
  - Check the digit count. After the leading `#` is stripped and a three-digit shorthand is
    expanded, the value must be **exactly six or eight** hex digits. `"#ABCDEFG"` (seven),
    `"#FF"` (two) and `"nope"` fall back to the system default for Dark/Light with no error.
  - Three-digit shorthand such as `"#ABC"` is **not** one of these cases — it is expanded to
    `"#AABBCC"` when the file is read.

## Unique nick colours and `ui.background`

With **unique nick colours** turned on, a nickname's colour is derived from the nickname itself
rather than from the theme — so it is not a field you set. It *is* affected by one field you do
set: the colour is adjusted until it reaches a **3:1 contrast ratio against `ui.background`**
(WCAG AA for large text; nicknames are short and bold).

**Any `ui.background` works**, a mid-grey one included: the adjustment tries both lightening and
darkening the colour and keeps the better.

Two consequences worth knowing:

- **The same nickname is a different colour in your light and dark palettes**, deliberately —
  each is corrected against its own `ui.background`.
- **macOS and iOS draw the same nickname the same colour.**

## Badge Icons

Themes can customize the SF Symbol icons used for status badges in the nicklist and user info tooltips:

- **operatorBadge**: SF Symbol name for IRC operator badge (default: "shield.checkered")
- **awayBadge**: SF Symbol name for away user badge (default: "moon.fill")
- **secureBadge**: SF Symbol name for secure connection badge (default: "lock.fill")
 - **botBadge**: SF Symbol name for bot user badge (default: "cpu")
- **blockedBadge**: SF Symbol name for blocked user badge (default: "hand.raised.fill")

These badges appear next to nicknames in the nick list (with the setting `show_nicklist_badges`, on by default: the operator, away, bot and blocked badges) and in the header of the user info card. On iOS, which has no hover card, the WHOIS sheet's header shows all five. reTerm draws no icons: its nick list marks operator `*`, bot `b` and blocked `x` after the nick, in the badge colours, under the same setting; an away nick is shown dimmed, with no marker.

Example badge customization:
```json
{
  "name": "Custom Badges",
  "operatorBadge": "star.fill",
  "awayBadge": "zzz",
  "secureBadge": "lock.shield.fill",
  "botBadge": "cpu",
  "blockedBadge": "hand.raised.fill",
  "light": { ... },
  "dark": { ... }
}
```

Common SF Symbol alternatives:
- Operator: "star.fill", "crown.fill", "badge.shield.checkered", "checkmark.seal.fill"
- Away: "moon.fill", "moon.stars.fill", "zzz", "bed.double.fill"
- Secure: "lock.fill", "lock.shield.fill", "checkmark.shield.fill", "lock.circle.fill"
- Blocked: "hand.raised.fill", "hand.raised.slash.fill", "xmark.circle.fill", "nosign"

If not specified, the default icons are used; a badge set to `""` draws no icon, on every app that
draws badge icons. The badge icons apply to both light and dark variants.

## Message Badge Symbols

Themes can customize the SF Symbol icons used for message type badges in the sender column. These badges appear instead of nicknames for non-PRIVMSG messages (joins, parts, quits, actions, notices, info, and topic messages).

The following optional fields can be set in the `ui` object of each palette (light/dark):

- **joinBadge**: SF Symbol name for join messages (default: "arrow.right.circle.fill")
- **partBadge**: SF Symbol name for part messages (default: "arrow.left.circle.fill")
- **quitBadge**: SF Symbol name for quit messages (default: "arrow.left.circle.fill")
- **actionBadge**: SF Symbol name for action (/me) messages (default: "star.fill")
- **noticeBadge**: SF Symbol name for notice messages (default: "exclamationmark.bubble.fill")
- **infoBadge**: SF Symbol name for info messages (default: "info.circle.fill")
- **topicBadge**: SF Symbol name for topic messages (default: "text.quote")

Example message badge customization:
```json
{
  "name": "Custom Message Badges",
  "light": {
    "ui": {
      "joinBadge": "arrow.right.circle.fill",
      "partBadge": "arrow.left.circle.fill",
      "quitBadge": "arrow.left.circle.fill",
      "actionBadge": "star.fill",
      "noticeBadge": "exclamationmark.bubble.fill",
      "infoBadge": "info.circle.fill",
      "topicBadge": "text.quote",
      ...
    },
    ...
  },
  "dark": {
    "ui": {
      "joinBadge": "arrow.right.circle.fill",
      "partBadge": "arrow.left.circle.fill",
      "quitBadge": "arrow.left.circle.fill",
      "actionBadge": "star.fill",
      "noticeBadge": "exclamationmark.bubble.fill",
      "infoBadge": "info.circle.fill",
      "topicBadge": "text.quote",
      ...
    },
    ...
  }
}
```

Common SF Symbol alternatives:
- Join: "arrow.right.circle.fill", "arrow.down.circle.fill", "person.badge.plus", "arrow.turn.down.right"
- Part/Quit: "arrow.left.circle.fill", "arrow.up.circle.fill", "arrow.turn.up.left", "person.badge.minus"
- Action: "star.fill", "sparkles", "star.circle.fill", "star.square.fill"
- Notice: "exclamationmark.bubble.fill", "bell.fill", "exclamationmark.triangle.fill", "info.bubble.fill"
- Info: "info.circle.fill", "info.square.fill", "info", "circle.fill"
- Topic: "text.quote", "quote.bubble.fill", "text.bubble.fill", "doc.text.fill"

If not specified, the default icons are used. Badge symbols can be customized independently for light and dark variants.

## Buffer list (sidebar) icons

Themes can customize the SF Symbol shown to the left of each buffer in the **sidebar buffer list** (and the same icon is used in a few related places: split-view pane headers, the split layout picker, and the log viewer’s buffer list). This is separate from **message** badges (`joinBadge`, etc.) and from **nicklist** badges (`operatorBadge`, etc.), which live at the top level of the theme or only affect message rows.

The following optional fields live in the `ui` object of **each** palette (`light` and `dark`):

| JSON key | Buffer kind | Built-in default (when omitted or `null`) |
|----------|-------------|---------------------------------------------|
| `bufferListChannelIcon` | Channel | `number` |
| `bufferListDirectMessageIcon` | Direct message | `person` |
| `bufferListSystemIcon` | Server / system | `gearshape` |
| `bufferListRawIcon` | Raw traffic | `waveform.path` |
| `bufferListWebIcon` | Web | `globe` |
| `bufferListScriptIcon` | Script | `terminal.fill` |
| `bufferListScriptDebugIcon` | Script debug | `terminal` |

### Semantics

- **Key omitted or value `null`**: use the built-in default symbol for that buffer kind (see table).
- **Value `""` (empty string)**: do not show a leading icon for that kind when using this palette (row shows only the buffer name).
- **Any other non-empty string**: must be a valid SF Symbol name for the current OS.

Typing the **exact** default symbol name (e.g. `"number"` for channels) in the theme editor is treated like “use built-in default” and is stored as `null` in JSON, same idea as message badge fields.

Per-buffer **custom** icons set from Tcl (`setbuffericon`) override the theme for that buffer.

## Buffer list activity, highlights, and count badges

When a buffer has **highlights**, a count badge is shown (a red capsule by default). When it has **unreads** (or suppressed highlights) but no highlight tally, a grey capsule shows the unread count. **reTerm** shows the same counts at the right-hand end of buffer-list rows as coloured text with no fill: the count takes the badge's background colour (without its opacity), and the badge foreground is not used.

Optional `ui` fields (per palette):

| JSON key | Purpose | Default when omitted |
|----------|---------|----------------------|
| `bufferListHighlightBadgeBackground` | Capsule fill behind the highlight count | System red, ~80% opacity |
| `bufferListHighlightBadgeForeground` | Text (and icon) colour on the highlight badge | White |
| `bufferListUnreadBadgeBackground` | Capsule fill behind the unread count | Grey (`#8E8E93`), ~60% opacity |
| `bufferListUnreadBadgeForeground` | Unread count text colour | White |
| `bufferListHighlightBufferNameColor` | Buffer **title** colour when the row has a **highlight** (red highlight badge) | Normal `foreground` |
| `bufferListActiveBufferNameColor` | Buffer **title** colour when the row has **unread** and/or **channel activity** but not a highlight tally | Normal `foreground` |
| `bufferListNotifyNameBold` | Whether the buffer name becomes **bold** while emphasized | `true` (bold) |

**reTerm** reads these keys from the theme file you have selected, not from a built-in
palette. It loads the same themes directory the apps do; `/set theme <name>` switches,
and `/set theme reload` re-reads the directory after you edit a file.

**Buffers out of view.** When a buffer with highlights, or a direct message with something
unread, is scrolled out of the buffer list, every app shows a marker at that edge. A highlight
marker uses `bufferListHighlightBadgeBackground` and `bufferListHighlightBadgeForeground`; a
direct-message marker is `ui.accent` with white text.

On iOS, the buffer drawer draws the seven `bufferList…Icon` symbols and the buffer names by the
same rules as the macOS sidebar, and the selected buffer's name in `sidebarSelectionForeground`.
Both side drawers are on `ui.background`, and the buffer drawer's names are in the theme's font.

**reTerm renders the `dark` palette unconditionally** — it has no appearance mode, and a
theme's `light` palette is never consulted there, so a key you set only under `light`
has no effect in reTerm. This is true of the whole theme, not just the fields in this
section.

### Example

Hide channel icons in dark mode only; keep defaults elsewhere:

```json
{
  "name": "Example",
  "light": {
    "ui": {
      "background": { "hex": "#FFFFFF" },
      "foreground": { "hex": "#000000" },
      "accent": { "hex": "#0A84FF" },
      "divider": { "hex": "#3C3C434A" },
      "topicBarBackground": { "hex": "#F2F2F7" },
      "topicBarForeground": { "hex": "#000000" },
      "badgeBackground": { "hex": "#0A84FF1F" },
      "badgeForeground": { "hex": "#0A84FF" },
      "sidebarSelectionBackground": { "hex": "#0A84FF1F" },
      "sidebarSelectionForeground": { "hex": "#0A84FF" }
    },
    "messageColors": { "categories": {}, "tags": {} }
  },
  "dark": {
    "ui": {
      "background": { "hex": "#000000" },
      "foreground": { "hex": "#FFFFFF" },
      "accent": { "hex": "#0A84FF" },
      "divider": { "hex": "#3C3C434A" },
      "topicBarBackground": { "hex": "#1C1C1E" },
      "topicBarForeground": { "hex": "#FFFFFF" },
      "badgeBackground": { "hex": "#0A84FF1F" },
      "badgeForeground": { "hex": "#0A84FF" },
      "sidebarSelectionBackground": { "hex": "#0A84FF33" },
      "sidebarSelectionForeground": { "hex": "#0A84FF" },
      "bufferListChannelIcon": ""
    },
    "messageColors": { "categories": {}, "tags": {} }
  }
}
```

## Font Configuration

Themes can specify fonts for message content to customize the appearance and enable IRC script alignment compatibility. The font configuration applies consistently throughout the app to all message content (messages, topics, raw buffer, notices).

**reTerm ignores `fonts`.** A terminal draws every character in the terminal's own font and size, so in reTerm you choose the font in your terminal's settings.

### Font Structure

The optional `fonts` object can be added to any `ThemePalette` (light or dark variant):

```json
{
  "fonts": {
    "messageFontName": "SF Mono",
    "messageFontDesign": "monospaced",
    "messageFontSize": "body"
  }
}
```

### Font Fields

- **messageFontName** (optional): String
  - Custom font name to use (e.g., "SF Mono", "Menlo", "Courier New", "Monaco")
  - Must be a font name available on the system
  - If the font is not found, the system font with the specified design will be used instead
  - Leave empty or omit to use system fonts only

- **messageFontDesign** (optional): String
  - Font design variant: `"default"`, `"monospaced"`, `"serif"`, or `"rounded"`
  - Default: `"default"`
  - Use `"monospaced"` for IRC script alignment compatibility (fixed-width characters)
  - If `messageFontName` is specified and found, this design parameter is ignored

- **messageFontSize** (optional): String
  - Font size: `"small"`, `"body"`, or `"large"`
  - Default: `"body"`
  - Scales the **Message font size** you set in Rete's settings (10–24 pt): `small` is 90% of it,
    `body` is that size, and `large` is 110% of it. The theme does not set a point size of its own.

### Font Resolution

1. If `messageFontName` is specified and the font exists on the system, use that custom font
2. Otherwise, use system font with the specified `messageFontDesign`
3. If no font configuration is provided, use the system font

In every case the size is the **Message font size** setting, scaled by `messageFontSize` when the palette sets one. The smaller text beside messages, such as timestamps, is scaled by the same factor.

**On Rete for Linux**, as on macOS, a `messageFontName` that is not installed falls back to `messageFontDesign`. `"monospaced"` is the system's monospace font, `"serif"` its serif font, and `"default"` and `"rounded"` the desktop's font.

**On iOS**, the nick list, the buffer drawer, the topic and the sender's nick use the theme's font name or design too, at iOS's compact sizes.

### Examples

**Monospace font for IRC alignment:**
```json
{
  "light": {
    "fonts": {
      "messageFontDesign": "monospaced"
    },
    ...
  },
  "dark": {
    "fonts": {
      "messageFontDesign": "monospaced"
    },
    ...
  }
}
```

**Custom font with specific design:**
```json
{
  "light": {
    "fonts": {
      "messageFontName": "SF Mono",
      "messageFontSize": "body"
    },
    ...
  },
  "dark": {
    "fonts": {
      "messageFontName": "SF Mono",
      "messageFontSize": "body"
    },
    ...
  }
}
```

**Design-only (no custom font name):**
```json
{
  "light": {
    "fonts": {
      "messageFontDesign": "monospaced",
      "messageFontSize": "small"
    },
    ...
  }
}
```

### Notes

- Font configuration is optional. Themes without `fonts` will use the default system font
- Font configuration can be specified independently for light and dark variants
- The font applies to all message content: message text, the topic bar's topic, the raw buffer, and notices
- UI elements (timestamps, badges, buttons) continue to use their standard fonts
- For IRC script alignment, use `"monospaced"` design or a custom monospace font name

## Script Editor Syntax Highlighting

Themes can customize the syntax highlighting theme used in the script editor (TCL script editor). This uses the Highlightr library, which supports many popular code editor themes.

### Configuration

The optional `scriptEditorHighlightrTheme` field can be set in the `ui` object of each palette (light/dark):

```json
{
  "light": {
    "ui": {
      "scriptEditorHighlightrTheme": "github",
      ...
    },
    ...
  },
  "dark": {
    "ui": {
      "scriptEditorHighlightrTheme": "github-dark",
      ...
    },
    ...
  }
}
```

### Which theme is used, on macOS and on Linux

The name is looked up among Highlightr's bundled stylesheets (Highlightr 2.3.0 has 271). Rete
on macOS and Rete on Linux read the **same** stylesheets, so a name chooses the same theme on both:

- no `scriptEditorHighlightrTheme` in the palette: `github` in light, `github-dark` in dark;
- an empty string: `github`, in light and dark alike;
- a name Highlightr has no stylesheet for: the editor keeps the theme it was already showing —
  for an editor opened with such a name, `pojoaque` (Highlightr's own initial theme);
- a stylesheet Highlightr bundles but cannot load — **`far`** — is refused on both: the editor
  keeps the theme it was already showing, or uses `github` when it was showing none (see the notes
  below).

The highlighting itself is highlight.js 11.11.1's `tcl` grammar for `.tcl` files and its `json`
grammar for `.json` files, on both. A stylesheet's colours are read the way Highlightr reads
them, which is not always what the CSS means in a browser: in `github`, for instance, strings and
numbers are drawn in the plain text colour. Where Highlightr itself is not consistent between
launches (two rules of a stylesheet setting the same class, merged in an order that varies), Rete
on Linux always takes the rule that names the class on its own.

### Available Highlightr Themes

Common themes available in Highlightr include:

**Light themes:**
- `github` (default for light mode)
- `xcode`
- `default`
- `atom-one-light`
- `vs`
- `monokai-sublime` (light variant)

**Dark themes:**
- `github-dark` (default for dark mode)
- `xcode-dark`
- `atom-one-dark`
- `monokai`
- `dracula`
- `vs2015`
- `tomorrow-night`

The complete list of available themes depends on the Highlightr library version. Common themes that work well with TCL syntax include:
- `github` / `github-dark` - Clean, readable (default)
- `xcode` / `xcode-dark` - Matches Xcode's editor
- `monokai` - Popular dark theme
- `atom-one-dark` / `atom-one-light` - Atom editor style
- `dracula` - Popular dark theme with vibrant colors

### Examples

**Using different themes for light and dark:**

```json
{
  "light": {
    "ui": {
      "scriptEditorHighlightrTheme": "github",
      ...
    }
  },
  "dark": {
    "ui": {
      "scriptEditorHighlightrTheme": "monokai",
      ...
    }
  }
}
```

**Using the same theme for both:**

```json
{
  "light": {
    "ui": {
      "scriptEditorHighlightrTheme": "xcode",
      ...
    }
  },
  "dark": {
    "ui": {
      "scriptEditorHighlightrTheme": "xcode-dark",
      ...
    }
  }
}
```

### Notes

- If `scriptEditorHighlightrTheme` is not specified, the editor uses `"github"` for light mode and `"github-dark"` for dark mode
- A name Highlightr does not bundle changes nothing: the editor keeps the theme it already has
- A theme Highlightr bundles but cannot load is refused, with a warning in the log, and the editor keeps
  the theme it already has — or uses `"github"` when it has none yet. Of the bundled themes, that is
  **`far`** (its editor background is the named colour `navy`; Highlightr reads only `#` colours,
  `white` and `black` there)
- Each palette (light/dark) can specify its own Highlightr theme independently
- The syntax highlighting automatically adapts to theme changes when switching between light and dark modes

## Versioning

- A theme may carry a `version` string. Rete does not interpret it; use it to track your theme's iterations.
