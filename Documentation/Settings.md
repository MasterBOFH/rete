# Settings — the `/set` command

`/set` is Rete's settings surface. On **reTerm it is the only one** — reTerm has no Preferences window — and on **macOS it sits beside Preferences**, writing the same storage, so a key changed here and a checkbox toggled there are the same setting.

## Grammar

| Form | Does |
|---|---|
| `/set` | Print every key and its current value. |
| `/set list` | The same table. |
| `/set <key>` | Print one key's current value. |
| `/set <key> <value>` | Set it, then print what it became. |
| `/set <key> toggle` | Flip a boolean key. |
| `/set help <key>` | Print that key's one-line description and current value. |

Keys are matched case-insensitively, and **aliases resolve to the same setting as the primary key** — `/set mirc off` and `/set show_mirc_colors off` are one command. A primary key always wins over another key's alias. An unknown key answers `Unknown key: <what you typed>. /set list`.

### Value grammars

| Accepted values | Written as |
|---|---|
| `bool` | `on` `off` `true` `false` `yes` `no` `1` `0` `enable` `disable` `enabled` `disabled`, or `toggle` to flip it. |
| `int` | A whole number. Out-of-range values are clamped, not rejected. |
| `number` | A decimal number. |
| `string or off` | Any text; `off`, `none`, `clear`, `nil` or `(none)` (any case) clears it back to unset. A blank value cannot be stored: /set &lt;key&gt; with nothing after it shows the current value, and quote marks are stored as typed. |
| `number or off` | Positive seconds; `off`, `none`, `clear`, `nil` or `0` clears the limit. |
| `short \| extended` | `short`/`s`, or `extended`/`e`/`long`/`full`. |

A rejected value leaves the setting alone and prints why (`Expected on, off, true, false, or toggle; got: <what you typed>`). Values outside a range are **clamped by whoever owns the setting**, not rejected here, so `/set` echoes back the clamped value rather than an error.

## Where the values live

- **macOS and iOS** — the app's preferences, the same storage Preferences writes. The engine-owned settings (quit/part message, the flood throttle, buffer rows, ban mask, log retention) are kept in the engine's own state file instead, which is why they read the same in every frontend.
- **Linux** — `~/.rete/irc-defaults.json`, written on every change, whether it is made with `/set` in reTerm or ReteCross or in a settings window, and read back when the program next starts.
- **Credentials** — `retecloud_password` is written to the Keychain (service `Rete.CDN`) and never to the mirror file. It prints as `(redacted)` when a password is stored and `(none)` when the store answered and there was none. When the store could not be read at all it prints `(unknown …)` instead of either, on both platforms: a keychain that would not answer is not evidence that there is no password. On macOS a refusal that will clear by itself — a keychain locked at login, a declined prompt — says so, and tells you to unlock and try again.

## Outbound pacing

The seven `flood_throttle*` keys are **two mechanisms, not one**, plus a byte bound on a burst. Rete models the rule a real IRC server enforces: it charges a *penalty* per line, lets a client run a fixed amount of credit ahead of real time, and stops reading once the credit is spent.

- `flood_throttle_min_delay_ms` (chat) and `flood_throttle_command_delay_ms` (everything else) are what one line **charges the penalty ledger**. `flood_throttle_burst_ms` is the **credit**: how far ahead of real time the ledger may run before a line waits at all. At the shipped 2000 / 10000 numbers, six chat lines go instantly and the seventh waits two seconds.
- `flood_throttle_max_bytes` / `flood_throttle_window_seconds` are a **token bucket** over outbound bytes. It starts *empty* on connect and on every settings change, so there is no initial burst. A line longer than `flood_throttle_max_bytes` is sent once the bucket is full, and the excess is paid back before the next line goes.
- `flood_throttle_burst_bytes` bounds a burst in **bytes**: lines that leave less than a second apart may carry at most that many bytes between them — 900 by default, under the smallest server receive queue measured (`ircd-irc2`, 1,000 bytes). At 900 a long paste goes two full chunks at a time rather than all at once; one line on its own always goes. `0` removes the byte bound.
- `PING`, `PONG`, `QUIT` and the registration handshake (`PASS`, `USER`, `CAP`, `AUTHENTICATE`) charge **nothing** and never wait, so a goodbye is never queued behind chat and a connect never stalls.

**The sustained rate is what you feel, and the 2000 ms chat penalty is deliberate.** Past the credit, four 59-byte chat lines take about 2 s each. So a long paste slows down after its first few lines; a message over ~500 bytes is split in two and the second half may be 2 s behind the first. Your *own* copy of what you sent is never delayed — it is shown before the pacing, not after.

It is not redundant with the bucket, and neither figure is decorative. Measured against a real ircu2 with a 1024-byte receive queue: a 61-byte line every 2 s survives and every 1 s is killed, while a 501-byte line every 4 s is killed and every 8 s survives — so **neither the line rate nor the byte rate predicts on its own**, and the server is charging roughly `2 + bytes/100` seconds per line. `flood_throttle off` disables both mechanisms entirely.

**`flood_throttle_command_delay_ms` is a flood bound, not the server's rule.** The server charges non-chat commands the same ~2 s it charges chat; 250 ms is shipped because Rete's own connect emits more commands than the credit covers — a ten-channel autojoin sends 36 of them. Raise it to 2000 for a strict server and expect a slower connect; set it to 0 and a stock ircu2 will drop you after 84 commands.

## Keys — all platforms

62 keys, present in both full products.

| Key | Aliases | Accepted values | Default | What it does |
|---|---|---|---|---|
| `auto_away` | `auto_away_enabled` | `bool` | `false` | auto-away after idle. |
| `auto_away_minutes` | — | `int` | `15` | idle minutes before auto-away (≥1, default 15). |
| `auto_away_template` | `auto_away_message` | `string` | `Auto-away: Idle since %%DATE%%` | away message template (%%DATE%% placeholder). |
| `bind_host` | `bind` | `string or off` | `(none)` | outbound bind address; off / none / clear unsets it. |
| `collapse_activity` | `collapse_activity_messages` | `bool` | `true` | group join/part/quit lines. |
| `collapse_activity_window_seconds` | `collapse_activity_window` | `number or off` | `off` | seconds to group activity (nil = no time limit). Use off, none, or clear. |
| `control.debug` | — | `bool` | `false` | also serve the debug tools (raw lines, engine state) while AI access is on. Default false. |
| `control.enabled` | — | `bool` | `false` | let AI agents read Rete over MCP on 127.0.0.1 (see Control.md). Default false. |
| `control.port` | — | `int 1024…65535` | `7421` (macOS app), `7422` (reTerm) | the port AI access listens on. Default 7421 in the macOS app, 7422 in reTerm. |
| `control.send` | — | `bool` | `false` | let AI agents send raw lines to IRC with send_raw while AI access is on. An agent can send anything as you, including text it read on IRC (see Control.md). Not saved: off at every launch, and setting it changes this run only. Default false. |
| `default_alt_nick` | `altnick` | `string` | `default_nick` + `_` | alternate nick. |
| `default_nick` | — | `string` | `rete` | default nickname for new connections. |
| `default_real_name` | `realname` | `string` | `Rete Client` | default real name (gecos). |
| `default_username` | `username`, `ident` | `string` | `rete` | default username (ident). |
| `emoticon_shortcuts` | `emoticons` | `bool` | `true` | replace ASCII emoticons in chat text, /me and /notice before send. |
| `flood_throttle` | `flood_throttle_enabled` | `bool` | `true` | pace outbound chat (PRIVMSG/NOTICE) so the server does not drop you for flooding. |
| `flood_throttle_burst_bytes` | `flood_burst_bytes` | `int` (0…65536) | `900` | the most bytes one send burst may carry: lines that leave less than a second apart (0…65536, default 900, under the smallest measured server receive queue, ircd-irc2's 1,000 bytes). A line that would take a burst past it waits; one line on its own always goes. 0 is no byte bound. |
| `flood_throttle_burst_ms` | `flood_burst_ms` | `int` (0…60000) | `10000` | burst credit: how far ahead of real time the penalty ledger may run before a line waits (0…60000, default 10000). At the default chat cost that is six lines instantly, then one every two seconds. 0 restores the flat floor. |
| `flood_throttle_command_delay_ms` | `flood_command_delay_ms` | `int` (0…60000) | `250` | what one non-chat command (WHO, MODE, JOIN, WHOIS, …) charges the same ledger, in ms (0…60000, default 250). PING, PONG, QUIT and the registration handshake charge nothing. 0 leaves them unpaced, and a stock ircu2 drops you after 84 of them. |
| `flood_throttle_max_bytes` | — | `int` (256…65536) | `2048` | token-bucket burst size in bytes per window (256…65536, default 2048). |
| `flood_throttle_min_delay_ms` | `flood_min_delay_ms` | `int` (0…60000) | `2000` | what one chat line (PRIVMSG/NOTICE) charges the outbound penalty ledger, in ms (0…60000, default 2000). Not a fixed delay between lines: see flood_throttle_burst_ms. |
| `flood_throttle_window_seconds` | `flood_throttle_window` | `number` (0.1…60) | `2` | token-bucket window in seconds (0.1…60, default 2). |
| `highlight_notification_sound` | `highlight_sound` | `bool` | `true` | play sound with highlight notifications. |
| `highlight_notifications` | — | `bool` | `true` | desktop notifications for highlights: banners in the macOS app, desktop notifications in ReteCross on Linux. reTerm shows none. |
| `image_embeds` | `image_embed_mode`, `images` | `click_to_load \| blur \| show` | `click_to_load` | what an image link in a message does before you click it: click_to_load requests nothing, so the image host never sees you; blur downloads it and hides it; show downloads and displays it. Preferences' Images in messages. Default click_to_load. |
| `log_retention_days` | `log_retention` | `int` (0…36500) | `0` | days of message log to keep; older rows are pruned (0…36500, default 0 = keep forever). |
| `max_buffer_rows` | `max_rows` | `int` (100…50000) | `500` (iOS: `250`) | max rows per buffer before purge (100…50000). Unset, the engine's default for this platform applies; /set list shows the value in use. |
| `media_apple_music` | — | `bool` | `false` | the macOS app's media bar offers Apple Music. Other apps do not read it. Default false. |
| `media_share_template` | — | `string` | `I’m listening to %TRACK% by %ARTIST% on %SOURCE%` | the message the macOS app's media bar shares: %TRACK%, %ARTIST%, %SOURCE% and %URL% are filled in. |
| `media_spotify` | — | `bool` | `false` | the macOS app's media bar offers Spotify. Other apps do not read it. Default false. |
| `media_stream_url` | `media_station_url` | `string or off` | `(none)` | the macOS app's media bar: a live stream's address (.mp3 or .m3u). Unset, the bar offers no stream. Other apps do not read it. |
| `message_timestamp_pattern` | `timestamp_pattern`, `ts_pattern` | `string` | `HH:mm` | Unicode TR35 pattern for timestamps (e.g. HH:mm, HH:mm:ss). |
| `nicklist_fixed_width_enabled` | `nicklist_fixed_width` | `bool` | `false` | use a fixed nicklist width instead of dynamic sizing. |
| `nicklist_fixed_width_value` | `nicklist_width` | `number` (120…600) | `220` | fixed nicklist width in points. Out-of-range values are clamped. |
| `nickname_hover_delay` | `hover_delay`, `nickname_hover_delay_seconds` | `number` | `1.5` | seconds before a nick's hover card opens, 0.0–5.0 (default 1.5). Out-of-range values are clamped. |
| `part_message` | `part` | `string` | `Leaving` | PART message. |
| `prefer_account_ban` | `account_ban` | `bool` | `true` | /ban and kickban use an account extban on servers advertising ACCOUNTEXTBAN when the target is logged in; otherwise a hostmask ban. |
| `prefer_server_time` | `prefer_server_timestamp` | `bool` | `true` | show a message's IRCv3 server-time instead of when Rete received it, when the server sent one. ReteCloud rows always show the event time. |
| `preferred_ban_mask_type` | `ban_mask_type`, `ban_mask` | `int 0…39` | `3` | hostmask style for /ban and kickban when an account ban is not used. 0–9 are the usual Eggdrop presets; 10–39 widen or strip host digits. Same table as Tcl `maskhost`. Default 3. Out-of-range values are clamped. |
| `quit_message` | `quit` | `string` | `Client exiting` | QUIT message. |
| `show_mirc_colors` | `mirc_colors`, `mirc` | `bool` | `true` | render mIRC colour/formatting codes in messages. |
| `show_nicklist_badges` | `nicklist_badges` | `bool` | `true` | show status badges in the nicklist. |
| `show_nicklist_user_count` | `nicklist_user_count` | `bool` | `true` | show user-count summary for the nicklist. |
| `sidebar_width` | — | `int 160…400` | `220` | the sidebar's width in pixels, kept when its divider is dragged, in the macOS app and in ReteCross (Linux); a change moves the sidebar at once. Out-of-range values are clamped. Default 220. reTerm sizes its buffer list from the terminal's width in columns and does not read it; the iOS app has no resizable sidebar. |
| `terminal_mouse` | `mouse` | `bool` | `true` | reTerm: let the terminal send clicks and the wheel to reTerm (click a buffer, a nick, a channel or a link; the wheel scrolls the pane under the pointer). On by default: to select text while it is on, hold Shift as you drag (Option in iTerm2, Fn in Terminal.app). Off, a plain drag selects text to copy as in any terminal program, and PageUp/PageDown scroll. Switches at once. Other frontends do not read it. |
| `theme` | — | `name or reload` | (empty) | theme to render with (from the Themes directory). `/set theme reload` re-reads the files. An unknown name is rejected and lists what is available. |
| `toolbar_show_block_list` | — | `bool` | `true` | show the Block List button in the toolbar. |
| `toolbar_show_channel_list` | — | `bool` | `true` | show the Channel List (/LIST) button in the toolbar. |
| `toolbar_show_connections` | — | `bool` | `true` | show the Connections button in the toolbar. |
| `toolbar_show_logs` | — | `bool` | `true` | show the Logs button in the toolbar. |
| `toolbar_show_new_web_buffer` | — | `bool` | `true` | show the New Web Buffer button in the toolbar. |
| `toolbar_show_scripts` | — | `bool` | `true` | show the Scripts button in the toolbar. |
| `toolbar_show_settings` | — | `bool` | `true` | show the Settings button in the toolbar. |
| `toolbar_show_theme_editor` | — | `bool` | `true` | show the Theme Editor button in the toolbar. |
| `typing_send_channels` | `typing_channels` | `bool` | `false` | send IRCv3 +typing indicators in channels (default false). |
| `typing_send_queries` | `typing_queries` | `bool` | `true` | send IRCv3 +typing indicators in queries/DMs. |
| `typing_show` | `typing_indicators` | `bool` | `true` | show inbound typing indicators above the input bar. |
| `upload_expire_hours` | `image_upload_expire_hours` | `int 0…8760` | `48` | hours before an image you upload to ReteCloud expires (0 never). Out-of-range values are clamped. Default 48. |
| `url_previews` | `url_embedding`, `embed_urls` | `bool` | `true` | show previews for links in messages (videos, music, images): Preferences' Embed URL previews. The macOS and iOS apps and ReteCross; reTerm shows none. Default true. |
| `use_unique_nick_color` | `unique_nick_color` | `bool` | `true` | deterministic colours per nick in buffers. |
| `user_info_display` | `user_info` | `short \| extended` | `short` | join/part/quit user info verbosity. |
| `window_transparency` | — | `number 0.2…1.0` | `1` | the main window's opacity (1.0 opaque): Preferences' Window transparency, 20–100 %. Out-of-range values are clamped. Default 1.0. On Linux it needs a compositing desktop. |

## Keys — Linux only

4 keys that exist only on Linux. They configure things the macOS app keeps in Preferences and the Keychain; in reTerm `/set` is the only way to reach them.

| Key | Aliases | Accepted values | Default | What it does |
|---|---|---|---|---|
| `chat_font_size` | `font_size`, `message_font_size` | `number 10…24` | `13` | the message font size in points, as Rete.app's Preferences slider: the chat, topic bar, input line, sidebar and nick list in ReteCross (Linux). Out-of-range values are clamped. Default 13. |
| `retecloud_password` | `cloud_password` | `string or off` | `(none)` | Linux ReteCloud password (Keychain `Rete.CDN` / authPassword; redacted in list). |
| `retecloud_username` | `cloud_username` | `string or off` | `(none)` | Linux ReteCloud username for token auth. |
| `tls_cert_path` | `tls_client_cert_path` | `string or off` | `(none)` | path to PEM client certificate+key for mTLS. A path that is set but cannot be read stops the connect rather than connecting without the certificate. |

## Keys — macOS only

2 keys that exist only on macOS. Same spellings as their Linux counterparts, different storage: these are the Keychain items Preferences edits.

| Key | Aliases | Accepted values | Default | What it does |
|---|---|---|---|---|
| `retecloud_password` | `cloud_password` | `string or off` | `(none)` | Rete Cloud password (Keychain `Rete.CDN` / authState, with the rest of the sign-in; redacted in list). |
| `retecloud_username` | `cloud_username` | `string or off` | `(none)` | Rete Cloud username (Keychain `Rete.CDN` / authState; same as Preferences). |

Neither set exists on iOS; its `/set` has the keys for all platforms.

## Keys each app handles itself

Most keys mean the same thing everywhere. A few are handled by the app you type them in: the key is spelled the same and stored the same way, but what it accepts, or what happens when it changes, depends on the app.

| Key | App | What it does there |
|---|---|---|
| `appearance` | macOS app | only the macOS app has it; the window follows it at once |
| `chat_font_size` | macOS app | the window follows it at once |
| `terminal_mouse` | reTerm | the terminal's mouse reporting switches at once |
| `theme` | macOS app | accepts the same list of themes Preferences shows |
| `theme` | reTerm | accepts the themes reTerm loaded |
| `window_transparency` | macOS app | the window follows it at once |

## Related documents

- [`Scripts.md`](Scripts.md) — the Tcl API. `::rete::*` commands read and write some of the same settings.
- [`Themes.md`](Themes.md) — what `/set theme` is choosing between.
