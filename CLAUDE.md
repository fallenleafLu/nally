# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Nally is an open-source macOS BBS client (telnet + ssh), descended from MacBlueTelnet/Nally by yllan.org.
It is a Cocoa app written in **Objective-C and Objective-C++ (MRR, not ARC)**, deployment target macOS 12.0.
Project-facing docs (README, `Deploy/Changelog*.markdown`) are written in Traditional Chinese; issues are tracked
on GitHub and on a YouTrack instance linked from the README.

## Build / Run

CocoaPods is used (`ImgurAnonymousAPIClient`), and `Pods/` + `Podfile.lock` are gitignored, so a fresh clone needs:

```sh
pod install                                           # required before the first build
xcodebuild -workspace Nally.xcworkspace -list         # available schemes
xcodebuild -workspace Nally.xcworkspace -scheme Nally -configuration Debug build
open build/Debug/Nally.app                            # or Products dir reported by xcodebuild
```

Always build the **workspace**, never `Nally.xcodeproj` directly — the project reads `Pods-Nally.*.xcconfig`
and has a "Check Pods Manifest.lock" build phase that fails without them.

Shared schemes: `Nally` (Debug) and `Nally-Release`.

The `Makefile` predates CocoaPods (it calls bare `xcodebuild`, and `make release` runs `Scripts/package.py`, a
Python 2 Sparkle-appcast packager that imports `commands`/`markdown2`). Treat it as legacy; don't rely on it.

### Tests

`Tests/TextSuiteTests.m` covers `YLTextSuite` line wrapping (CJK width, 避頭尾 punctuation rules). The
`TextSuiteTests` target is an old-style `.octest` bundle built against **SenTestingKit**, which modern Xcode no
longer ships, and the `Nally` scheme's `<Testables>` list is empty — so `xcodebuild test` does not currently run
anything. Migrating this to XCTest is a prerequisite for any test-driven work here.

## Architecture

### Connection → terminal → view pipeline

Bytes flow one way through three objects per tab:

1. **`YLConnection`** (`Code/YLConnection.mm`) — abstract transport, `YLConnectionProtocol`. Two subclasses:
   - `YLTelnet` (`.mm`) — telnet option negotiation ported from PuTTY's `telnet.c`, runs over `NSStream`.
   - `YLSSH` (`.mm`) — `forkpty()` + `execlp("/usr/bin/ssh", …)`. An `ssh://bbs@host` address logs in as the
     `bbs` user; plain `ssh://host` uses the site account. Password is typed into the pty, not passed as an arg.
   - `+connectionWithAddress:` picks the subclass from the URL scheme.
2. **`YLTerminal`** (`Code/YLTerminal.mm`, ~1200 lines) — the VT100/VT102 emulator. `feedBytes:length:` is a
   byte-at-a-time state machine (`TP_NORMAL`/`TP_ESCAPE`/`TP_CONTROL`/`TP_SCS`) using `std::deque` buffers for
   control-sequence args. State lives in `cell **_grid` plus a parallel `char *_dirty` byte-per-cell map.
   `CommonType.h` defines `cell` = `unsigned char byte` + a 16-bit `attribute` bitfield (fg/bg/bold/underline/
   blink/reverse/doubleByte/url). Also owns derived per-row state: `updateURLStateForRow:` marks URL runs,
   `updateDoubleByteStateForRow:` marks CJK lead/trail bytes.
3. **`YLView`** (`Code/YLView.mm`, ~1650 lines) — **an `NSTabView` subclass that is also the terminal canvas and
   the `NSTextInput` client.** It reads the front terminal's grid directly (`YLDataSourceProtocol`, an informal
   `NSObject` category) and renders into a cached `_backedImage`; `updateBackedImage` only repaints cells the
   terminal marked dirty, then clears the dirty map. There is no redraw timer: `YLTerminal` schedules the
   view's `-tick` with `performSelector:withObject:afterDelay: 0.07` at the end of every chunk it parses, and
   `YLController` runs a 1 Hz `updateBlinkTicker:` that redisplays the view while any cell is blinking.

Each tab is an `NSTabViewItem` **whose `identifier` is the `YLConnection` itself** — `[[self selectedTabViewItem]
identifier]` is how `frontMostConnection`/`frontMostTerminal` work. There is no per-tab controller object.

### Controller and globals

`YLApplication` (`NSPrincipalClass`) loads `MainMenu.xib`; `YLController` (`Code/YLController.mm`) is the app
delegate and owns the sites list, emoticons, the `PSMTabBarControl` tab bar wired to the `YLView`, and the
plugin loader. `newConnectionWithSite:` is the single place a tab + terminal + connection get wired together.

`YLLGlobalConfig` is a `@public`-ivar singleton holding fonts, the 2×10 ANSI color table, grid size, and user
prefs. `YLView.mm` caches it into file-static `gConfig`/`gRow`/`gColumn` in `-configure`, and `YLTerminal`
allocates `_grid` from `[gConfig row]`/`[gConfig column]` at init — **grid geometry is fixed per terminal
instance**, so size/font changes need a reconnect or re-`configure`, not just a redraw.

`Code/encoding.m` is ~8000 lines of static Big5/GBK ↔ Unicode tables; `init_table()` must run before any
conversion and is called once from `main.m` before `NSApplicationMain`.

### Paste / contextual menu

`PasteController` (singleton) implements "Smart Paste": on ⌘V it inspects the pasteboard and may substitute a
TinyURL (URLs longer than 40 chars) or upload image data/file-URLs to Imgur, inserting the returned URL via
`[[(YLController *)[NSApp delegate] telnetView] insertText:]`. `YLContextualMenuManager` builds the right-click
menu (open URL(s), Google image search, dictionary lookup, Paste TinyURL, Paste image to Imgur) from the current
selection. `anotherPasteStillOnGoing` guards against overlapping network pastes.

### Plugins

`YLPluginLoader` scans the app's `PlugIns` folder and `~/Library/Application Support/Nally/PlugIns` on a
background thread and loads any bundle whose identifier starts with `org.yllan.Nally.Plugin`; plugin principal
classes subclass `YLBundle` to contribute a menu. `YLTerminal -feedData:connection:` forwards every inbound
chunk to the loader. `Plugins/HelloNally` and `Plugins/ImagePreviewer` are sample plugins with their own
`.xcodeproj`s — they are **not** built by the main project.

## Conventions and gotchas

- **Manual retain/release.** New code must `retain`/`release`/`autorelease` and use `NSAutoreleasePool` to match;
  ARC is not enabled on any target.
- `.mm` files are Objective-C++ (`YLTerminal`, `YLView`, `YLSSH`, `YLTelnet`, `YLConnection`, `YLController`,
  `YLApplication`). Adding C++ to a `.m` file requires renaming it in the pbxproj.
- **Localized nibs are generated, not checked in.** Only `Resources/English.lproj/MainMenu.xib` and
  `Preferences.xib` exist. A shell-script build phase runs `ibtool` to export English strings and compile
  `zh_TW`/`zh_CN` nibs from `Resources/<lang>.lproj/*.strings`. Edit the English xib and the `.strings` files;
  never expect a localized xib on disk.
- The app is sandboxed (`Nally.entitlements`: app-sandbox + network client) for Mac App Store submission — keep
  that in mind for anything touching `forkpty`/`/usr/bin/ssh` or the filesystem.
- Site passwords go through `SSKeychain` (`Dependencies/SSKeychain`), keyed by service = site address.
- `Dependencies/` holds vendored sources (PSMTabBarControl tab bar, SSKeychain, DBPrefsWindowController) that are
  compiled straight into the app target — changes there are local forks, not upstream.
- Default branch is `develop`.
