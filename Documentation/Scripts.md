---
layout: doc
title: Scripts Tutorial
permalink: /docs/scripts/
---
This document describes Rete's Tcl scripting API: how to bind to events, how return
values influence client behaviour, what globals are available, and which helper
commands you can call from Tcl.

### Script bundles

**A script is a bundle: a directory containing a `manifest.json`.** That is the only shape
Rete loads. A bare `.tcl` file dropped into the scripts directory is never read, and neither
is a directory without a manifest — so the manifest is not optional metadata, it is what
makes your script exist.

Bundles live in the **scripts directory**, one directory per bundle:

| Platform | Scripts directory |
|---|---|
| macOS (`Rete.app` and `reTerm`) | `~/Library/Application Support/Rete/Scripts/` |
| Linux (`reTerm`) | `~/.rete/Scripts/` |

If you are not sure which one your build uses, `reTerm` prints it: `/script rehash` answers
`Script bundles rehashed from …`, naming the directory, and `/script list` names the same
one when it finds nothing there. Scripting requires Tcl, so it is available on `Rete.app`
and on `reTerm` (macOS and Linux); the iOS companion has no script engine.

**A worked bundle**

```
Scripts/
  hello/
    manifest.json
    main.tcl
    greeting.tcl
    hello_view.json
    README.md
```

`manifest.json`:

```json
{
  "id": "hello",
  "title": "Hello",
  "description": "Greets a channel, and shows how a bundle is put together.",
  "author": "you@example.net",
  "version": "1.0.0",
  "entrypoint": "main.tcl",
  "files": [
    { "filename": "main.tcl", "type": "tcl" },
    { "filename": "greeting.tcl", "type": "tcl" },
    { "filename": "hello_view.json", "type": "json" },
    { "filename": "README.md", "type": "md" }
  ]
}
```

`main.tcl` — the entrypoint, and the only Tcl file Rete evaluates:

```tcl
# hello — the entrypoint named by manifest.json.

namespace eval ::hello {
    variable loaded 1
}

# Sibling .tcl files are not loaded for you — source them yourself. The working
# directory is the bundle directory while the entrypoint runs, so a bare relative
# name is enough. Create the namespace first: `proc ::hello::greeting` in the
# sourced file fails with "unknown namespace" if ::hello does not exist yet.
source greeting.tcl

proc ::hello::say {nick} {
    return [::hello::greeting $nick]
}
```

`greeting.tcl` — an ordinary Tcl file, reached only through that `source`:

```tcl
proc ::hello::greeting {nick} {
    return "Hello, $nick!"
}
```

`hello_view.json` — a view, registered because the manifest lists it as `"type": "json"`
(see **Registering a view name**):

```json
{
  "name": "hello_view",
  "title": "Hello",
  "widgets": [
    { "type": "label", "id": "lblTitle", "props": { "text": "Hello" } }
  ]
}
```

Drop that directory into the scripts directory and it loads when a network first connects.
A network that is already connected does not pick it up by reconnecting — its interpreter
outlives the reconnect (see **How the entrypoint runs**): use `/script rehash` in `reTerm`, or
**Refresh** then **Load** in the macOS Script Manager.

**`manifest.json` fields**

| Key | Required | Meaning |
|---|---|---|
| `id` | no | The bundle's identifier, and its **namespace** for `::rete::store`. Omit it and the **directory name** is used. It also becomes the bundle's **folder name**, so it must be a single folder name — see *What an `id` may be* below. |
| `entrypoint` | no | The one Tcl file Rete evaluates. Omit it and the **first `files[]` entry with `"type": "tcl"`** is used, in manifest order. |
| `files` | no | Everything the bundle ships. Each entry needs **both** `filename` and `type`. |
| `title` | no | Display name in the macOS Script Manager and `/script info`. |
| `description` | no | One line of prose, shown in the same two places. |
| `author` | no | Shown in the same two places. |
| `version` | no | Shown in the same two places. Rete does not compare or enforce versions. |

`files[].filename` is relative to the bundle directory and **must stay inside it** (see *What
an `id` may be* below); `files[].type` is `tcl`, `json` or `md`. **Both keys are required on every entry** — one entry missing `type` fails the whole
manifest, and the bundle disappears (see below).

What `files[]` is actually used for is worth being precise about, because it is not loading:

- **`"type": "tcl"`** — supplies the entrypoint fallback, and populates the file list in the
  macOS Script Manager's editor. **Listing a `.tcl` file does not load it.** Only the
  entrypoint is evaluated; everything else is reached by `source`.
- **`"type": "json"`** — the file is read for view names when the bundle loads. A view file
  the manifest does not list this way is never read, so `::ui::view load` cannot reach it.
- **`"type": "md"`** — documentation for the humans; Rete does nothing with it.

**What an `id` may be**

The `id` becomes a folder inside your scripts directory, so it has to be usable as one. Rete
**refuses the install and tells you why** rather than repairing the id, because the id is your
bundle's Tcl namespace and the name every frontend lists it under — an id quietly changed to
something installable is a bundle you cannot find again.

An `id` is refused when it:

- is **blank**, or is only whitespace;
- contains **`/`** — including `../anything`, which would put the bundle outside your scripts
  directory altogether;
- is **`.`** or **`..`**;
- contains a **control character** — a NUL, a tab, a newline, an escape, a delete. A NUL cannot
  be passed to the operating system as part of a path at all, and a folder whose name holds a
  newline is invisible in a list and awkward to type at a shell. This applies wherever the id
  comes from, including a manifest whose JSON writes one as `\u000a` and a catalogue URL that
  percent-encodes one as `%00`: both arrive decoded, as a real control character;
- **begins with `.`** — a folder whose name starts with a dot is never enumerated, so the bundle
  would install and then never appear (the sixth row of *When a bundle does not appear* below);
- is **longer than 255 bytes** — the most a folder name may be. The limit is counted in bytes of
  UTF-8, as your filesystem counts it, so a character outside ASCII takes two to four of them and
  an id in a non-Latin script reaches it sooner; the refusal names the limit and the id's length in
  bytes. The id is refused, never shortened — a shortened id would be a different namespace.

Discovery applies the same limit, so a bundle copied in by hand whose manifest declares a longer
`id` is skipped with that reason.

The refusal names the id and says which of the rules above it broke. Nothing is installed and
nothing already installed is touched.

`entrypoint` and every `files[].filename` are paths **inside** the bundle, and the rule for them
is different: **`/` is allowed** — `src/main.tcl` is a supported layout, described under *How the
entrypoint runs* — but a leading `/` and any `..` component are refused, because they name a file
that is not part of the bundle. A *single file name* typed into the macOS bundle editor is held
to the id's rule instead, minus the dot-prefix clause: it is one path component, so `/` and
control characters are refused there too.

The `id` rule is applied wherever a bundle folder is chosen: importing a folder or `.zip`,
downloading from the catalogue, updating from the catalogue, `/script fetch`, **New Script**, and
renaming a bundle's namespace — so a namespace you type is held to the same rule as one a manifest
declares. The `entrypoint` and `files[]` rules are applied when a bundle is installed **and again when it
is discovered**, so a bundle copied into your scripts directory by hand is held to them too. Such
a bundle is not listed and not loaded: the directory is named with the reason, and **nothing the
manifest points at outside the bundle is read, parsed or run**. **The `id` rule is applied at
discovery too**, so a bundle placed in your scripts directory by hand is held to it as well. Such a
bundle is not listed and not loaded, and the directory is named with the reason. **Your `id` is never changed
for you**: it is your script's Tcl namespace and the key its `::rete::store` data is saved under, so
a silently corrected one would be a bundle you could not find.

**Uploading a bundle to the catalogue** (the macOS Script Manager's **Upload**) applies the
`entrypoint` and `files[]` rules once more, with the same reasons, because upload **sends** every
file the manifest lists. It also refuses a listed file that is a **link to a file outside the
bundle** — a rule install and discovery do not have. A link to another file inside the bundle is
fine, and so is a bundle that is itself a link (*Linked bundles*, below). Nothing is sent when an
upload is refused.

**How the entrypoint runs**

- Rete evaluates **the entrypoint and nothing else**. Split your script across as many files
  as you like and `source` them from the entrypoint.
- **Every network gets its own interpreter, and your entrypoint runs once in each.** Two
  networks running the same bundle share no Tcl variables, procs or state.
- **The interpreter lives as long as the network, not the connection.** It is built when the
  network first connects, and a reconnect — after a ping timeout, a dropped socket, a `/quit`
  and connect again, or a ReteCloud API link that drops and comes back — **keeps it**: your
  variables, procs, `bind` handlers and armed timers are all still there, and **your entrypoint
  does not run again**. Direct IRC and ReteCloud behave the same. Deleting the network, or
  closing its server, ends it: once the network has disconnected (your handlers still see the
  `QUIT` it sends), the interpreter and everything in it is gone.
- **What a reconnect does change is the connection your script sees.** When a direct-IRC
  connection ends `server-online` goes to `"0"`, and it is set again when the new connection
  registers (`bind REGISTERED` fires for each one). A script that keeps per-connection state —
  "have I joined my channels yet", "is the server up" — should reset it from those events
  rather than rely on its interpreter being new. On ReteCloud `server-online` follows the
  session's own events — see *Disconnecting on a ReteCloud connection*.
- **Timers keep running while you are disconnected**, on their own schedule. A timer that
  sends while there is no connection sends nothing; check `server-online` first if that
  matters.
- **Auto-load bundles are discovered when the interpreter is built**, which is the network's
  first connect. A bundle you drop into the scripts directory later is picked up by
  `/script rehash` (or **Refresh** in the macOS Script Manager), not by a reconnect. A rehash
  is the one thing that still builds a fresh interpreter — every bundle's state is lost and
  every auto-load entrypoint runs again.
- **Nothing runs for a network that has never connected.** Selecting a network you have not
  connected yet — to read its server buffer — gives it no interpreter, and a rehash leaves such a
  network alone; its auto-load bundles first run when you connect it. So an entrypoint that
  connects, joins or sends at load time only ever does so for a network you asked to connect.
- **While the entrypoint is running, the working directory is the bundle directory**, which
  is what makes `source greeting.tcl` resolve. Relative paths are relative to the bundle
  root, not to the entrypoint's own directory — an entrypoint at `src/main.tcl` sources its
  neighbour as `source src/lib/util.tcl`.

  That working directory is **your interpreter's**, not the process's. `pwd`, `cd`, `source`,
  `open`, `glob` and `file` all report and resolve against it (for `file`, the subcommands that
  take a path to something on disk — `exists`, `isfile`, `normalize`, `mkdir`, `delete` and the
  like; the string-only ones, such as `join`, `split` and `dirname`, never look at a directory),
  and every connection has its own, so two connections loading bundles at the same moment do not
  share one directory. A `cd` is not visible outside your interpreter.

  Anything that reaches the filesystem *without* going through those commands — `exec` and its
  redirections, `load`, and any compiled extension — sees the process working directory instead,
  which is some unrelated directory rather than your bundle. Give those an absolute path built
  from `$::rete::bundle_dir` (below).

- **`~` and `~/…` are your home directory, on every platform — and nothing else is.**
  `source ~/lib/util.tcl`, `open ~/notes.txt`, `glob ~/Scripts/*.tcl` and `file exists ~/x`
  all reach `$::env(HOME)`, inside a bundle load and outside one. **`~name` is an ordinary
  name that begins with `~`**, not another user's home: `open ~alice.log` opens a file
  called `~alice.log` in the working directory (the bundle directory, during a load), and
  `~root/x` is `x` inside a directory called `~root`. No user is ever looked up, so a name
  that matches no user is not an error — opening a file that is not there is. This is what
  lets a path built from IRC data stay where you put it: an ident is `~alice` whenever the
  user runs no identd, and `open [file join $dir ~alice.log]` opens `$dir/~alice.log`.
  `pwd`, `cd`, `source`, `open`, `glob` and `file` are the commands this covers — the same
  six as above. Anything reaching the filesystem another way (`exec`, `load`, a compiled
  extension) gets whatever the underlying Tcl does with a tilde, which is not the same
  thing on every platform.

  Rete expands the tilde itself, because the Tcl versions it runs on do not agree about it:
  **Tcl 8.6 expands `~` in the filesystem commands and Tcl 9.0 does not**, and macOS ships 9.0
  while Linux ships 8.6. Rete's rule gives one answer on both. On Linux the error for a missing
  `~name` file quotes the path as `./~name…`, and two kinds of `glob` call return their `~name`
  matches spelled `./~name…`: a single call that mixes a `~name` pattern with an absolute one,
  and `glob -path ~name…` outside a bundle load. All of these name the same files as on macOS.

  To name a file or directory that really is called `~`, write `./~`, as plain Tcl requires.

  **`~` and `~/…` are absolute paths, so `file join` starts again at them; `~name` is a
  relative name, so it joins like one.** `file pathtype ~/notes.txt` is `absolute`, and
  `file join $::rete::bundle_dir ~/notes.txt` is `~/notes.txt` — the bundle directory is
  discarded, exactly as it would be for `/tmp/notes.txt` — so
  `open [file join $::rete::bundle_dir ~/notes.txt]` opens the file in your home directory.
  `file pathtype ~alice.log` is `relative`, and `file join /x ~alice.log` is `/x/~alice.log`.
  Both commands stay string operations: they do not expand the tilde. A tilde that does not
  lead an argument is an ordinary character: `file join /x a/~/y` is `/x/a/~/y`. These answers
  are the same on macOS and Linux.

  The other string-only `file` subcommands — `dirname`, `tail`, `split`, `nativename`,
  `rootname`, `extension` — are passed to Tcl unchanged, and on a tilde path some of them
  differ between the two Tcl versions: `file tail ~` is `~` on Tcl 9.0 and the last part of
  your home directory on Tcl 8.6, `file nativename ~/y` is expanded only on 8.6, and on 8.6
  `file tail ~alice` looks the user up and raises an error when there is none. A file really
  called `~` differs too: on Tcl 8.6 `file tail ./~` and `file split ./~` spell that part `./~`
  where 9.0 spells it `~`, and `file join /x ./~` is `/x/~` on 8.6 and `/x/./~` on 9.0 — the
  same file either way. Build paths with `file join` around a tilde rather than taking a tilde
  path apart.

- **The working directory is restored as soon as the entrypoint returns.** A relative
  `source` (or `open`) from a timer, a `bind` handler or a UI callback runs later, with some
  other directory current, and will not find your file. Capture the directory while you
  still have it:

  ```tcl
  namespace eval ::hello {
      variable dir $::rete::bundle_dir
  }
  proc ::hello::reload_data {} {
      variable dir
      source [file join $dir data.tcl]
  }
  ```

  `::rete::bundle_dir` is **the bundle root, set while your entrypoint runs and empty
  afterwards** — one interpreter loads every bundle you have, so a value left standing would
  name whichever loaded last. Read it at load time, as above. `[file dirname [info script]]`
  also works, but it gives you *the file's own*
  directory: for an entrypoint at `src/main.tcl` that is `src/`, not the bundle root that
  relative paths resolve against.

- The namespace your `::rete::store` values are filed under is the bundle's `id` — the
  manifest's `"id"`, or the bundle directory's name when the manifest does not give one.
  Nothing in a `.tcl` file can change it. **Changing the `id` orphans everything the bundle
  has stored**, so pick one and keep it.

  A `# Namespace: <name>` comment in the entrypoint does nothing. A bundle that carries one
  and stored data under that name finds it again by setting the manifest's `"id"` to the same
  name.
- If the entrypoint raises, the load **fails loudly**: the bundle is not marked loaded, the
  buffers, binds, nicklist menu items and custom commands it had already created are taken
  back, and the error text goes to the **Script Debug** buffer (`/script debug on` in
  `reTerm`). This is the one failure you can see without going looking. **Timers are not
  taken back**, so a script that arms one before it can fail should arm it last, or kill it
  from an error handler.

**Auto-load**

A bundle auto-loads unless the user has said otherwise. The choice is stored in
**`autoload.json`**, a single JSON object in the scripts directory — beside the bundles, not
inside them:

```json
{
  "hello": false
}
```

- Keys are **bundle ids** (the manifest `id`, or the directory name where there is no `id`) —
  not directory names, and not filenames. Keyed wrongly, the entry silently does nothing.
- **An absent key means auto-load.** The file only ever records a decision that has been made;
  deleting it returns every bundle to loading.
- The macOS Script Manager's auto-load toggle and `reTerm`'s `/script load` / `/script unload`
  write this file. You can edit it by hand.
- **Installing a bundle you have never had writes `false` for it**, so a script never starts
  running because you downloaded it. Both catalogue installs do this — the macOS Script
  Manager's Download and `reTerm`'s `/script fetch`. It is the one place an entry appears
  without you having made a decision, and `/script load <bundle>` (or the toggle) is how you
  reverse it.
- **Re-installing a bundle you already have does not touch the entry.** Taking a newer version
  is not a decision about auto-load, so `/script fetch <bundle>` on an installed id leaves
  whatever you chose. `/script fetch --load <bundle>` is the exception: it switches auto-load
  **on** and loads the bundle immediately.
- A bundle switched off is still *installed*: it is listed by `/script list` and by the Script
  Manager, it just is not loaded at connect.
- **A file Rete cannot read is not a file with no choices in it.** If `autoload.json` is there
  and will not decode — a hand edit that went wrong, a truncated write — every bundle
  auto-loads, because an absent key means auto-load and an unreadable file has no keys at all.
  Rete says so, naming the file and the reason; the same notice appears when a choice cannot be
  written or the file cannot be removed. The next choice you make replaces the file, so fix or
  delete it before making another choice.

**Sharing a bundle**

Zip the bundle directory. Both shapes install correctly — the archive can hold
`manifest.json` at its root (zipped from inside the directory) or one wrapper directory
containing it (zipped from outside); a single wrapper level is flattened on install. **If any
file cannot be moved out of that wrapper folder, the install is refused** with an error naming the
file, and nothing is installed or changed — a bundle missing one of its files would otherwise
install and fail later, when a `source` could not find it. (Hidden files in the wrapper, such as
`.DS_Store`, are not moved and are not a reason to refuse.)

`reTerm`'s `/script fetch <bundle>` and the macOS Script Manager's catalogue install
archives this way, and extraction shells out to `unzip`, so a machine without it cannot
install from an archive — copying the directory in by hand always works.

**Links and files opened with Rete**

A **`rete-script://host/path/bundle.zip`** link — clicked in a channel, or opened from a browser —
installs the zipped bundle at `https://host/path/bundle.zip`, in Rete on macOS and on Linux, **only
after Rete asks you**:

- **Only an https `.zip`.** `rete-script://` always means https. A link whose path does not end in
  `.zip` (any case), or that has no host, is refused with a notice, and nothing is asked or downloaded.
- **The question comes before any network access.** It names the bundle (the link's file name), the
  host and the full address, says Rete will download and install it, and that a script can run code:
  use the network, read and change your files, and send on your IRC connections in your name.
  **Cancel is the default**; declining downloads nothing.
- **On Install Script**, the archive is downloaded over https only — a redirect to anything that is not
  https is refused — with no shared cookies, cache or credentials, at most **10 MB**, giving up after
  30 seconds without an answer or 2 minutes in all. It is then installed by the same importer as the
  Script Manager's Import (the archive rules under *Sharing a bundle* above and the manifest rules
  apply), and the downloaded file is deleted.
- **Nothing runs at install.** The bundle has no `autoload.json` entry, so it loads at the next connect,
  as an imported bundle does. Switch auto-load off in the Script Manager or with `/script unload` if you
  want to read it first.
- A failure — refused, too large, not a bundle, no network — is a notice naming the link and the reason.

A bundle folder opened *with* the macOS app from outside it (Finder's Open With, on a folder whose
name ends in `.tcl`) is installed only after the same kind of question, naming the folder and its
path. **Cancel is the default.** The Script Manager's own Import and the catalogue's Download ask
nothing more: you choose those in Rete's window.

**Where a bundle lands**

A bundle's folder is named after its id — the manifest's `"id"`, or, when the manifest gives
none, the name of the folder or archive it came from. Importing a folder called `hello copy`
whose manifest says `"id": "hello"` therefore installs `Scripts/hello`, and **replaces** an
existing `hello` rather than sitting beside it. If that id is already installed under some
other folder name, the new files go into **that** folder, so installing or updating never
leaves you with two folders claiming one id. And if `Scripts/<id>` exists but holds a
*different* bundle, **nothing is installed** — you are told which folder and which two ids,
and renaming or removing that folder is the fix.

**If `Scripts/<id>` is a plain file, or a folder that is not a bundle, the install goes ahead
and what was there is kept.** "Not a bundle" means a folder with no `manifest.json`, or one whose
`manifest.json` cannot be read as a manifest — the folders Rete never lists or loads (see *When a
bundle does not appear*, below). The file or folder is moved aside to a hidden name beside the new
bundle, `.<id>.rete-replaced-<uuid>` — for `hello`, something like `.hello.rete-replaced-1F0C…` —
and Rete never deletes it. Rename it to get it back. An existing bundle being updated — a folder
whose `manifest.json` reads, even if its entrypoint is missing — is replaced and its old copy
discarded, and a link is replaced as described under *Linked bundles* below.

If an install fails part-way and what was there cannot be put back, the error names the hidden
name it is under. For a bundle being updated that is a hidden install folder Rete removes once it
is an hour old, and the error says so; for a kept file or folder it is the
`.<id>.rete-replaced-<uuid>` name, which Rete never removes.

**Bundle ids ignore case.** `Hello` and `hello` are the same bundle. Installing,
updating or fetching one finds the other; the catalogue shows a bundle installed
under either spelling as installed and offers *Update*, not *Download*;
`/script load` and `/script info` match this way; and renaming a
namespace to a different spelling of its own id just rewrites the manifest
without moving anything.

**Uniqueness ignores case too, so `Hello` and `hello` are one id, not two.** Two
bundles claiming them are duplicates in the sense the next section describes:
one is kept and the other is **listed with a reason and not loaded**, exactly as
if both manifests had spelled the id the same way. This is reachable anywhere,
because a manifest may declare any `id` it likes regardless of its folder name —
`Scripts/alpha` claiming `Hello` and `Scripts/beta` claiming `hello` are one
bundle and a skipped folder, on every platform. On macOS the Scripts folder is
normally case-insensitive as well, so `Scripts/Hello` and `Scripts/hello` cannot
be separate folders there at all; on Linux they can be, and then the same rule
applies to them. Rename one. Nothing is deleted or merged: the skipped folder keeps its files,
its `autoload.json` entry and its stored data, and renaming it gives you two bundles again —
with everything saved under its old id still under it.

**All of that applies to `reTerm`'s `/script fetch` too.** Fetching an id you already have under
another folder name updates that folder, and fetching an id whose folder name is taken by a
*different* bundle is refused. The target is decided before anything is downloaded, and a refusal
prints the same sentence the macOS alert shows.

**What `/script fetch` tells you.** It answers `Fetched script "<id>".`, then `auto_load = …`
read back from `autoload.json` — so a re-fetch of a bundle you had switched on says
`auto_load = true`, even without `--load`. With `--load` it also answers `loaded = …`, and that
line is read from the bundles the interpreter actually holds: **a bundle whose entrypoint raises
reports `loaded = false`**, with the reason in the Script Debug buffer.

**A connection with no Tcl runtime still fetches.** Downloading is a disk operation, so it is
allowed on a build without Tcl — the bundle installs, `/script list` shows it, and a build or connection that has Tcl loads it.
Only the loading is withheld: `/script fetch --load` there prints *"Scripting is disabled…"* —
the same sentence `/script load`, `/script unload` and `/script rehash` refuse with — followed
by `loaded = false`.

**A failed install never costs you the bundle you had**

Installing, updating from the catalogue, `/script fetch` and importing a folder or `.zip`
all build the new bundle somewhere else first and only put it in place once it is complete.
**If anything goes wrong the bundle already installed is left exactly as it was**, at its old
version, still loadable — you get an error and nothing else changes. That covers the whole
list: no `unzip` on the machine, `unzip` failing, a download that was cut short, a folder
that could not be copied, a destination folder that turns out to hold a different bundle, an
archive whose wrapper folder could not be flattened whole (*Sharing a bundle*, above), and a
manifest whose `id`, `entrypoint` or `files[]` names something outside the bundle (*What an `id`
may be*, above).

The bar for "complete" is the same one discovery uses: the new bundle must have a
`manifest.json` that reads, that manifest must name a Tcl entrypoint, and **that entrypoint
file must actually be in it**. An archive that unpacks cleanly but holds no bundle is a
failed install, not a new bundle. Note the consequence for **importing by hand**: a folder
whose entrypoint is missing is *refused at import* with that reason, where a folder copied
into the scripts directory yourself is instead listed with an error (the fifth row below).

An install that is interrupted — the app is killed part way — can leave a working directory
behind in the scripts folder. Its name starts with a dot, so it is never mistaken for a
bundle and never reported as a problem, and the next install clears it away.

**When a bundle does not appear**

Discovery is strict. Each of the following stops the bundle auto-loading, and **both
frontends say which one happened** — except for the three that are silent on purpose:

| The bundle has… | What you see |
|---|---|
| no `manifest.json` | **nothing, deliberately** — a directory that never claimed to be a bundle |
| a `manifest.json` that is not valid JSON | the directory is named, with the decoder's complaint. macOS: a banner above the list. `reTerm`: an error line above `/script list` |
| a `files[]` entry missing `filename` or `type` | as above — the whole manifest fails to decode, however correct the rest of it is |
| no `entrypoint` and no `"type": "tcl"` in `files[]` | the directory is named, with *"names no Tcl entrypoint"*, in the same two places |
| an `entrypoint` naming a file that is not there | the bundle is **listed with an error** on both frontends, and still does not load |
| an `entrypoint` or a `files[]` filename that **leaves the bundle** — a leading `/`, or any `..` component | the directory is named, with *"it lists “…”"* or *"its entrypoint is “…”"* and the reason, in the same two places. **Nothing outside the bundle is read, parsed or run** |
| an `id` that is not usable as a folder name — blank, `.`, `..`, containing `/` or a control character, or beginning with `.` | the directory is named, with *"its manifest.json declares the id “…”"* and the reason, in the same two places. **This is the same rule and the same sentence installing the bundle would have given you** |
| a directory name starting with `.` | **nothing, anywhere** — hidden directories are never enumerated |
| an `id` another bundle already uses | the directory is named, with *"its id … is already used by script bundle …"*, in the same two places |
| a link to the same folder as another bundle | the link is named, with *"it is a link to the same folder as script bundle …"*, in the same two places — see *Linked bundles* below |
| a link to a file, a link to nothing, or a link loop | **nothing, deliberately** — like a directory with no `manifest.json`, it never was a bundle |

The scripts folder itself being unreadable is reported too, and replaces the "no script
bundles" message rather than being added to it: an empty list is what a user with no scripts
sees, and the two must not look the same. That covers the path **not being a folder at all** —
a file, a link to a file, or a link to nothing — and the message says which. A Scripts folder
that is itself a **link to a folder** is followed, and lists the bundles in the folder it points
at. A folder that is genuinely empty is not an error and is not reported: you are simply told
there are no bundles.

**Linked bundles.** A bundle in the Scripts folder may be a symbolic link to a bundle folder kept
somewhere else — a checkout, a synced folder. The link is followed wherever it points, **including
outside your home folder**, and the bundle is listed and loads exactly as if the folder were in
place:

- **Its id is the manifest's `id`**, as for any bundle. A manifest that declares no `id` takes the
  **link's** name — the name you see in the Scripts folder — and never the name of the folder it
  points at.
- **A bundle is loaded once, however many names reach it.** Two links to one folder, or a link
  beside the real folder it points at, are one bundle: one is listed and the others are named
  with *"it is a link to the same folder as script bundle …"*. **The real folder wins over a
  link**, because it is the bundle's own folder — an update is then written into it and every link
  to it stays valid. Between links, the link whose name is the id wins, and otherwise the
  alphabetically-first name, the same fixed rule as for a shared id above.
- **A bundle reached by more than one link should declare an `id` in its `manifest.json`.**
  Without one, its id is the name of whichever link wins — so adding another link with an
  earlier name (`a-tools` beside `tools`) changes the id, and the bundle's auto-load setting and
  `::rete::store` data stay behind under the old one. A declared id does not depend on which
  link wins. The skipped link's message says the same.
- **The link's own name decides whether it is hidden.** A link whose name starts with a dot is
  never enumerated, wherever it points.
- **Importing a folder that is a link imports the folder it points at.** Its files are copied
  into your Scripts folder as an ordinary bundle folder, and the linked folder is left as it was.
- **Removing a linked bundle removes the link.** Delete and uninstall take away the link in your
  Scripts folder and never touch the folder it points at. **Updating** one from the catalogue
  installs the new version as an ordinary folder in place of the link, and likewise leaves the
  linked folder exactly as it was: an install never writes through a link. (Editing the bundle's
  files in the macOS Script Manager does change them where they live, in the linked folder —
  which is what a link is for.)
- **An install over a name that is some other link replaces the link the same way.** If the
  folder name a new bundle installs under is already taken by a link that is not a bundle — a link
  to nothing, to a file, or to a folder with no `manifest.json` — the install puts the bundle there
  as an ordinary folder in place of the link, and never touches, creates or removes anything the
  link pointed to. Such a link is not listed (see the table above), so this is the only way the
  install would ever meet it. If the install fails, the link is left exactly as it was.

**Bundle ids are unique, and one bundle keeps the id.** Two bundles can claim the
same id two ways: both manifests declare it, or one declares it while the other
omits `"id"` and inherits it from its directory name — so the directories need not
look alike. **Uniqueness ignores case, so `Hello` and `hello` are one id, not
two.** Duplicates do not stack: `::rete::store` data, the auto-load
preference in `autoload.json`, and the engine's loaded-script table are **all
keyed by the id**, so two bundles sharing one would read each other's saved data
and unload together.

Discovery therefore keeps exactly one and reports the rest. **The bundle whose
folder name is the id wins; otherwise the alphabetically-first folder does** —
a fixed rule on purpose, because a winner that changed between refreshes would
make a script's stored data appear and disappear. Copying a bundle folder is the
usual way in: rename the copy's `id` in its `manifest.json` before loading it.

**Nothing is merged and nothing is thrown away.** The folder that loses keeps its
files exactly where they are, its own `autoload.json` entry, and its own
`::rete::store` data — none of it is copied onto the winner and none of it is
deleted. It is simply not read while the winner is there. Rename the losing
folder's `id` and it comes back with everything it had, which is why the rule is
safe to state as *"one of these is skipped"* rather than *"one of these is
lost"*.

`/script load <name>` and `/script info <name>` answer the same way. A name that matches a
directory which cannot be used gives the reason above; a name that is the start of more than
one bundle's name lists the bundles it matches, so you can type more of it; *"Unknown script
bundle"* means what it says — nothing on disk matches that name. `/script info` will still describe a bundle
whose entrypoint file is missing, since its manifest is readable.

If a bundle you installed is not there and nothing was said about it, it is one of the three
deliberate silences: check that the directory contains a `manifest.json`, that its name
does not start with a dot, and — if it is a link — that it points at a folder that exists.

**An install never destroys the bundle it is replacing** — the replacement is built in a hidden
staging folder and swapped in only once it is known to be a bundle, so a download that fails,
an archive that will not extract, or a `.zip` with no manifest all leave what you had exactly
where it was.

There is one failure that design cannot prevent: the swap succeeding halfway, so the bundle you
had is neither back in place nor replaced. The error then says so, and names the hidden folder
it is in — `.rete-install-<uuid>-previous`, in your Scripts folder. **Rename that folder back to
the bundle's folder name to recover it**, and do it promptly: Rete removes leftover install
folders once they are an hour old.

Note the asymmetry in the fifth row: a bundle whose entrypoint file is missing is *listed*,
with the error on its row, but auto-load skips it like the rest. A listed bundle is not
necessarily a loadable one.

### The `/script` command

`/script` manages bundles from the input line. **It is the same command in `Rete.app`, `reTerm` and
Rete on Linux**, with the same answers, written to the server buffer of the connection you typed it
on. On iOS, which has no Tcl, it only says that scripting is not available there.

| Subcommand | What it does |
|---|---|
| `/script list` | Lists the installed bundles, each with `loaded` (on this connection) and `auto_load`, and any bundle that will not load with the reason. |
| `/script info <bundle>` | The bundle's manifest fields, `loaded` and `auto_load`. Where this connection cannot run scripts, it says why in place of `loaded`. |
| `/script catalogue` | The ReteCloud script catalogue, marking what you have installed. |
| `/script fetch [--load] <bundle>` | Installs a bundle from the catalogue; `--load` also switches it on and loads it (see **Sharing a bundle** above for what it answers). |
| `/script load <bundle>`, `/script unload <bundle>` | Switches the bundle's auto-load on or off and loads or unloads it, then answers `script = …` and `loaded = …`. |
| `/script rehash` | Re-reads the Scripts folder: each interpreter loads what auto-load allows. |
| `/script debug on\|off` | Shows or hides the Script Debug buffer. |
| `/script init <namespace> [title…]` | Starts a new bundle — the same as **New Script** in the Script Manager, with the same rules for the namespace and the same refusals. It creates `<namespace>/` in your Scripts folder with an empty `<namespace>.tcl` as its entrypoint and a `manifest.json` (the rest of the line is the title), then tells you where it is and to load it with `/script load <namespace>`. A namespace that is already installed is not touched. |

`Rete.app` and Rete on Linux load a script into **every** connection's interpreter, so there
`/script load`, `/script unload`, `/script rehash` and a `/script fetch` of a loaded bundle act on
all connections, exactly as the Script Manager's buttons do — and a bundle the Script Manager is
busy with (an update, say) answers that it is busy rather than being changed underneath it. The
`loaded = …` they print is what the interpreters answered. In `reTerm` each connection has its own
interpreter, and the command acts on the one you typed it in. `/script debug` is the **Show Script
Debug** toggle in `Rete.app` and Rete on Linux, and selects the Script Debug buffer in `reTerm`.

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
    # return 1 to suppress the line this event would have produced
    # return 2 to suppress the line *and* the state change it would have made
}
```

#### Return convention

> **Every handler returns `{code ?data…}`** — a control code, optionally followed by
> data. The first word is *always* the control code. A bare `return 0` / `return 1` /
> `return 2` is a one-element list in Tcl, so the common case is just an integer.

- **Return `0`** → Rete continues processing (other Tcl handlers, then its own logic).
- **Return `1`** → Rete stops processing this event in Swift: no further handlers
  are run, and **the line the event would have produced is neither displayed nor
  logged**. Logging follows display — the log records what was written to a buffer,
  not what arrived on the wire — so one `return 1` suppresses both. Use the raw
  buffer to see traffic the buffers did not keep. **The session model still tracks
  the server**: membership, nick, topic and mode state are unaffected.
- **Return `2`** → as `1`, and Rete additionally **skips the state change**. You are
  taking ownership of the event, and any resulting divergence from the server is
  yours. Vetoing `JOIN` with `2` leaves the joiner out of the nick list, so a later
  op or voice change for them is not tracked either (the `MODE` line itself still
  shows) until the nick list is next refreshed from the server; vetoing `QUIT` with
  `2` keeps a departed user in it.
- A **missing** return value — a handler that ends with a bare `return`, or with a
  command that produces nothing — is `0`, silently. This is the normal case.
- A return value whose first word is **not** `0`, `1` or `2` is a **logged script
  error**, reported once per handler in the script debug buffer, and treated as `0`.
  It does not veto anything: a handler that cannot say what it wants does not get to
  suppress an event.

**Mind what your handler's last command returns.** A Tcl `proc` evaluates to the
value of its last command, so a handler ending in `set ::last $text` returns the text
and will be reported as a malformed control code. End handlers with an explicit
`return`.

Outbound events are the exception: for **`RAW_OUT`** and
**`MSG_OUT`** there is no line to hide and no state to keep, so `1` and `2` mean the
same thing — *do not send this line*.

**Data after the code.** The words after the control code are yours; the only event
that reads them is `SASL_AUTH` (below), which takes `{0 username password}`. A
`SASL_AUTH` handler that returns `{username password}` without the leading code applies
no override, and says so in the script debug buffer. Handlers for every other event
should return a bare code.

**`return 1` never prevents a state change**, on any inbound event — `QUIT`, `TOPIC`,
`MODE` and a `JOIN` inside a ReteCloud netjoin batch included. To prevent one, use
`return 2`.

#### When a handler does not fire

Some events are skipped before dispatch, so no bound handler runs at all:

- **Your own `QUIT`.** `bind QUIT` fires for other users only. Use the disconnect
  path (or `RAWIN`) if you need to see your own.
- **Parting a channel whose buffer you have already closed.** When *you* part a
  channel whose buffer is hidden, Rete removes the buffer and fires no `PART`.
- **Replayed and historical rows never reach a handler at all.** Chathistory pages,
  ReteCloud replay and log reload are shown but not dispatched, so scrolling back
  through 500 joins does not fire `bind JOIN` 500 times.

**A line you hide counts as seen.** When a handler hides a message, notice, action, join,
part, kick, topic or mode line with `1` or `2`, the chathistory Rete fetches from the server
after a reconnect starts after that line, so it does not come back as a replayed row (which no handler could
hide). A hidden `QUIT` or `NICK` line is the exception: if the server sends it again in that
history, it is shown.

One event accepts a control code and does nothing with it: **`REGISTERED`** appends
no line, and the connection bookkeeping around it is not a server fact you can
decline, so `1` and `2` are both no-ops there. Use `ENDOFMOTD` for post-connect work
in any case.

**An event whose prefix carries no `user@host` still fires.** A prefix that is a bare
nick or a servername is legal (RFC 2812 §2.3.1 makes both the user and the host
optional), and bouncers and services replays send them routinely. `JOIN`, `PART`,
`QUIT`, `ACCOUNT`, `AWAY`, `CHGHOST`, `INVITE` and `NICK` all fire for such a prefix.
**`bind JOIN` is the only one of these whose arguments include `user` and `host`, and
it receives them as empty strings** when the prefix did not carry them — a handler
that assumes they are non-empty needs a guard.

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

`RAWIN` fires for live lines only. Replayed and history rows — chathistory pages
on direct IRC, ReteCloud reconnect replay and scroll-up history — are shown in
the raw buffer but are not dispatched, like every other event.

**RAW_OUT** (outgoing raw line)
```tcl
proc my_raw_out {line} {
    # line: raw line about to be sent to the server
    # return 0 to allow send, 1 or 2 to block it (outbound has no line and no
    # state, so the two codes mean the same thing here)
    return 0
}
```

**`RAW_OUT` sees every line exactly as it goes on the wire, secrets included.** A server
password (`PASS`), SASL `AUTHENTICATE` payloads, the password of `OPER`, and the service logins
Rete masks everywhere else — NickServ `IDENTIFY`, `ID`, `REGISTER`, `GHOST`, `RECOVER`,
`SET PASSWORD`, `GROUP` and `DROP`, X (`x@channels.undernet.org`) `LOGIN`, and Q
(`Q@CServe.quakenet.org`) and AuthServ `AUTH` — reach the handler **unmasked**, whoever sent the
line: you (`/quote`, `putserv`, `/oper`, a typed `/msg NickServ IDENTIFY …`, `/smsg`,
`::rete::msg -sensitive`), Rete itself while it connects, or an AI agent through `send_raw`, on a
direct IRC connection and a ReteCloud one alike.
Returning 1 or 2 blocks the line.

> **Warning.** A `RAW_OUT` handler that logs, stores or forwards the lines it sees **will log,
> store or forward your passwords** — your server password, your SASL password, your oper password
> and your services password — and the text of every `/smsg`. Leave those lines out before writing a line anywhere.

If a `RAW_OUT` handler raises an error, or returns something that is not `{code ?data…}`, for a line
that carries a secret (`PASS`, `OPER`, `AUTHENTICATE`, or a service login in any spelling Rete
masks), the report in the script debug buffer does not quote the error message or the returned
value: it says `<not shown: the RAW_OUT line it was given carries a secret>`. For any other line the
report quotes them. A handler that writes the line somewhere itself — a file, a buffer,
`putlog` — still writes the password; Rete cannot stop that.

**Everything else Rete shows or keeps stays masked** — except a line you send literally, below.
The raw buffer, the transcript log, Rete's own log, and everything AI access can read — `raw_since`, a `send_raw` line's `AI access sent:` row
and the AI access log — show `PASS :********`, `AUTHENTICATE ********`, `OPER <name> ********` and,
for a service login, only the credential masked: `PRIVMSG NickServ :IDENTIFY ********`. The
mechanism lines `AUTHENTICATE PLAIN`, `AUTHENTICATE EXTERNAL` and `AUTHENTICATE SCRAM-*`, and the
bare `AUTHENTICATE +` and `AUTHENTICATE *`, are not secret and are shown as sent; **any other
`AUTHENTICATE` is masked**, including a mechanism name Rete does not list and an `AUTHENTICATE`
with more than one argument. Rete reads a line's command **after any IRCv3 `@tags` word and
`:prefix`**, so `@label=1 PASS :secret` is masked as `PASS :********` (the tags are not shown in the
masked form), and the flood throttler treats it as the `PASS` it is. On a ReteCloud
connection the masking goes by what a line is rather than who sent it: an `OPER` with a single
parameter is masked whole, and a service login after a `:prefix` is masked too.

**A line you send literally is shown literally in your raw buffer.** On a direct IRC connection a
line typed with `/quote`, typed into the raw buffer, or sent with `::rete::putserv` is shown in the
raw buffer exactly as it was sent, a password included: `/quote PASS :secret` shows
`PASS :secret`. Rete masks a credential by how you sent it, so `/pass secret`, `/oper <name>
<password>`, `/ns identify <password>` and `/msg NickServ IDENTIFY <password>` are shown masked, and
the same credential typed with `/quote` is not. What AI access reads (`raw_since`) is masked whichever
way the line was sent, and on a ReteCloud connection the raw buffer masks it too, by what the line
is.

**Lines an AI agent sends pass through `RAW_OUT` too.** When AI access lets an agent send
(`control.send`, see [`Control.md`](Control.md#sending)), each line it sends with `send_raw` goes
the way a `/quote` line goes, so your handler sees it before the server does, and **returning 1 or
2 vetoes it**: nothing is sent, the agent is told `line not sent: a script blocked it or the
connection could not take it`, and no `AI access sent:` row is written.

**But a handler cannot tell an agent's line from yours or from Rete's own.** `RAW_OUT` carries
the line and nothing about who sent it, so a handler written to restrict an agent restricts you
and Rete as well. **Blocking everything but `WHO` also blocks the `PONG` Rete answers the server's
`PING` with**, and the server drops the connection a few minutes later — and your own messages,
`JOIN`s and `QUIT` with it. Block specific commands instead, reading the command after any tags
and prefix as Rete does:

```tcl
# Refuse KICK, MODE and QUIT from anyone — an agent, a script or you.
proc block_risky {line} {
    set words [split $line " "]
    if {[string index [lindex $words 0] 0] eq "@"} { set words [lrange $words 1 end] }
    if {[string index [lindex $words 0] 0] eq ":"} { set words [lrange $words 1 end] }
    if {[string toupper [lindex $words 0]] in {KICK MODE QUIT}} { return 1 }
    return 0
}
::rete::bind RAW_OUT block_risky
```

To stop an agent sending altogether, switch `control.send` off rather than writing a handler.

**`RAWIN`, unlike `RAW_OUT`, receives your own service logins masked** when they come back from the
server live — echoed under `echo-message`, or played back live by a bouncer. A `PRIVMSG`/`NOTICE`
sent by your own nick that is a service login arrives as, for example,
`PRIVMSG NickServ :IDENTIFY ********`, with its tags and prefix unchanged; the `DIRECTMSG`,
`CHANMSG`, `DIRECTNOTICE` and `CHANNOTICE` handlers, the buffer row and the log see the same
masked text. History replayed through chathistory is masked in the buffer row and the log too,
but — like every replayed line — it fires none of these handlers. **Lines from other users are
never altered.**

**MSG_OUT** (outgoing message)
```tcl
proc my_msg_out {target text} {
    # target: channel name or nickname the message is addressed to
    # text: the message as you composed it, before Rete splits it for the wire
    # return 0 to allow send, 1 or 2 to block it (outbound has no line and no
    # state, so the two codes mean the same thing here)
    return 0
}
```

`MSG_OUT` fires once for each message **you** send, before anything reaches the wire.
Exactly two things reach it:

| What you did | `target` | `text` |
|---|---|---|
| typed plain text in a channel or DM buffer | that buffer's name | exactly what you typed, spacing included |
| typed `/msg <target> <message…>`, from any buffer | the first word after `/msg` | the remaining words **joined by single spaces** |

**Typed text loses its leading and trailing blanks before it is sent — with one exception.** A line
that starts with a space or tab and whose first other character is `/` (`␠/msg NickServ IDENTIFY pw`)
is not a command: it is sent to the buffer as text **with its leading whitespace kept**, so `text` is
`" /msg NickServ IDENTIFY pw"`, and it fires `MSG_OUT` for that buffer, not for NickServ. Trailing
whitespace is still trimmed.

That second row is the one behavioural difference between the two, and it is not
cosmetic: the `/msg` form is tokenised before the event fires, so `/msg #chan  a   b`
reaches your handler as `a b`. The plain-input form is not tokenised at all and
arrives byte for byte. If your handler compares `text` against something the user
typed, only the plain-input form will match runs of whitespace.

**One `MSG_OUT`, possibly several wire lines.** `text` is the whole message. Rete
splits it on newlines and chunks each line to fit the IRC line-length limit *after*
the event, so a two-line paste fires `MSG_OUT` once and `RAW_OUT` twice, and a
1,200-character message fires `MSG_OUT` once and `RAW_OUT` three times. Match on `text`; do
not assume it is one PRIVMSG. Each chunk leaves room for the `:nick!user@host ` the server
puts in front of it when it relays it (a lower `LINELEN`, when the server advertises one,
is honoured too), so the recipient receives every chunk whole; how many bytes a chunk holds
therefore depends on your nick and host, and is never more than 510 less that prefix.

**Blocking it.** `return 1` and `return 2` are the same thing here — *do not send
this message*. A blocked message is not sent, **not echoed into your own buffer, and
not logged**: nothing records that you tried. `RAW_OUT` does not fire for it either,
because `MSG_OUT` sits upstream of the wire line.

**Only the control code is read.** Words after it are accepted and ignored — a
handler returning `{0 something else}` sends the original message unchanged:
returning a replacement message changes nothing.

**What does not fire `MSG_OUT`**, although it puts a `PRIVMSG` or a `NOTICE` on the
wire:

- **`::rete::msg`** — see below.
- **`/me`** (a CTCP `ACTION`, which is carried in a `PRIVMSG`), **`/notice`** and
  **`/ctcp`**. `MSG_OUT` is "the user sent a message", not "a PRIVMSG went out"; for
  the latter, bind `RAW_OUT`.
- **Text typed into the raw buffer**, which is sent as a raw command.
- **`/smsg`**, which sends a message whose text Rete masks everywhere it shows or
  stores it. It does not fire `MSG_OUT`; `RAW_OUT` still fires, once per wire line,
  and — like every sensitive line — it receives the **real** text, as the server
  does (see `RAW_OUT` above). Everything else shows `PRIVMSG <target> :********`.
- **Messages a frontend composes for you** rather than taking from the input bar —
  the macOS now-playing share and the image-upload links go straight to the
  transport. `RAW_OUT` sees these; `MSG_OUT` does not.
- **Plain text in a buffer that cannot take it** — a system buffer answers with an
  error, a script buffer routes the text to its own input callback, and neither
  dispatches anything.

**`::rete::msg` never fires `MSG_OUT`.** This is deliberate, so that a handler can
send a message from inside its own `MSG_OUT` bind without re-entering it. The bypass
is unconditional rather than reentrancy detection: `::rete::msg` fires no `MSG_OUT`
from a `RAWIN` handler, or from the top level of a script, either. The line it sends
does still reach `RAW_OUT`.

> **Ordering is not guaranteed.** `::rete::msg` queues its send while the message
> that triggered your handler is still on its way out, so a reply sent from inside an
> `MSG_OUT` handler can reach the server *before* the message it is replying to. If
> the order matters, do not send from inside the handler.

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
    # text: an error line from the IRC server
    return 0
}
```

`ERROR` is the **IRC server's** error event. It fires once for each error line the
server sends that Rete is about to write to the server buffer, and `text` is that line
exactly:

| Where the error came from | What `text` holds |
|---|---|
| The server's `ERROR` command — direct IRC, and ReteCloud when the hosted session relays one | the command's trailing parameter, **verbatim**: `Closing Link: 1.2.3.4 (Ping timeout)` |
| An IRCv3 `FAIL` standard reply | `FAIL <command> <code> [context…]: <description>` — Rete composes that line out of the reply's parts, so it is not a wire line. `WARN` and `NOTE` are not errors and fire nothing |

It also fires for the few error lines Rete writes about one message or request rather
than about the connection: a message it received but could not read, a message the
ReteCloud server would not send, a history page that did not load, a saved SASL password
it could not read. Their wording is not part of this API and may change.

**The connection itself failing never fires `ERROR`, and no script can hide it.** On a
ReteCloud connection, failing to connect, authenticate, create or sync the session, or
to take it off IRC, writes an error line to the server buffer (`ReteCloud authentication
failed: …`, `ReteCloud API link failed: …`, `ReteCloud history sync failed: …`,
`disconnect_session failed: …`, `Cannot /quit: …`, and so on). That line fires no
event and is always shown, because it is often the only thing that tells the user the
session stopped. On a direct IRC connection a lost or refused socket fires no event
either, so a script sees the same thing on both.

If you want the server's `ERROR` command and nothing else, bind `RAWIN` and look for it
there.

`return 1` suppresses the line. `2` does nothing `1` does not: the event carries no
session state of its own.

Not an `ERROR` event: the error lines your own `/commands` produce (*"Usage: /join
#channel [key]"*, *"Unknown command: /exit"*). Those are written straight into the
buffer as errors and dispatch nothing. (A `/command` Rete does not know is **not** an
error: it is sent to the server as typed — see **Custom commands** — and what the server
answers arrives like any other server line.)

#### Disconnecting on a ReteCloud connection — three verbs, two meanings

`disconnect_session failed: …` (see **ERROR** above) is a **refusal**, and which verb
the user reached for decides whether a refusal was even possible. A ReteCloud
session has two things that can be taken down and they are independent — the
session's attachment to IRC, and this client's link to the API — and the three
controls make only two different choices between them:

| the user does | session taken off IRC? | this client's link closed? |
|---|---|---|
| `/quit [msg]` | **yes** | no |
| `/disconnect [msg]` | no | **yes** |
| **Disconnect IRC** (sidebar / menu) | **yes** | no |

`/quit` and `/disconnect` are the exact reverse of each other, which is the
opposite of what they mean on a direct IRC connection. **Disconnect IRC does what
`/quit` does**: it asks the server to take the session off IRC and leaves this
client's link up. `/disconnect`
asks the server for nothing, so it can never produce a `disconnect_session failed:`
line.

**What this means for the user.** A `disconnect_session failed: …` line on the server
buffer means **the session is still on IRC** — the server declined to detach it. It
fires no script event and no script can hide it.

**A refusal changes nothing a script can see except that one line.** The line reads
`disconnect_session failed: … The session is still connected to IRC, and this client
is still attached to it.` — the second sentence is part of the same line. Neither `/quit` nor Disconnect IRC closes this client's WebSocket, whatever the
server answers, so a refusal leaves `server-online` unchanged and delivers no close to
the interpreter. The one failure that closes the link is the link itself dropping under
the request — then the line has no second sentence and `server-online` goes to `"0"`
as for any link loss.

**`server-online` is not *"is the session on IRC"* on this transport.** It goes to
`"0"` whenever the client handles a close: the link going down, an acknowledged
`/quit` or **Disconnect IRC**, or the server reporting the session off IRC with a
`state` frame. So after `/disconnect` it is `"0"` while the session is still on IRC,
and after an acknowledged `/quit` or Disconnect IRC it is `"0"` while the link is still
up. If your handler needs to know whether the session is on IRC, `server-online` does
not answer that here.

**An acknowledged Disconnect IRC and an acknowledged `/quit` are the same event to a
script**. Both write the disconnect notice, clear the session's membership and
modes, take `server-online` to `"0"`, and make the client resync the session when it
next comes back on IRC. When there is no live API link to ask over, neither sends
anything and both say so on the server buffer — `Cannot /quit: …` and
`Cannot disconnect from IRC: …` respectively, neither through a script event — and the
session is left exactly as it was.

On a direct IRC connection none of this applies: the socket is the session, `/quit`
and `/disconnect` both end it, and no line of this shape exists.

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
    # credentials, or return {0 newUsername newPassword} to override what is sent
    # to the server. The leading 0 is the control code every handler returns; the
    # two words after it are the data. A bare {newUsername newPassword} with no
    # control code applies no override and is reported in the script debug buffer
    # as a malformed control code.

    # Example: override username and password
    set newUser "my sasl user"
    set newPass "my secret pass"
    return [list 0 $newUser $newPass]
}

bind SASL_AUTH my_sasl_handler
```

The handler runs once per authentication attempt, when the server asks for the
credentials; a mechanism that takes several rounds (SCRAM-SHA-256) does not run it again
for the later rounds. The handler is given the real password, and the password it returns
is the one Rete authenticates with. If a `SASL_AUTH` handler raises an error, or returns something that is not
`{code ?data…}`, the report in the script debug buffer names the handler but does not quote the
error message or the returned value, whether or not a password was configured: it says
`<not shown: a SASL_AUTH handler is given and may return the SASL password>`. The script debug
buffer is kept in the transcript, and either text could hold a password. A handler that writes
the password somewhere itself — a file, a buffer, `putlog` — still writes it; Rete cannot stop
that.

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

Netsplit batches (ReteCloud) dispatch one `QUIT` per split nick, so a batched quit
reaches your handler exactly as an individual one does, and a veto applies to that
nick alone. `message` is the pair of server names, which is what a direct IRC
connection puts in the quit reason.

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
    #
    # Returning 1 suppresses the nick-change lines in every channel, in the DM
    # buffer and (for your own nick) in the server buffer, and leaves the rename
    # itself alone. Returning 2 declines the rename as well.
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
    #
    # Returning 1 suppresses the mode line, and 2 also suppresses the setter's
    # last-activity stamp. Neither affects the mode state itself: op/voice and
    # channel modes are applied ahead of this event and are not vetoable.
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

**Bundle variables**:

- `::rete::bundle_dir` – the directory of the bundle whose entrypoint is running, and `""`
  at every other moment. It is the directory relative paths resolve against
  during the load, so `[file join $::rete::bundle_dir data.txt]` names the same file as a
  bare `data.txt` does there — and keeps naming it from a timer or a `bind` handler, where a
  bare relative name does not resolve. Capture it at load time; do not read it later. See
  **How the entrypoint runs**.

**Identity variables**:

- `mynick`   – your current nickname
- `myuser`   – your user/ident
- `myhost`   – your host
- `myaccount` – your services account name (or `""` if not authenticated)

**Server variables**:

- `server` – IRC server name from **004 RPL_MYINFO** (falls back to the 001 prefix until 004 arrives). Prefer this over `serveraddress` when matching networks (e.g. behind ZNC).
- `serveraddress` – configured connect host and port (`host:port`), i.e. the bouncer or direct endpoint you connected to
- `serverdaemon` – the detected IRC daemon type (e.g., `"ircu"`, `"ircd"`, `"ratbox"`, `"hybrid"`, `"bahamut"`, `"unrealircd"`, `"inspircd"`, `"solanum"`, `"snircd"`, or `"unknown"`)
- `server-online` – unixtime when the current connection registered, or `"0"` if disconnected. The interpreter outlives a reconnect, so this is how a script tells one connection from the next

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
matching). This is the comparison Rete itself uses for nicks and channel names, and it
respects the server's `CASEMAPPING` ISUPPORT token.

Returns `1` if the strings are equal (case-insensitive, with RFC 1459 character
equivalences: `{}|~` == `[]\^`), `0` otherwise.

Only ASCII letters and those characters are folded; every other byte must be the same, with no
Unicode normalisation, as a server compares nicks — so a precomposed `é` and `e` followed by U+0301
(a combining accent) are **not** equal.

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

- **Buffer-list icons** (returns SF Symbol names): `bufferListChannelIcon`,
  `bufferListDirectMessageIcon`, `bufferListSystemIcon`, `bufferListRawIcon`,
  `bufferListWebIcon`, `bufferListScriptIcon`, `bufferListScriptDebugIcon` — the symbol drawn
  beside a buffer of that kind in the sidebar. See *Buffer-list icons answer with the effective
  symbol* below, because these are the only options that can answer with an empty string for a
  reason other than "no such option".

- **UI colors** (returns hex color strings): `background`, `foreground`, `accent`,
  `divider`, `topicBarBackground`, `topicBarForeground`, `badgeBackground`,
  `badgeForeground`, `sidebarSelectionBackground`, `sidebarSelectionForeground`,
  `highlightRowBackground`, `expandedGroupRowBackground`

- **Badge colors** (returns hex color strings): `operatorBadgeColor`,
  `awayBadgeColor`, `secureBadgeColor`, `botBadgeColor`, `blockedBadgeColor`

- **Theme information**: `theme` or `themename` (current theme name),
  `scriptEditorHighlightrTheme` (Highlightr theme name for script editor)

Returns a string value, or an empty string if the option is not found. Colors are
returned in hex format (e.g., `"#FF0000"`). A colour the theme leaves unset answers with the
colour drawn in its place: `operatorBadgeColor`, `awayBadgeColor` and `botBadgeColor` the
palette's `secondary` (else its `foreground`), and `highlightRowBackground` and
`expandedGroupRowBackground` the defaults [Themes.md](Themes.md) gives, for the current
appearance.

**It answers on every frontend.** `theme_get` is a question about the theme, not about the
window system, so it is not one of the commands that do nothing where there is nothing to
show — see *Frontends without a UI* below. The values are the ones the frontend you are
running on is actually drawing with:

- **`Rete.app`** uses the appropriate light/dark palette for the current appearance mode,
  and the answers change when you change theme or when the system flips light/dark.
- **`reTerm`** answers from the theme's **dark** palette, always. A terminal has no
  appearance to ask about, so there is nothing to follow; `reTerm` renders with the dark
  palette for the same reason. `/set theme` changes what `theme_get` answers, immediately.
  The badge and buffer-list-icon options still return SF Symbol names verbatim even though
  nothing draws them there — the name is part of the theme, and a script that passes it on to
  something else should get it. `scriptEditorHighlightrTheme` is answered the same way, though
  `reTerm` has no script editor: the dark palette's value, or `github-dark` when the theme names
  none.

**Buffer-list icons answer with the effective symbol.** A theme may leave one of the seven
`bufferList…Icon` fields out, in which case the sidebar draws Rete's built-in symbol for that
buffer kind and `theme_get` answers with **that** symbol's name — `number` for a channel,
`person` for a direct message, `gearshape` for the system buffer, `waveform.path` for raw,
`globe` for web, `terminal.fill` for a script buffer and `terminal` for script debug. A theme
may also set one of them to the **empty string**, which means *"draw no icon for this kind"*;
`theme_get` then answers with the empty string, because that is what is on screen. So for these
seven options — and only these — an empty answer is a real answer and not "no such option". If
you need to tell the two apart, ask for an option you know exists as well.

`theme` and `themename` report the theme that is **in use**, which is not always the one
that was selected: if the chosen theme's file is missing or unreadable the frontend falls
back to the built-in and says so here, rather than naming colours that are not on screen.

If no frontend has published a theme yet — which should not happen in either shipped
product — every option answers the empty string. `theme_get` never raises an error for an
option it does not know.

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

The option list above is the whole of what `theme_get` can answer, and **this page is the whole
story about reading a theme from a script** — [Themes.md](Themes.md) documents the theme
*format*, every palette key, colour role and field a theme file may set, and says nothing about
Tcl. Read it to learn what a theme may contain; read this list to learn what a script can ask
for. Every SF-Symbol-valued field the theme format has is readable from a script.

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
::rete::ischanquiet quiet #channel
```

These commands check if a specific mask is present on the channel's ban list,
exception list, invite-exempt list, or quiet list. Each returns `1` if found,
`0` otherwise.

```tcl
::rete::chanbans #channel
::rete::chanexempts #channel
::rete::chaninvites #channel
::rete::chanquiets #channel
```

These commands return Tcl lists of entries for the respective channel lists.
Each entry is a sublist of the form `{mask bywho age}`:

- `mask`: the ban/exempt/invite/quiet mask
- `bywho`: the nickname or server that set the entry (may be empty)
- `age`: seconds since the entry was set (may be empty if timestamp is unknown)

`chanquiets` and `ischanquiet` work on the channel's `+q` list, which only the
Solanum family and its relatives implement (RPL_QUIETLIST, 728/729). On a server
without quiets the list is simply empty, so `chanquiets` returns nothing and
`ischanquiet` returns `0`.

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

> **These two take effect immediately**, unlike the buffer commands below — the change is
> in place by the time the command returns, so the next line of the same script reads it
> back. That includes calling them from inside a `bind` handler: the value you set stands,
> and the event that ran your handler does not overwrite it.

### Helper commands (buffer manipulation)

> **When these take effect.** Seven commands — `createbuffer`, `delbuffer`, `hidebuffer`,
> `setweburl`, `addnickmenu`, `addmenubutton` and `addtoolbarbutton` — **apply their change
> after the command returns, not during it.** Bad arguments are still reported to you as a
> Tcl error straight away; a failure only discovered while the change is being applied — an
> unknown buffer, say — goes to the Script Debug buffer instead, because by then the command
> has already returned.
>
> The practical consequence is one line long: **you cannot read back your own change on the
> next line of the same script.**
>
> ```tcl
> ::rete::createbuffer "MyBuffer" script
> set id [::rete::getbuffer "MyBuffer" script]   ;# still "" — too early
> ```
>
> Use the name you passed in — the buffer commands all accept a name where they accept a
> UUID — or resolve the UUID from a later event or timer:
>
> ```tcl
> ::rete::createbuffer "MyBuffer" script
> ::rete::appendmsg "MyBuffer" "" "Ready" "info" "script"   ;# fine: applied in order
> ```
>
> Ordering is preserved, so calls apply in the order you made them — and `appendmsg` is
> deferred the same way, which is why appending to a buffer you just created works. The
> getters (`getbuffer`, `getserverbuffer`, `channels`, …) are **not** deferred: they answer
> from the state as it is at the moment you ask, which is why the first example returns `""`.

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

On a frontend with no SF Symbols (`reTerm`), the name cannot be checked and is stored as
given; nothing displays it there. An unknown buffer is still an error on every frontend.

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

The namespace is the **bundle's `id`** — the manifest's `"id"`, or the bundle
directory's name when the manifest omits it. You do not declare it anywhere else, and
nothing in a `.tcl` file can change it.

A `# Namespace:` comment in the entrypoint is inert. If your script carries one and saved
data under that name, set the manifest's `"id"` to the same name to keep reading it.

A bundle whose `id` cannot be determined has no namespace, and storage commands in it
return the error *"store: script has no namespace"*.

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

- All storage operations are scoped to the script's namespace, which is the bundle's `id`
- Data persists across sessions in a JSON file at `~/Library/Application Support/Rete/script_storage.json`
- Scripts can only access data in their own namespace - namespace isolation is enforced
- List operations return proper Tcl list objects that can be used with standard Tcl list commands
- Dict operations return proper Tcl dict objects that can be used with standard Tcl dict commands
- If you need to store a string with spaces, use `setstring`/`getstring`, not list operations

**When the store cannot reach the disk.** A `set…`, `lappend`, `lremove`, `dictset` or
`dictunset` writes the whole file every time, and **it still returns success when the write
fails** — the value is in memory for this session and gone at the next launch. Scripts should
not try to detect it.

Rete reports the failure to the user through the same error notice it uses for its own
settings — an alert on macOS and iOS, a line in the app buffer in `reTerm` — naming
`script_storage.json` and the reason the filesystem gave. The same notice appears when the file
is there and cannot be read, or holds something Rete cannot parse: in both cases scripts start
with nothing stored **and the next `set…` replaces the file**. Repeats are collapsed, so a
script writing in a loop produces one notice rather than one per call. A script sees none of
this — no return value, no error.

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

This is the entrypoint of a bundle whose `manifest.json` reads
`{"id": "example", "entrypoint": "main.tcl", …}` — that `id` is the namespace its stored
values are filed under. There is no header to write.

```tcl
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
  on every UI re-render. They are called each time it opens.
- Their items appear just before the `addnickmenu` items.
- A provider should answer quickly: if the providers have not answered within
  a quarter of a second, the menu shows without their items.
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
    # serverName - host:port of the connection's server
}
```

Example:

```tcl
proc my_storeheader_action {storeId serverName} {
    ::rete::echo "Clicked store header for $serverName ($storeId)"
}

::rete::addstoreheaderbutton "My Store Action" my_storeheader_action
```

**Stability of `storeId` on this menu.** The value is the same UUID for as long as the
store exists, so it is safe to use as a dictionary key. A store reports its saved
configuration's UUID; before that configuration is attached to it — a short window while
the connection's store is being created — it reports an id minted with the store instead.
It is **not** stable across restarts and it is **not** written to disk, so do not persist it.

**What `serverName` carries on the store menus.** For a connection with a saved IRC
configuration it is that configuration's **first server**, written as `host:port` — for
example `irc.libera.chat:6697` — with an IPv6 address written without brackets. It is the
configured server, not whichever fallback server the connection is using at the moment, and
it changes when the connection is edited. It is an empty string if it is not known yet. The same value is passed to
`addmenubutton` callbacks and to a `setstoreheadermenuprovider` provider.

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
    # serverName - host:port of the connection's server
}
```

Example:

```tcl
proc my_storemenu_action {storeId serverName} {
    ::rete::echo "Clicked top-level store menu for $serverName ($storeId)"
}

::rete::addmenubutton "My Store Menu Action" my_storemenu_action
```

**Stability of `storeId` on the top‑level store menu.** It is the same value the store
header menu passes for the same connection, and it is stable in the same way: the same UUID
for as long as the store exists. It is not stable across restarts and must not be persisted.
`serverName` is the value described under the store header menu above.

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
the following helpers. All of them operate on the current connection.

- **Raw lines**

```tcl
::rete::putserv RAWLINE...
```

Example:

```tcl
::rete::putserv PRIVMSG #channel :Hello from Tcl
```

**The line is sent as written; only CR, LF and NUL are touched**, as RFC 2812 forbids only those
in a line — for `::rete::putserv` and `/quote` alike. A CR or LF at either end is dropped (it is
the line's own ending), and the line is cut at its first CR, LF or NUL anywhere else, so one call
sends at most one line. Everything else goes out byte for byte, ends included: a VT, FF, U+0085,
U+2028 or U+2029 at the start or end of the line is sent. A typed `/quote` sends the line exactly as
typed: only the spaces and tabs between `/quote` and the line are dropped, because they separate the
command from its argument, so `/quote <VT>PING x` sends `<VT>PING x`.

**A raw line is at most 510 bytes, not counting its message tags when the server takes them**
— on a direct connection, for `::rete::putserv` and `/quote` alike. On a connection that has
negotiated the `message-tags` capability, a line that starts with IRCv3 tags (`@key=value;… `) has
a tag section up to its first space, and the 510 applies to the message after it: a longer
message is cut at 510 bytes and a notice says so (`Your <VERB> to <target> was <N> bytes and IRC
allows 510, so the last <M> bytes were not sent, starting at: …`), but **the tags are never cut**.
**Without `message-tags` the whole line counts**, tags included, and is cut at 510 bytes — such
a server would read the tags as part of the message anyway. **With `message-tags`, tag data over 4094 bytes is not sent at all**: the line is dropped
and a notice says
`Your <VERB> to <target> had message tags longer than 4094 bytes, so the line was not sent.`
(without ` to <target>` when the command names none). Without `message-tags` there is no 4094
refusal — the line is cut at 510 like any other. Both notices go to the buffer the line was for,
or the server buffer.

- **Basic commands**

```tcl
::rete::join #channel ?key?
::rete::part #channel ?reason...?
::rete::msg ?-sensitive? ?--? target text...
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

The last argument of `::rete::msg`, `::rete::notice` and `::rete::action` (the text),
`::rete::part` and `::rete::kick` (the reason), `::rete::topic_set` (the new topic),
`::rete::quit` (the message), `::rete::silence` (the input) and `::rete::ctcp` (the
parameters) takes every remaining argument and joins them with single spaces, so
`::rete::msg #channel Hello world` and `::rete::msg #channel "Hello world"` send the same text.

**A reason is cut so the server can relay it whole.** `::rete::part`, `::rete::kick` and
`::rete::topic_set` — and `/part`, `/kick` and `/topic` — leave room
in the line for the `:nick!user@host ` the server puts in front of it when it relays it, exactly as
a `::rete::msg` chunk does (see `MSG_OUT`). These are one-line commands, so a longer reason is **cut, not split**, on a
character boundary, and a notice says so: `Your <VERB> to <target> was <N> bytes and IRC allows <M>
once the server adds your address, so the last <K> bytes were not sent, starting at: …`. A quit message (`::rete::quit`, `/quit`) is sent as typed. A topic (`::rete::topic_set`, `/topic`) is also cut, on a character boundary and without a notice, to the server's advertised `TOPICLEN`; a topic the cut would leave empty is not sent. The server's own `KICKLEN` still applies on top. `::rete::putserv` and `/quote` are
not cut this way.

**A line still waiting when its connection ends is not sent.** Commands go out one
at a time, paced by the flood throttler, so a burst can still be waiting when the connection
ends. If the user disconnects — the Disconnect button, or a typed `/quit`, which does not wait
behind lines already queued — every line still waiting is dropped: one the throttler was already
holding is reported in its buffer as `Not sent (disconnected): …`, and so is every later paced
line for that connection until the next connect — one notice per chunk, so a long message that was
mid-way through its chunks reports each chunk it did not send. A script command that had not
started yet is dropped **without** a notice. The
notice quotes a PRIVMSG or NOTICE whole, and any other command — or a service login to any target
of a comma list — by its verb and first target only, so it never repeats a password. After an
unexpected drop or a reconnect — the user's, or the automatic one after the link is lost — nothing
queued for the old connection is sent, and each line **the user** typed that was dropped gets the
same `Not sent (disconnected): …` notice. **A line a script sent that is dropped this way gets no
notice, and the command that sent it reports nothing**: it returned when the line was queued, so a
script cannot learn from it that the line never went out. **A line is
never written to a connection other than the one it was queued for.** `::rete::quit` is a script
command like the others and goes out after the script's earlier ones.

**One exception, and it is deliberate: `::rete::msg` is not the same as typing
`/msg`.** A typed `/msg` dispatches `MSG_OUT` first and can be blocked by a handler;
`::rete::msg` fires no `MSG_OUT` at all, so a script can send from inside its own
`MSG_OUT` bind without re-entering it. See `MSG_OUT` under **Event arguments** for
what that means for ordering. `::rete::action` and `::rete::notice` need no such
carve-out, because `/me` and `/notice` do not dispatch `MSG_OUT` either.

**`::rete::msg -sensitive target text...` is `/smsg` for a script.** The server
receives the text in full, while everything Rete itself shows or stores for it —
the echo row in your buffer, the raw buffer, the logs, and what AI access can read —
carries `********` in its place. **`bind RAW_OUT` is the exception**: like every sensitive
line, it receives the real text (see `RAW_OUT` under **Event arguments**). Like `/smsg` it fires no
`MSG_OUT` (as plain `::rete::msg` does not), and the ordering note above applies
unchanged.

On a connection that cannot mask the text (a ReteCloud session), it
**sends nothing** and raises a Tcl error:

```
cannot send sensitive message: this connection cannot mask it
```

Unlike the typed `/smsg`, which reports its refusal into the buffer, this is an
error your script sees, so it can learn that its secret was not sent — wrap the
call in `catch` if you want to handle it.

Options come before the target. Any other word beginning with `-` in that position
is an error — `unknown option "-secret": must be -sensitive` — so to send to a
target that begins with `-`, end the options with `--`:
`::rete::msg -- -target text` (or `::rete::msg -sensitive -- -target text`). So
`::rete::msg -foo hello` raises `unknown option "-foo": must be -sensitive` rather than sending
to `-foo`; if your script computes a target that might begin with `-`, end the options with `--`.

### Custom commands

Scripts can register custom commands that will be called when the user types
`/command` in the input field. Custom commands can override native commands.

**A `/command` that neither Rete nor any script defines is sent to the server as typed**:
`/knock #chan please` goes out as `KNOCK #chan please` — the command name
upper-cased, the rest unchanged — and fires `RAW_OUT` like any other line. A script
command is looked up first, so registering a name takes it away from the server, and
removing it gives it back. A line that carries a credential by the client's masking
rules (`NS`, `NICKSERV`, `CS` or `CHANSERV` with a login verb such as `IDENTIFY`, `PASS`,
`AUTHENTICATE`) reaches the server in full and is shown masked in the raw buffer;
`RAW_OUT` sees the real line, as it does for every sensitive line. Two kinds of name
still answer *"Unknown command: /x"* and send nothing: a word that is not letters and
digits (`/@x`, `/#chan`), and a command one of Rete's apps owns — `exit`, `help`, `set`,
`server`, `network`, `control`, `buffer`, `raw`, `script`, `rowaudit` — typed where that
app's input does not claim it (for example `/exit` on macOS).

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

**Ordering.** Creating and killing a timer take effect immediately, in script
order. A timer is listed by `timers`/`utimers` and can be killed on the very next
line after the `timer`/`utimer` that created it, and once `killtimer`/`killutimer`
returns the timer is gone and will not fire — including when it is killed before
its first scheduled fire. Scripts do not need to yield or wait between the two.

**Re-using a name replaces the timer.** `timer`/`utimer` given a `timerName` that
is already armed stops the timer holding that name and installs the new one in its
place, so a name always refers to exactly one live timer. Re-arming a named timer
— at the top of an entrypoint, say — therefore leaves nothing running behind it,
and a later `killtimer`/`killutimer` on that name stops the one you created last.

Example:

```tcl
set t [::rete::timer 5 my_task 0]
# ... later ...
::rete::killtimer $t
```

**Lifetime.** A timer belongs to the bundle that armed it, and **stops when that
bundle does**. Unloading a bundle disarms every timer it created, exactly as it
drops the bundle's `bind` handlers and custom commands — you do not need a
`killtimer` in a teardown proc. The same applies to the paths a bundle leaves by
without being unloaded:

| What happens to the bundle | Its timers |
|---|---|
| Unloaded | disarmed |
| Reloaded | disarmed, then re-armed by the new run of the entrypoint |
| Load fails part-way through the entrypoint | disarmed, along with its binds and commands |
| The interpreter is destroyed — the network is removed or closed, or `/script rehash` | disarmed |

So a timer armed at the top of an entrypoint does not accumulate a second copy
each time you reload the bundle while developing it, and a timer armed on the
line before one that raises does not survive the failed load to call a procedure
that was never defined.

A timer armed **outside** any bundle — typed into an interpreter directly rather
than run from a bundle's files — has no owner, so no unload sweeps it. Stop it
with `killtimer`/`killutimer`. The last row of the table is the exception: the
interpreter's own teardown disarms **every** timer, owned or not, because nothing
survives the interpreter it would fire into.

**Read that last row with "How the entrypoint runs" beside it.** A reconnect does not
destroy the interpreter — on direct IRC or on ReteCloud — so your timers keep their
schedule across it, and keep firing while the network is disconnected.

A timer that ticks in the instant one of these stops lands is dropped silently —
it is noted in the Script Debug buffer at `debug` level, not as an error. Timers
are stopped from a different thread than the one they fire on, so a tick that was
already on its way when you called `killutimer` (or when the connection went
away) always has that one arrival left. **That dropped tick does not run your
command** — it is refused before the command is reached. A tick that had already
*started* running the command finishes it, and then stops rather than
rescheduling.

**Notes:**

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

### Frontend identity and capabilities

Rete ships more than one frontend — `Rete.app` on macOS, Rete on Linux, and `reTerm` in a
terminal — and they run the same scripts. Two commands let a script find out where it is.

- **Which frontend is running:**

```tcl
::rete::frontend
```

Returns a short, stable identifier:

| Value | Frontend |
|---|---|
| `rete` | Rete: `Rete.app` on macOS, and Rete on Linux |
| `reterm` | `reTerm`, the terminal client, on macOS and Linux |
| `unknown` | the host did not identify itself |

`unknown` is a normal answer, not an error — treat it as "some frontend, don't assume".
The command never fails and never returns an empty string.

Rete on Linux is the same product as `Rete.app` and answers `rete` too: a script behaves
the same in both. **To tell the operating system apart, use Tcl's own `$tcl_platform(os)`**
— `Darwin` on macOS and `Linux` on Linux, on every frontend (Rete has no command of its own
for it):

```tcl
::rete::debug "[::rete::frontend] on $::tcl_platform(os)"   ;# e.g. "rete on Linux"
```

- **Whether a feature is available:**

```tcl
::rete::has_capability <name>
```

Returns `1` or `0`. Defined names:

| Name | True when |
|---|---|
| `ui` | the frontend can actually present `::ui::*` views and controls — `1` in `Rete.app` and in Rete on Linux, `0` in `reTerm` |

An unrecognised name returns `0` rather than raising an error, so a script written for a
later version of Rete can ask about a capability this version has never heard of and get a
truthful answer.

> **Branch on `has_capability`, not on `frontend`.** What a script depends on is whether
> something can be *done*, and that is what the capability query answers. `::rete::frontend`
> is for log lines, bug reports, and telling a user why a window did not appear — a script
> that keys behaviour off the product name breaks the day a frontend gains or loses a
> feature.

```tcl
if {[::rete::has_capability ui]} {
    ::ui::view load my_view -callback on_view_loaded
} else {
    ::rete::debug "no UI here ([::rete::frontend]) — printing instead"
    ::rete::appendmsg [::rete::getserverbuffer] "" "Report ready" "info" "script"
}
```

You should rarely need this. Everything in the next section is safe to call unconditionally
on every frontend; see **Frontends without a UI** at the end of it.

### Script UI API (`ui::` namespace)

Scripts can build script-driven windows and forms using the `::ui::` namespace. `Rete.app` and
Rete on Linux both present them, from the same view file and the same commands; `reTerm` does
not (see **Frontends without a UI** below). Rete on Linux draws each view as its own window,
widget for widget as `Rete.app` does; what differs there is listed under **Rete on Linux**
at the end of this section.

**View:**

- **Load a view by name** (async; the view handle is passed to the callback):

```tcl
::ui::view load <viewName> ?-callback <proc>?
```

`<viewName>` is a **registered view name** — not a filename, and not a path. Names are
registered from your bundle's JSON files when the bundle loads; see **Registering a view
name** below. The argument is never resolved against the filesystem, so a name that is not
registered for your script raises an error (*"view '…' not found for current script"*)
rather than reading a file.

`load` returns the empty string immediately; the view handle arrives later, as the single
argument to `-callback`.

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

The view file is a single JSON object with these top-level keys:

- **`name`** (**required** string) — the logical name `::ui::view load` takes. **A file with no `name` is never registered, so no `load` call can reach it** — not under its filename, not under its path, not at all. This is the one key the file cannot be loaded without.
- **`title`** (optional string) — window title.
- **`widgets`** (optional array) — list of root widgets. Widgets are laid out in order, top to bottom. Containers group children vertically (`container` / `vbox`) or horizontally (`hbox`).

One file can instead carry **several views**, as a top-level **`views`** array of objects,
each with its own `name`, `title` and `widgets`. Every element that has a `name` is
registered under it; an element without one is skipped, for the same reason a whole file
without one is.

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
  "name": "my_view",
  "title": "My Window",
  "widgets": [
    { "type": "label", "id": "lblTitle", "props": { "text": "Hello" } },
    { "type": "button", "id": "btnGo", "props": { "title": "Go" } }
  ]
}
```

That is the file `::ui::view load my_view` opens — the `name`, not the filename.

**Example with containers and hbox**

```json
{
  "name": "form",
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

**Registering a view name**

View names come from your script bundle's `manifest.json`. When the bundle loads, Rete reads
every file the manifest lists with `"type": "json"` and registers the `name` it finds inside
(or each `name` under `views`). **A view file the manifest does not list with type `json` is
never read, so its name is never registered** — the same dead end as a file with no `name`,
reached from the other side.

```json
{
  "id": "myscript",
  "entrypoint": "main.tcl",
  "files": [
    { "filename": "main.tcl", "type": "tcl" },
    { "filename": "my_view.json", "type": "json" }
  ]
}
```

Names are scoped to the bundle that registered them, so two bundles can each ship a view
called `settings` without colliding and neither can load the other's. **They are scoped to
the connection as well**: registering a name is part of loading the bundle into a
connection's interpreter, and every connection has its own (see **How the entrypoint runs**
under **Script bundles**). Each connection therefore registers the names for itself, and
unloading or reloading the bundle on one connection retires them on that connection alone —
a `::ui::view load` running on any other connection still resolves.

Registration happens **before** your entrypoint runs, so `::ui::view load` works from the
top level of the script as well as from a handler.

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

  The value is the one the field last published to the script side, not a synchronous read of
  the field itself. It is up to date by the time any bound handler runs — `onClick`, `onSubmit`
  and the rest — so the Save-button case above is exact. The only way to observe otherwise is a
  `get` from a timer that lands in the moment between a keystroke and the field publishing it,
  which returns the value from immediately before that keystroke. A value your own script has
  just written with `ui::input set` always reads back as written.

- **Table** (rowId-keyed):

```tcl
::ui::table setcolumns <elemHandle> <column1> <column2> ...
::ui::table add <elemHandle> -id <rowId> -data <dict>
::ui::table update <elemHandle> -id <rowId> -data <dict>
::ui::table remove <elemHandle> <rowId>
::ui::table clear <elemHandle>
```

`-data` is a Tcl dict of column name to cell text, however you build it — all of these are
the same row:

```tcl
::ui::table add $tbl -id r1 -data [dict create Player $nick Score "10 pts"]
::ui::table add $tbl -id r1 -data [list Player $nick Score "10 pts"]
::ui::table add $tbl -id r1 -data {Player alice Score {10 pts}}
```

A repeated key keeps its last value, as `dict get` does. Text wrapped in one extra pair of
braces — `-data "{Player alice Score {10 pts}}"` — is read as the dict inside it. A `-data` that is not a dict (an odd number of words,
or unbalanced braces) adds the row with no cells; it is not an error.

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

Shows a modal dialog with a badge on the left and the message text on the right. While it is open, Rete keeps receiving and showing messages in every buffer, and the script keeps running. **-style** sets the badge: `error` (red, default) or `warning` (orange). Default **-title** is `Error`; use `-title` to override (e.g. `-title "Validation"`).

**Events:** `button` → `onClick`; view window → `onClose`; `table` → `onSelect`, `onDoubleClick`; `input` → `onSubmit` (Enter key; `::ui(event) data value` is the field text).

When an event fires, the script runs on the serial queue with `::ui(event)` set to a dict: `type`, `target`, `view`, `data`.
`data` is itself a dict, and every key and value in it — and `type`, `target` and `view` — is
exactly one Tcl word holding the text as it was, with nothing substituted: an `onSubmit` of
`a b c` is `value {a b c}`, so `[dict get $::ui(event) data value]` is the whole field. (Before
this version a value with a space was split into extra keys, and `value` held only its first
word.)

```tcl
::ui::bind $inp onSubmit {
    set text [dict get $::ui(event) data value]   ;# the whole line, spaces and all
}
```

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
::ui::view load my_view -callback on_view_loaded
```

`my_view` here is the `name` inside `my_view.json`, and `my_view.json` is listed in the
bundle's `manifest.json` with `"type": "json"`. Do not pass a filename or a path — a path
your script builds itself (`[file join [file dirname [info script]] my_view.json]`, say) is
just an unregistered name and the call fails.

**Behaviour:** UI events run your Tcl on the connection serial queue; UI updates are applied on the main thread. The app does not block the serial queue waiting for the main thread, so your handlers can safely call UI commands.

#### Rete on Linux

Rete on Linux presents every `::ui::*` view, widget, event and alert above, with the same
layout file, the same handles, the same event names and the same `::ui(event)` data as
`Rete.app`. Only the drawing is the desktop's own:

- **Colours.** A container's `background` and `borderColor` names are drawn in the colours
  `Rete.app` uses for them, chosen for the light or dark appearance in effect when the window
  opens; a window does not recolour if the appearance changes while it is open.
- **A vertical container's background and border** are drawn once, around all of its children.
  (`Rete.app` draws them around each child, which looks the same for a container with one
  child — the usual case.)
- **A table row** fires `onSelect` on a click, and a double click fires `onSelect` for its
  first click and then `onDoubleClick`.
- **Alerts** use the desktop's error and warning icons, in the same red and orange.

#### Frontends without a UI

**Every `::ui::*` command exists on every frontend, and a script that uses them loads and
runs unmodified everywhere.** On a frontend that cannot present them — `reTerm` — the
windows and controls simply do not appear. You do not need to guard these calls, and you
do not need to test which frontend you are on.

What that means in detail:

- **Argument checking is identical everywhere.** A wrong number of arguments, an unknown
  `::ui::view` subcommand, or a view name that is not registered for your script raise the
  same error on every frontend. `catch {::ui::label set}` gives the same answer in
  `Rete.app` and in `reTerm`, so a bug in your script surfaces in both rather than being
  swallowed in one.
- **`::ui::view load` still calls your `-callback`, with an empty view handle.** Your setup
  proc runs, so everything in it happens: `::rete::` calls, variables you set, handlers you
  register with `::rete::bind`. Only the `::ui::*` calls inside it do nothing. This is the
  same thing `Rete.app` does when a view file cannot be read, so it is not a special case.
- **Handles stay well-formed.** `::ui::view elem $vh lblTitle` returns a handle string as
  usual (`::lblTitle` when `$vh` is empty), and passing it to `::ui::label`, `::ui::text`,
  `::ui::table` or `::ui::input` does nothing rather than failing.
- **`::ui::bind` returns a normal token** that `::ui::unbind` accepts. The binding never
  fires, because there is no window to fire it. **If your only cleanup is in an `onClose`
  handler, that cleanup never runs** — put anything that must happen on unload in the
  script's own teardown instead.
- **`::ui::input get` always returns a string**, and it is `""` — including for a value your
  script just wrote with `::ui::input set`, since there is no field to hold it.
- **`::ui::alert` shows nothing.** It still requires `-message`.

`::rete::theme_get` is **not** part of this: it is a question about the theme, not about the
window system, and it answers on every frontend — with that frontend's real palette, not with
empty strings. On `reTerm` that palette is the theme's dark one, and the badge options return
their SF Symbol names verbatim. See *Theme settings* above.

`::rete::setbuffericon` stores the icon name either way. On `Rete.app` an unrecognised SF
Symbol name is rejected — the buffer falls back to its default icon and a warning goes to
the Script Debug buffer. A frontend with no SF Symbols cannot check the name, so it stores
what you gave it verbatim; nothing renders it there.
