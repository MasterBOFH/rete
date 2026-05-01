---
layout: doc
title: Themes Tutorial
permalink: /docs/themes/
---
Rete supports custom themes defined as JSON files. Themes provide UI colors and message colors for both Light and Dark variants. Users can import themes from Preferences or by placing files in the Themes folder.

## Changelog

### 2026-04-28

- **Boolean true/false styling**
  - Added message category keys: `true` and `false` for full-line semantic states (for flows that intentionally mark the whole line as true/false)
  - Added segment kinds: `true` and `false` for value-only styling inside otherwise normal info lines (used by `/set` list rows)
  - Theme editor exposes category colors under Message colors -> Categories, and segment colors under Message colors -> Segment colours
  - Built-in themes map true to a success green and false to an error-like red in both light and dark palettes

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

- macOS (user): ~/Library/Application Support/Rete/Themes
  - The app creates this directory on first import or load attempt.
  - You can also use Preferences → Appearance → “Import theme…” to copy a .json file into this directory.

Notes:
- The directory name is hardcoded to “Rete/Themes” under Application Support and does not depend on the bundle identifier.
- Files must have the .json extension.

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
- `background`
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
    - `normal`, `true`, `false`, `serverNotice`, `serverReply`, `info`, `error`, `join`, `part`, `quit`, `topic`, `nick`, `notice`, `mode`, `kick`, `action`, `ctcp`
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
  - Keys are segment kind names for state message parts. Used to color specific parts of join/part/quit/topic/mode/kick/account/CTCP etc. messages.
  - Supported keys: `literal`, `channel`, `person`, `topic`, `reason`, `hostmask`, `account`, `timestamp`, `mode`, `modeParam`, `command`, `value`
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

- Important: Only 6-digit RGB is supported by the current `Color(hex:)` helper (no alpha channel). If you want a “lighter” effect, choose a lighter color rather than using transparency.

Example `HexColor`:
`{ "hex": "#0A84FF" }`

## Fallback behavior

If a color is missing or invalid:
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
  - Place it in: ~/Library/Application Support/Rete/Themes
  - Validate JSON syntax and schema. The app currently ignores malformed files silently, but with debug logs enabled you’ll see:
    - Theme directory path
    - List of candidate files
    - Decoding errors per file
- Decoding error mentioning “Expected to decode Dictionary<String, Any> but found a string”
  - You likely used a plain string for a color. Use an object: "background": { "hex": "#RRGGBB" }.
- Alpha/transparency in hex
  - Not supported. Use 6-digit hex. Choose lighter/darker colors to simulate translucency.

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
