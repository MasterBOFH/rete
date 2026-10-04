---
layout: doc
title: Themes Tutorial
permalink: /docs/themes/
---
Rete supports custom themes defined as JSON files. Themes provide UI colors and message colors for both Light and Dark variants. Users can import themes from Preferences or by placing files in the Themes folder.

## Changelog

### 2026-08-19

- **A colour that will not parse now takes the system default for Dark/Light**, on every
  themed key, rather than falling through to whatever the fallback chain had next. Three cases
  instead of two — see *Fallback behavior*. A typo in a theme file should look like a typo.
- **What counts as a parseable colour is now the same on all three frontends**: exactly six or
  eight hex digits. reTerm previously accepted six-or-more and used the first six, so
  `#ABCDEFG` drew there and fell back on macOS.
- **`segmentColors.channel` works.** It had no effect on any frontend before today; each
  renderer drew `ui.accent` in its own view code. It now behaves exactly like
  `segmentColors.topic` — accent when unset, your colour when set — and a channel name is
  **always bold**, because the styling marks it as clickable.
- **The built-in themes no longer set `segmentColors.channel`.** They carried one that was
  never drawn, whose value was the same green as `categories.join`; honouring it would have
  made channel names vanish into join lines. With it removed the accent applies, so the
  built-in themes look exactly as they did — and channel names now follow `ui.accent` if you
  change it.

### 2026-08-16

- **Four rules that were only in the code are now written down. No behaviour changed** —
  these describe what Rete has always done, and every one of them was found by a test
  written against the code rather than against this document.
  - **`segmentColors` and `categories` are not independent.** A segment with no colour of
    its own borrows a category colour — `person` borrows `categories.nick`, `channel`
    borrows `categories.join`, `command` borrows `categories.ctcp`, and so on. The full
    order and the borrowing table are under *How a segment's colour is resolved*.
  - **`"segmentBold": []` means `person`, `timestamp` and `command`**, not "bold nothing":
    an absent key and an empty array are the same JSON to the decoder. There is currently no
    way for a theme to switch every bold off.
  - **A non-empty `segmentBold` replaces the defaults rather than adding to them.**
    `["channel"]` bolds channel names *and un-bolds* nicks, timestamps and command names.
  - **`topic` takes `ui.accent`** when `segmentColors.topic` is absent, so topic text stands
    out from the label before it. `segmentColors.topic` overrides it.
  - ~~Also noted, because it surprises theme authors: **`segmentColors.channel` has no effect
    today** — both frontends draw a channel name in `ui.accent` and bold, for clickability,
    without consulting it.~~ **Fixed 2026-08-19**: it works on all three frontends now, on the
    same terms as `topic`, and a channel name is always bold. See the entry for that date.

### 2026-08-15

- **Alpha in hex colours: documentation corrected, and reTerm now honours it.**
  - This document previously said, in two places, that alpha was not supported and that only
    6-digit hex should be used. **That was wrong when it was written.** The SwiftUI colour
    parser has always accepted `"#RRGGBBAA"` and applied the alpha, the built-in theme uses it
    for five colours, and this document's own default values for `highlightRowBackground`
    (`#FF000026`) and `expandedGroupRowBackground` (`#0A84FF14`) are 8-digit. No theme
    behaviour changed on macOS or iOS; the description of it did.
  - **reTerm changed.** It previously used a translucent colour's opaque base and discarded
    the alpha, so the built-in theme's 15%-red highlight row was drawn as full red. It now
    composites against the theme's `ui.background`, so the same theme file renders alike on
    all three frontends. A theme with no alpha is unaffected — its output is byte-identical.

### 2026-05-04

- **Buffer list unread / highlight badges and row labels**
  - Optional `ui` colours: `bufferListHighlightBadgeBackground`, `bufferListHighlightBadgeForeground`, `bufferListUnreadBadgeBackground`, `bufferListUnreadBadgeForeground` (capsule badges in the macOS/iOS sidebar, split headers, topic bar, and reTerm sidebar counts)
  - Optional buffer **title** colours (separate highlight vs active): `bufferListHighlightBufferNameColor` (mention / highlight tally), `bufferListActiveBufferNameColor` (unread count and/or channel activity, when not in the highlight state). Replaces the earlier single `bufferListNotifyNameForeground` key — update hand-written JSON accordingly.
  - Optional `bufferListNotifyNameBold`: when JSON `false`, buffer names stay regular weight even with emphasis; when omitted or `true`, names use bold (default: bold, no name tint)
  - **Theme editor**: controls under **Buffer list (sidebar)** → “Buffer list: highlight / unread” use human-readable labels; JSON keys are shown as captions.

### 2026-04-28

- **Boolean true/false styling**
  - Segment kinds `true` and `false` color boolean values inside lines that use ordinary message categories (e.g. `/set` list rows); there are no `true` / `false` message categories
  - Theme editor exposes these under Message colors -> Segment colours only
  - Built-in themes map segment `true` to a success green and segment `false` to an error-like red in both light and dark palettes

### 2026-04-12

- **Buffer list (sidebar) SF Symbols**
  - Added optional string fields on each palette’s `ui` object: `bufferListChannelIcon`, `bufferListDirectMessageIcon`, `bufferListSystemIcon`, `bufferListRawIcon`, `bufferListWebIcon`, `bufferListScriptIcon`, `bufferListScriptDebugIcon`
  - **Omit the key or use JSON `null`**: the app uses its built-in default SF Symbol for that buffer kind (same defaults as before this feature existed)
  - **`""` (empty string)**: no leading icon for that buffer type in the sidebar, split-view headers, split layout picker, and log viewer buffer list for that light/dark palette
  - **Any other string**: treated as an SF Symbol name (same validation as other badge fields in the theme editor)
  - Light and dark palettes can specify different symbols or hide icons in one appearance only
  - Per-buffer icons set via scripts (`setbuffericon`) still override the theme for that buffer

### 2026-03-19

- **Sender prefix colour (nick mode prefix)**
  - Added `messageColors.senderPrefixColor`: `HexColor` (optional; color for sender mode prefix symbols such as `@`, `+`, `%` in the nickname column)
  - When omitted, the app uses a subtle secondary-grey fallback so existing themes continue to work unchanged

### 2026-03-05

- **Segment-based message styling**
  - Added `messageColors.segmentColors`: `{ String: HexColor }` (optional; keys are segment kind names for state message parts)
  - Added `messageColors.segmentBold`: `[ String ]` (optional; list of segment kind names that should be rendered bold)
  - Added `messageColors.segmentBraceColor`: `HexColor` (optional; color for parentheses and brackets around segments, e.g. `(reason)`, `[account]`)
  - State messages (join, part, quit, topic, mode, kick, account, CTCP, etc.) are built from tagged segments; these options control how each part is colored and whether it is bold
  - When `segmentColors` or `segmentBold` is omitted (e.g. older themes), the app uses semantic fallbacks so topic, hostmask, account, and quitmessage/partmessage/kickmessage get distinct default colours (e.g. burgundy for quit reason on red quit line) and person/timestamp/command default to bold
  - When `segmentBraceColor` is omitted, a subtle default (e.g. secondary grey) is used so `( )` and `[ ]` don’t contrast with the content

### 2025-12-16

- **Script editor syntax highlighting theme configuration**
  - Added `scriptEditorHighlightrTheme`: String (optional; Highlightr theme name for script editor syntax highlighting. Default: "github" for light mode, "github-dark" for dark mode)
  - Themes can now specify which Highlightr theme to use for TCL script syntax highlighting in the script editor
  - Each palette (light/dark) can specify its own Highlightr theme independently
  - This allows themes to match the code editor appearance to the overall theme aesthetic

### 2025-12-15

- **New blocked user badge**
  - Added `blockedBadge`: String (optional; SF Symbol name for blocked user badge. Default: "hand.raised.fill")
  - Added `blockedBadgeColor`: HexColor (optional; color for blocked badge icon. Default: system red)
  - When a user's hostmask is silenced/blocked, this badge is shown next to the nick in the nicklist and in the user info tooltip header.

- **Font configuration support**
  - Added optional `fonts` field to `ThemePalette` for configuring message fonts
  - Supports custom font names, font designs (default, monospaced, serif, rounded), and font sizes (small, body, large)
  - Font configuration applies to all message content throughout the app (MessageRow, TopicBar, raw buffer, notices)
  - If not specified, the app uses the default system body font
  - This allows themes to specify monospace fonts for IRC script alignment compatibility

### 2025-12-01

- **New optional UI field**
  - Added `expandedGroupRowBackground`: HexColor (optional; background color for expanded grouped message rows when uncollapsed. Default: accent color with ~8% opacity #0A84FF14)
  - This is optional - existing themes will use the default if not specified.

- **New bot status badge**
  - Added `botBadge`: String (optional; SF Symbol name for bot user badge. Default: "cpu")
  - Added `botBadgeColor`: HexColor (optional; color for bot badge icon. Default: system secondary label color)
  - When `person.isBot` is true, this badge is shown next to the nick in the nicklist and in the user info tooltip header.

### 2025-11-27

- **New message badge symbols**
  - Added optional badge symbol fields to UI: `joinBadge`, `partBadge`, `quitBadge`, `actionBadge`, `noticeBadge`, `infoBadge`, `topicBadge`
  - These allow themes to customize the SF Symbol icons used for message type badges in the sender column
  - All fields are optional - existing themes will use standard defaults if not specified
  - Badge symbols can be customized independently for light and dark variants

- **New metadata field**
  - Added `description`: String (optional; short human-readable description of the theme, used in UI and documentation).
  - This field is optional - existing themes without a description continue to work without changes.

### 2025-11-26

- **Message color system restructured**
  - **BREAKING**: Changed message color resolution order. Previously, `tags` had priority over `categories`. Now `categories` are checked first, then only contextual `tags`.
  - **Action required**: If your theme uses tag colors for message types (`join`, `part`, `quit`, `topic`, `error`, `notice`, `reply`, `action`, `ctcp`), move them to `categories` instead. `tags` should only be used for contextual metadata (`highlight`, `bold`, `server`, `channel`, `dm`, `raw`, `in`, `out`, `script`).

- **Categories expanded**
  - Added new category types that were previously tags: `nick`, `notice`, `mode`, `kick`, `action`, `ctcp`
  - **Action required**: If your theme defined colors for these as `tags`, move them to the `categories` object using the same keys.

- **Tags redefined**
  - Removed message-type `tags` from valid tag list: `join`, `part`, `quit`, `topic`, `error`, `notice`, `reply`, `action`, `ctcp` (these are now `categories`)
  - Added new formatting `tags`: `highlight`, `bold`
  - Added new technical metadata `tag`: `script`
  - `tags` are now only for contextual/metadata information, not message types
  - **Action required**: Remove any `tag` definitions for `join`, `part`, `quit`, `topic`, `error`, `notice`, `reply`, `action`, or `ctcp` from your `tags` object and move them to `categories` instead.

- **New optional Theme fields**
  - Added `operatorBadge`: String (optional; SF Symbol name for IRC operator badge. Default: `"shield.checkered"`)
  - Added `awayBadge`: String (optional; SF Symbol name for away user badge. Default: `"moon.fill"`)
  - Added `secureBadge`: String (optional; SF Symbol name for secure connection badge. Default: `"lock.fill"`)
  - These are optional - existing themes will continue to work and use defaults.

- **New optional UI field**
  - Added `highlightRowBackground`: HexColor (optional; background color for highlighted message rows. Default: red with ~15% opacity `#FF000026`)
  - This is optional - existing themes will use the default if not specified.

## Where to put theme files

Every frontend asks the same resolver for this folder, so it is always `Themes/` inside Rete's
per-user configuration directory. Where *that* is depends on the platform, and on macOS on whether
the program is sandboxed:

| You are running | Themes folder |
|---|---|
| **Rete.app** on macOS (sandboxed) | `~/Library/Containers/masterbofh.Rete/Data/Library/Application Support/Rete/Themes` |
| **reTerm** on macOS (not sandboxed) | `~/Library/Application Support/Rete/Themes` |
| **reTerm** on Linux | `~/.rete/Themes` |
| **Rete** on iOS | `Library/Application Support/Rete/Themes` inside the app's own data container |

**Every row above has now been run** (2026-08-25). Two of them had only ever been read out of the
source, and this is a published document, so they were measured instead:

- The **Linux** row was run in the project's 2-core Linux container (`aarch64`, Ubuntu 24.04, Swift
  6.3.1), asking the product's own resolver: `ReteUserConfigDirectory.current` answered
  `/home/rete/.rete` and `EngineConfigPaths.userThemesDirectoryURL()` answered
  `/home/rete/.rete/Themes` — i.e. `$HOME/.rete/Themes`, exactly as the row says.
- The **iOS** row was run in an iPhone 17 Pro simulator, inside a real sandboxed app data
  container. `FileManager.urls(for: .applicationSupportDirectory, in: .userDomainMask)` returned
  **exactly one** URL, `<data container>/Library/Application Support`, and `Rete/Themes` under it
  was created, written to and read back. The row previously omitted `Library/`, which is where it
  actually sits.

- The app creates this directory on first import or load attempt.
- On macOS you can also use Preferences → Appearance → “Import theme…” to copy a .json file into
  this directory, which saves you finding it.

Notes:
- The last two path components are always “Rete/Themes”, and they do not depend on the bundle
  identifier. What changes between the rows above is the directory they hang off — Rete.app's
  sandbox container, your home directory's `Library/Application Support`, `~/.rete`, or the iOS
  app's own data container.
- **`~/Library/Application Support/Rete/Themes` is reTerm's folder, not Rete.app's.** The two
  macOS programs do not share a themes folder, because Rete.app is sandboxed and reTerm is not.
  Copying a theme into one does not put it in the other.
- Files must have the .json extension.
- **What an import calls the file it writes** (changed 2026-08-26 — A1170/A1171):
  - Importing from a **URL** normally keeps the URL's own filename, so
    `https://example.com/themes/solarized.json` lands as `solarized.json`. That is unchanged.
  - If the URL does not end in something Rete can use as a filename, the file is named after the
    theme's own `name` field instead — `"<name>.json"`, the name the picker shows. A URL is
    unusable when its last path component is empty, is `.` or `..`, contains a `/` (which a URL
    *can* do — a `%2F` decodes into it), contains a control character, is longer than the
    filesystem allows, or does not end in `.json`.
  - **If the theme's `name` is not usable as a filename either, the import refuses and says so**,
    naming both. Rete will not invent a name for you and will not silently shorten one: a file
    under a name you cannot predict is a theme you cannot find again.
  - Before this, the URL's filename was used exactly as given. A URL ending `..%2Fconnections.json`
    wrote **outside** the themes folder and replaced the file there; a URL ending in a directory
    landed a file with no `.json` extension, which the picker then never listed while the import
    reported success; and an over-long one reported the operating system's *"File name too
    long"*.
- **What “Save As” is allowed to call a theme** (changed 2026-08-26 — A1209):
  - Saving a new theme writes a file called `"<name>.json"`, so the theme's **name** has to be
    usable as a filename — the same test an import applies, and the same list: it may not be
    empty, may not be `.` or `..`, may not contain a `/` or a control character, and must be short
    enough for the filesystem once `.json` is on the end. **If it is not, “Save As” refuses and
    names the theme**, rather than writing the file somewhere you did not ask for.
  - A dot inside a name is fine — `Solarized 1.2` saves as `Solarized 1.2.json`, as it always has,
    and so does a leading one (`.hidden`). What is refused is a name that is a *path* rather than a
    name, and a name that is **only** dots (`.`, `..`), because `".json"` and `"...json"` read as
    hidden files with no extension and the picker lists themes by their `.json` extension.
  - Before this, a theme named `../evil` was written **outside** the themes folder — beside
    `connections.json` rather than inside `Themes/` — and then never appeared in the picker, so the
    save looked like it had worked and the theme was gone. Where a file was already at that path
    the save was refused, but the message named *that* file (*“A file already exists at
    “connections.json””*) rather than the theme name that caused it.
  - **Plain “Save” (overwrite) is unchanged**, including for a theme already on disk whose name is
    one of the above: it writes back to the file the theme was loaded from, which is a file inside
    the themes folder by construction. To save such a theme under a new file you have to rename it
    first.
- **An import never destroys anything until the replacement is known to be good** (changed
  2026-08-26 — A1171). Importing a file that turns out not to be a theme used to delete whatever
  theme was already under that filename **before** reading the file, so a failed import destroyed a
  working theme. Importing a valid theme over an existing one still replaces it, as it always did.
- **If a *folder* is sitting where the theme file would go, the import refuses** and tells you the
  path — *“…” is a folder, not a theme file, so Rete did not replace it.* It used to delete that
  folder and everything inside it, recursively, without a word, and report success.
- **If that path exists but is not a folder, saving and importing a theme say so too**, in the
  same words — *“…” is a file, not a folder, so Rete cannot use it as your themes folder.* That
  covers Save, Save As, “Import theme…”, importing from a URL, and importing a folder of themes.
  Until 2026-08-25 those reported the operating system's own message instead, which named the
  path as a folder, and Save (overwrite) reported that the theme was not on disk.
- **If that path exists but is not a folder, Rete says so** and falls back to the built-in
  `System` theme rather than behaving as though you had no themes. The same is true if the
  folder cannot be listed for any other reason. This is the same message on macOS, iOS, Linux
  and reTerm.

## JSON schema (overview)


Top-level object (`Theme`):
- `name`: `String` (unique theme name; used in the picker)
- `description`: `String` (optional; short human-readable description of the theme)
- `author`: `String` (optional)
- `version`: `String` (optional)
- `preferDark`: `Bool` (optional; hint for default variant choice)
- `operatorBadge`: `String` (optional; SF Symbol name for IRC operator badge. Default: `"shield.checkered"`)
- `awayBadge`: `String` (optional; SF Symbol name for away user badge. Default: `"moon.fill"`)
- `secureBadge`: `String` (optional; SF Symbol name for secure connection badge. Default: `"lock.fill"`)
- `botBadge`: `String` (optional; SF Symbol name for bot user badge. Default: `"cpu"`)
- `blockedBadge`: `String` (optional; SF Symbol name for blocked user badge. Default: `"hand.raised.fill"`)
- `light`: `ThemePalette`
- `dark`: `ThemePalette`

`ThemePalette`:
- `ui`: `UI`
- `messageColors`: `MessageColors`
- `fonts`: `Fonts` (optional)

`Fonts` (optional):
- `messageFontName`: `String` (optional; custom font name, e.g., `"SF Mono"`, `"Menlo"`, `"Courier New"`)
- `messageFontDesign`: `String` (optional; one of: `"default"`, `"monospaced"`, `"serif"`, `"rounded"`)
- `messageFontSize`: `String` (optional; one of: `"small"`, `"body"`, `"large"`)

`UI` (all colors are `HexColor` objects):
- `background` — also what per-nick colours are made readable against; see
  "Unique nick colours and `ui.background`" below
- `foreground`
- `accent`
- `divider`
- `topicBarBackground`
- `topicBarForeground`
- `badgeBackground`
- `badgeForeground`
- `sidebarSelectionBackground`
- `sidebarSelectionForeground`
- `highlightRowBackground` (optional): Background color for highlighted message rows. Default: red with ~15% opacity (`#FF000026`)
- `expandedGroupRowBackground` (optional): Background color for expanded grouped message rows (uncollapsed message groups). Default: accent color with ~8% opacity (`#0A84FF14`)
- `operatorBadgeColor` (optional): `HexColor` for operator badge icon color. Default: system secondary label color.
- `awayBadgeColor` (optional): `HexColor` for away badge icon color. Default: system secondary label color.
- `secureBadgeColor` (optional): `HexColor` for secure connection badge icon color. Default: system green.
- `botBadgeColor` (optional): `HexColor` for bot badge icon color. Default: system secondary label color.
- `blockedBadgeColor` (optional): `HexColor` for blocked badge icon color. Default: system red.
- `bufferListHighlightBadgeBackground` (optional): Capsule fill for the **highlight** count badge in the buffer list (sidebar, split pane headers, topic bar). Default: red at ~80% opacity (SwiftUI) / theme-driven in reTerm when set.
- `bufferListHighlightBadgeForeground` (optional): Text colour on that highlight badge. Default: white (or bright foreground on highlight-coloured text in reTerm when no capsule background is set).
- `bufferListUnreadBadgeBackground` (optional): Capsule fill for the **unread** count badge (shown when there are unreads and no highlight badge). Default: grey at ~60% opacity.
- `bufferListUnreadBadgeForeground` (optional): Text colour on the unread badge. Default: white.
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
- `bufferListChannelIcon` (optional): `String` (SF Symbol for channel rows in the buffer list; omit/`null` = built-in default `"number"`; `""` = no icon)
- `bufferListDirectMessageIcon` (optional): `String` (direct message / nick buffer rows; default `"person"`; `""` = no icon)
- `bufferListSystemIcon` (optional): `String` (server / system buffer rows; default `"gearshape"`; `""` = no icon)
- `bufferListRawIcon` (optional): `String` (raw traffic buffer; default `"waveform.path"`; `""` = no icon)
- `bufferListWebIcon` (optional): `String` (web content buffers; default `"globe"`; `""` = no icon)
- `bufferListScriptIcon` (optional): `String` (script buffers; default `"terminal.fill"`; `""` = no icon)
- `bufferListScriptDebugIcon` (optional): `String` (script debug buffer; default `"terminal"`; `""` = no icon)
- `scriptEditorHighlightrTheme` (optional): `String` (Highlightr theme name for script editor syntax highlighting. Default: "github" for light mode, "github-dark" for dark mode)

`MessageColors`:
- `categories`: `{ String: HexColor }`
  - Keys are `IRCMessageCategory.rawValue` (primary message type classification):
    - `normal`, `serverNotice`, `serverReply`, `info`, `error`, `join`, `part`, `quit`, `topic`, `nick`, `notice`, `mode`, `kick`, `action`, `ctcp`
  - **Categories are the primary way to color messages** - each message has exactly one category
- `tags`: `{ String: HexColor }`
  - Keys are `IRCMessageTag.rawValue` (contextual/metadata tags only):
    - `highlight`, `bold` (formatting)
    - `server`, `channel`, `dm` (context - where the message appears)
    - `raw`, `in`, `out`, `script` (technical metadata)
    - `do` (TCL command input/output in script debug buffer)
    - `silence` (silence/block related messages)
  - **Tags are only for contextual information** - do NOT duplicate category values (e.g., don't use `"join"` tag if category is already `"join"`)
- `segmentColors` (optional): `{ String: HexColor }`
  - Keys are segment kind names. Used to color specific parts of join/part/quit/topic/mode/kick/account/CTCP etc. messages, and boolean literals via the `true` / `false` segment kinds in settings-style rows.
  - Supported keys: `literal`, `channel`, `person`, `topic`, `reason`, `hostmask`, `account`, `timestamp`, `mode`, `modeParam`, `command`, `value`, `true`, `false` (boolean literals in settings-style rows)
  - If omitted, semantic fallbacks (e.g. category `info` for hostmask/account/reason, accent for topic) keep parts distinct from the line
- `segmentBold` (optional): `[ String ]`
  - List of segment kind names that should be rendered bold (e.g. `["person", "timestamp", "command"]`)
  - When omitted or empty, the app still bolds `person`, `timestamp`, and `command` by default
- `segmentBraceColor` (optional): `HexColor`
  - Color for parentheses and brackets around segments, e.g. `(reason)` and `[account]` in part/quit messages
  - When omitted, a subtle default (e.g. secondary grey) is used so braces don’t contrast with the content
- `senderPrefixColor` (optional): `HexColor`
  - Color for sender mode prefix symbols in the nickname column, e.g. `@nick`, `+nick`, `%nick` (the prefix glyph itself)
  - When omitted, a subtle default (secondary grey) is used


`HexColor`:
- Object with a single field:
  - `hex`: `"#RRGGBB"`

- Two forms are accepted: `"#RRGGBB"` and `"#RRGGBBAA"`, where `AA` is alpha (`00` fully
  transparent, `FF` fully opaque). Case does not matter, and the leading `#` is optional —
  `"0A84FF"` and `"#0a84ff"` name the same colour as `"#0A84FF"`.
- **Three-digit shorthand (`"#ABC"`) is expanded to six** when the file is read: `"#0af"`
  becomes `"#00AAFF"`. It works with or without the `#`, and it works everywhere a `HexColor`
  appears.

  > **Changed 2026-08-21.** This document said in two places that shorthand was *not*
  > supported and in a third that it was expanded, and the code agreed with the first two: the
  > normaliser that expands `"#ABC"` ran only on colours built inside the app and was skipped
  > on the one path that carries hex a person actually typed. A shorthand value was read as-is,
  > failed to parse, and the key silently fell back — which is the worst of the three possible
  > behaviours. **The code now matches the third sentence, which is the one this document
  > should always have meant.** Nothing Rete writes changes: the app saves the expanded form,
  > so an existing theme file is byte-identical after a load and save.
- Alpha is most useful for the keys that tint something behind them — `highlightRowBackground`,
  `expandedGroupRowBackground`, `sidebarSelectionBackground`, `badgeBackground`, `divider`.
  The built-in theme uses it for exactly those.
- **How alpha renders on each frontend.** macOS and iOS draw the colour translucently over
  whatever is behind it. A terminal cell has no alpha channel, so **reTerm composites the
  colour against the theme's own `ui.background`** and draws the result — the same arithmetic,
  resolved against the backdrop reTerm actually paints. One theme file therefore looks the
  same on all three. Where a colour sits over something other than the background (a badge
  inside the topic bar), reTerm's result is an approximation of what the GUI shows.

Example `HexColor`:
`{ "hex": "#0A84FF" }`

## Fallback behavior

### A colour that will not parse is not the same as a colour you did not set

**Three cases, not two** (2026-08-19). A hex value is checked before it is drawn, and a value
that cannot be drawn takes **the system default for Dark/Light** — not the fallback the key
would have had if you had left it out.

| The theme file | What is drawn |
|---|---|
| names no colour for the key | the fallback for that key, below |
| names a colour that **will not parse** | **the system default for Dark/Light** |
| names a valid colour | that colour |

This applies to **every** themed colour — categories, tags, segment colours, `ui.*` — not to
one key. The point is that a typo should look like a typo. Before this, an unparseable value
fell through every fallback in the chain and the segment quietly took the colour of the line
around it, which is indistinguishable from having set nothing.

- On `Rete.app` and iOS the system default is `Color.primary`, the adaptive label colour.
- On `reTerm` it is the terminal's own default foreground (`ESC [ 39 m`), which follows
  whatever light or dark scheme you have set on your terminal. It is **not** the grey reTerm
  used to fall back to, which was the same answer it gave for a colour you had not set.

**What counts as parseable is now one rule on all three frontends**: `#RRGGBB` or `#RRGGBBAA`,
exactly six or eight hex digits, with or without the leading `#`. A three-digit value is
expanded before this check. Anything else — `#ABCDEFG`, `#FF`, `nope` — will not parse.
Previously reTerm accepted anything six digits or longer and used the first six, so
`#ABCDEFG` drew on reTerm and fell back on macOS.

**And that rule now really does run on a file, not only on a colour built in code**
(2026-08-21). It did not before: the expansion and the optional `#` were applied when Rete
constructed a colour and skipped when it decoded one, so a hand-written `"#0af"` fell back
and a hand-written `"0A84FF"` drew correctly while the theme editor refused to accept the
file it came from. Both are fixed, and the macOS theme editor and the renderers now agree
about every value a theme file can hold.

If a color is missing:
- UI background falls back to system window background (macOS) / systemBackground (iOS/iPadOS).
- Foreground falls back to Color.primary.
- Accent falls back to Color.accentColor.
- Divider/topic bar/badge/sidebar selection colors fall back to reasonable defaults based on accent and system colors.

Message color resolution:
1. **Category color** is checked first (message.category.rawValue) - this is the primary classification
2. **Tag colors** are checked only for contextual tags (highlight, bold, server, channel, dm, raw, in, out, script, do) that don't map to categories
3. If neither category nor tag color is defined, the UI foreground color is used
4. Final fallback is Color.primary

**Important**: Categories and tags no longer overlap. Categories handle message types (join, part, mode, etc.), while tags only provide contextual metadata (where the message appears, formatting, etc.). This eliminates confusion about which to use for coloring.

## Segment-based message styling

State messages (join, part, quit, topic, mode, kick, account, CTCP, nick change, etc.) are built from **segments**: each part (nick, channel, reason, topic text, user@host, account, timestamp, command name, etc.) is tagged with a **segment kind**. The theme can then color and bold those parts independently via `segmentColors` and `segmentBold`, and style the parentheses/brackets around them via `segmentBraceColor`.

### Segment kinds

| Key | Used for |
|-----|----------|
| `literal` | Plain text (e.g. " has joined ", " has left ") |
| `channel` | Channel names (e.g. #foo); also styled by accent for clickability |
| `person` | Nicks; use nick color rules when in channel, else segment style; often bold |
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
   under the borrowing table. It is not in that table's `categories.join` row any more.
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

> **`channel` works like `topic`, and is always bold** (changed 2026-08-19). It takes
> `ui.accent` when `segmentColors.channel` is absent and that colour when it is set — on all
> three frontends. **Bold is not negotiable**: the accent marks a channel name as *clickable*,
> so `segmentBold` cannot switch it off. A theme may recolour a channel name; it may not make
> it stop reading as a control.
>
> **Before this, `segmentColors.channel` had no effect at all** — each renderer drew the accent
> in its own view code without consulting the theme. The built-in themes carried a
> `segmentColors.channel` entry that was never drawn, and it has been **removed** rather than
> honoured: its value was the same green as `categories.join`, so a channel name would have
> become indistinguishable from the join line it usually sits in. With no entry the accent
> applies, which is what the built-in themes have always looked like — and it now follows
> `ui.accent` if you change that.

### Which segments are bold

`segmentBold` is a **replacement, not an addition**, and an omitted key is not the same as
an empty list only in the sense that JSON cannot tell them apart:

- **Omitted** — `person`, `timestamp` and `command` are bold.
- **`"segmentBold": []`** — the same three are bold. An absent key and an empty array decode
  identically, so **there is currently no way for a theme to say "bold nothing"**.
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

The app supports:
- Automatic: follows system appearance
- Light: forces theme.light
- Dark: forces theme.dark

The ThemeManager resolves the active palette at runtime. The theme must define both light and dark palettes.

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
        "error": { "hex": "#FF453A" }
      },
      "tags": {
        "notice": { "hex": "#FF6B81" },
        "ctcp": { "hex": "#FF453A" }
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
  - **This changed on 2026-08-23.** Before then malformed files were ignored silently and this
    entry told you to enable debug logging to find out which one. The debug log still carries
    the decoding error in full — the field and the coding path, which the user-facing message
    does not — so it remains the place to look for *why* a file failed rather than *which*.
  - A file with no `.json` extension is not a candidate at all and is not reported; it is not a
    theme file as far as Rete is concerned.
- Decoding error mentioning “Expected to decode Dictionary<String, Any> but found a string”
  - You likely used a plain string for a color. Use an object: "background": { "hex": "#RRGGBB" }.
- Alpha/transparency in hex
  - Supported: use `"#RRGGBBAA"`. macOS and iOS draw it translucently; reTerm composites it
    against the theme's `ui.background` and draws the result, so the three frontends agree.
  - A colour looks right in the app and wrong in reTerm
    - Check whether it is drawn over something other than `ui.background` — a badge inside
      the topic bar, for instance. reTerm composites against the background, so a tint over a
      different backdrop is an approximation there.
- A colour is ignored entirely
  - Check the digit count. After the leading `#` is stripped and a three-digit shorthand is
    expanded, the value must be **exactly six or eight** hex digits. `"#ABCDEFG"` (seven),
    `"#FF"` (two) and `"nope"` fall back to the system default for Dark/Light with no error.
  - Three-digit shorthand such as `"#ABC"` is **not** one of these cases any more — it is
    expanded to `"#AABBCC"` when the file is read. Before 2026-08-21 it was read as-is, failed
    to parse and fell back silently, which is what this entry used to describe.

## Unique nick colours and `ui.background`

With **unique nick colours** turned on, a nickname's colour is derived from the nickname itself
rather than from the theme — so it is not a field you set. It *is* affected by one field you do
set: the colour is adjusted until it reaches a **3:1 contrast ratio against `ui.background`**
(WCAG AA for large text; nicknames are short and bold).

**Any `ui.background` works** as of 2026-08-21. Before then only clearly dark and clearly light
backgrounds did: the adjustment walked in a single direction chosen by whether the background was
dark, and a **mid-grey** background is dark by that test while being too light to walk away from.
On `#A0A0A0`, 94% of nicknames came out below 3:1 and the worst were drawn in *exactly* the
background colour — invisible. The adjustment now tries both directions and keeps the better, so a
grey theme is a supported choice rather than a broken one. Nothing changes on a background that
already worked.

Two consequences worth knowing:

- **The same nickname is a different colour in your light and dark palettes**, deliberately —
  each is corrected against its own `ui.background`.
- **macOS and iOS draw the same nickname the same colour.** They did not before 2026-08-21;
  iOS used a different colour model and applied no correction at all. If your iOS nicknames
  look different from how you remember them, that is why.

## Badge Icons

Themes can customize the SF Symbol icons used for status badges in the nicklist and user info tooltips:

- **operatorBadge**: SF Symbol name for IRC operator badge (default: "shield.checkered")
- **awayBadge**: SF Symbol name for away user badge (default: "moon.fill")
- **secureBadge**: SF Symbol name for secure connection badge (default: "lock.fill")
 - **botBadge**: SF Symbol name for bot user badge (default: "cpu")
- **blockedBadge**: SF Symbol name for blocked user badge (default: "hand.raised.fill")

These badges appear next to nicknames in the nicklist and in the user info tooltip when hovering over a nickname.

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

If not specified, the default icons are used. The badge icons apply to both light and dark variants.

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

Themes can customize the SF Symbol shown to the left of each buffer in the **sidebar buffer list** (and the same icon is used in a few related places: split-view pane headers, split layout assignment UI, and the log viewer’s buffer list). This is separate from **message** badges (`joinBadge`, etc.) and from **nicklist** badges (`operatorBadge`, etc.), which live at the top level of the theme or only affect message rows.

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

Per-buffer **custom** icons set from TCL (`setbuffericon`) still override the theme for that buffer.

## Buffer list activity, highlights, and count badges

When a buffer has **highlights**, a count badge is shown (red capsule by default on macOS / iOS). When it has **unreads** (or suppressed highlights) but no highlight tally, a grey capsule shows the unread count. The same counts appear in **reTerm** in the right-hand columns of sidebar rows (truecolour when theme hex is set).

Optional `ui` fields (per palette):

| JSON key | Purpose | Default when omitted |
|----------|---------|----------------------|
| `bufferListHighlightBadgeBackground` | Capsule fill behind the highlight count | macOS/iOS: system red ~80% opacity; reTerm: no filled pill (foreground-only badge) |
| `bufferListHighlightBadgeForeground` | Text (and icon) colour on the highlight badge | White (reTerm: coral-ish default foreground when no background is set) |
| `bufferListUnreadBadgeBackground` | Capsule fill behind the unread count | macOS/iOS: grey ~60% opacity; reTerm: no pill unless set |
| `bufferListUnreadBadgeForeground` | Unread count text colour | White on macOS/iOS when a capsule background is used; reTerm: muted grey default when no background |
| `bufferListHighlightBufferNameColor` | Buffer **title** colour when the row has a **highlight** (red highlight badge) | Normal `foreground` |
| `bufferListActiveBufferNameColor` | Buffer **title** colour when the row has **unread** and/or **channel activity** but not a highlight tally | Normal `foreground` |
| `bufferListNotifyNameBold` | Whether the buffer name becomes **bold** while emphasized | `true` (bold) |

**reTerm** reads these keys from the theme file you have selected, not from a built-in
palette. It loads the same themes directory the apps do; `/set theme <name>` switches,
and `/set theme reload` re-reads the directory after you edit a file. Where the
"default when omitted" column gives a different answer for reTerm, that is because a
terminal cell cannot draw a capsule — not because the key is ignored.

**reTerm renders the `dark` palette unconditionally** — it has no appearance mode, and a
theme's `light` palette is never consulted there, so a key you set only under `light`
has no effect in reTerm. This is true of the whole theme, not just the fields in this
section. (Separately, a colour you leave *unset* falls back to the terminal's own
default foreground, which does follow your terminal's scheme — see "If a color is
missing" above.)

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
  - Maps to: `small` → `.callout`, `body` → `.body`, `large` → `.title3`

### Font Resolution

1. If `messageFontName` is specified and the font exists on the system, use that custom font
2. Otherwise, use system font with the specified `messageFontDesign`
3. If no font configuration is provided, default to system body font (`.body`)

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
- The font applies to all message content: MessageRow text, TopicBar topic text, raw buffer, and notices
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
- The theme applies only when the Highlightr package is available in the app
- Each palette (light/dark) can specify its own Highlightr theme independently
- The syntax highlighting automatically adapts to theme changes when switching between light and dark modes

## Versioning

- The Theme structure supports a version string. There's no strict validation yet, but you can use it to track your theme iterations.
