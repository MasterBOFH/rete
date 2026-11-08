v0.9 - 8 November 2026
====================
* **Commands:** a reference of every command, with its arguments, examples and which apps have it, is on the website.
* **iOS and bouncers:** connect to ReteCloud and to GoBNC bouncers at the same time, with a lighter
  connection that only streams the channel you are reading. Choose which networks this device shows,
  per network. A GoBNC bouncer with a self-signed certificate is trusted on first use and pinned; sign
  in with a password or a client certificate, which you can now create on the phone. A bouncer's
  address, name and sign-in can be edited later.
* **iOS buffer list:** every network shown on the device sits in one buffer list, each under its own
  collapsible header, and the app opens on the buffer you last read. A Connections pane chooses which
  networks the device shows and adds new ones; a buffer still syncing says so. With the list closed,
  its button counts the highlights and unread direct messages waiting in other buffers. A network's
  long press opens its Block list where the server supports one, and its nick colours.
* **Blocked users:** every nick list marks each user a Block list entry matches, wildcard entries
  included. On iOS, the Block list and the nick list's user details arrive with the connection.
* **iOS push:** push notifications on every backend, end-to-end encrypted. A buffer's Highlight switch
  now applies on every device.
  A burst of messages from one person becomes a single notification that counts them, and only the
  first in a minute makes a sound; Rete for Mac and Rete for Linux do the same with their own notifications.
* **iPad and landscape:** on an iPad, and on an iPhone in landscape, the buffer list and nick list sit
  beside the chat; swipe either away and back. In landscape the channel's name, modes and topic share
  one row.
* **Images:** tap an image to see it full screen, zoom and pan, and share or save it; Rete for Mac
  opens it in its own window.
* **Channel modes and lists:** on iOS, tap a channel's modes to view or change them, with its ban,
  exception, invite and quiet lists; on every app the channel lists are fetched fresh from the server
  when shown.
* **Topic:** edit a channel's topic on iOS where you may set it; `/topic -clear` clears a topic on
  every app.
* **Copying messages:** a copied line reads `[time] <nick> text`, the same on every app; on iOS, a long
  press copies a message or lets you select part of it.
* **iOS message log:** the phone keeps 60 days and 5,000 messages per channel or query by default,
  since your server holds the history; change either in Settings. On every app, `/set` can now also
  cap the log per channel or query.
* **Rete for Linux:** a new graphical app for Linux (GTK 4), with the sidebar, message pane, nick list,
  topic bar, desktop notifications and the same settings as reTerm.
* **ReteCloud on Linux, including image hosting:** sign in to ReteCloud from Rete for Linux's settings, and
  upload images from the input line: choose, paste or drop an image, add a comment and choose when it expires.
  Image upload now needs only a ReteCloud account, on every platform.
* **AI access:** let an AI agent such as Claude Code read your IRC sessions over a local connection
  protected by a token, and optionally send lines on your behalf. Off by default.
* **Log search:** logs are now kept in a database, and the Log Viewer can search every buffer of a
  network at once, and finds Rete's own status lines by the words they show. Copied log lines use a fixed 24-hour time. The Log Viewer shows messages in their
  colours, and log export can keep the IRC formatting codes.
* **Sensitive messages:** `/smsg` sends a message that Rete hides everywhere it shows or stores it,
  while the server receives it in full.
* **Reconnecting:** a connection that drops reconnects by itself, whether or not the network connects
  at launch.
* **Commands:** a `/command` Rete does not know is sent to the server, and a line that starts with a
  space is sent as text, keeping its leading spaces.
* **reTerm:** reorder and hide buffers and choose per buffer what is logged, highlighted and shown as
  activity; auto-away; mouse support and pasting several lines into the input line; a coloured nick
  list and buffer list; Tab completion for command arguments and settings; status markers in the nick
  list, folded join/part lines that open and close with a click or `/fold`, and unread counts drawn as badges.
* **reTerm keys:** the input line has a cursor and the usual terminal editing keys (Ctrl+A, Ctrl+E,
  Ctrl+U, Ctrl+K, Ctrl+W and more); Alt+Up and Alt+Down move the current buffer in the buffer list;
  Down on a typed line clears it and Up brings it back; `/help keys` lists them. The top of a transcript says when PageUp can load older messages.
* **reTerm rules:** `/colour` colours nicks and `/highlight` chooses which messages are highlighted, on
  the current network, with the same rules as Rete; a colour can be a name such as `yellow`.
* **Themes:** reTerm follows the theme in its header, buffer list, status bar and input line, and
  highlighted lines fade out as in Rete; Rete for Linux draws buffer-type icons in the sidebar, and
  Rete for Linux and iOS draw icon badges on join, part, quit, notice, info, topic and action lines; every app draws the
  selected buffer in the theme's selection colours; themes can colour the date shown between days; on iOS and Linux the topic bar shows its colours,
  with clickable links and channels, and iOS shows your own details above the server window.
  Every app follows the theme's colours, with the same defaults, and Rete for Linux its fonts too;
  the theme editor can set the invite, secondary and date colours. A theme no longer carries a light
  or dark preference: the app's appearance setting chooses.
* **Scrolling back:** scrolling up past the loaded lines reads older messages from your saved log, and
  then from the server where it keeps history, in every app. Rete and Rete for iOS mark the place
  where the server could not supply missing history. In Rete, sending a line while scrolled up returns
  to the newest messages, as on iOS.
* **Nick list (macOS and Linux):** drag its divider to set its width, which every channel keeps;
  double-click the divider to go back to automatic, which now fits the nicks in the channel. A long nick
  is shortened instead of wrapping.
* **iOS buffer list:** long-press a buffer to turn its logging, highlights and activity on or off, and to
  part, re-join or close it.
* **reTerm on macOS** can use the macOS app's client certificate.
* **Scripting:** several changes to `::rete::` commands are not backwards compatible — see
  [Scripts.md](Scripts.md).
* **Links and nicks:** addresses starting with `www.` are clickable links, and a nick written in parentheses
  or quotes, like (alice), is coloured and clickable. A link at the end of a sentence leaves the full stop,
  comma or bracket after it out of the link.
* **Settings with `/set`:** `/set` opens a table of every setting in Rete and Rete for Linux: click a value
  to change it in place. `/set` now reaches every setting in Preferences, including link previews, image
  loading, the media bar, appearance and font size.
* **Rete Cloud sign-in** is kept as a single keychain item. If you sign in with a username and password,
  you will be asked to sign in once more after updating.
* **Removed:** WeeChat relay connections.

v0.7 - 13 April 2026
====================
* Theme editor
* Configurable timestamp format
* Nick colouring configurable
* Auto-away
* Auto-completion suggestions for /commands
* Do not clear SASL config when disabled

v0.6pl2 - 18 March 2026
====================
* Fixed a bug causing long messages to become clamped
* Added support for SASL SCRAM-SHA-256
* Refactored how we track last activity of a user
* Added support for /LIST
* Changing fontsize will change the font size of the input bar too.

v0.6 – 10 March 2026
====================
* Initial release of full-functioning Rete.
* Major refactor since previous test builds.