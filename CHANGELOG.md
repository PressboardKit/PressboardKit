# Changelog

Formát dle [Keep a Changelog](https://keepachangelog.com/), verzování [SemVer](https://semver.org/).
Nahoře drž rozpracovanou verzi; při vydání doplň datum a bumpni.

## [Unreleased]

### Added
- **`PressboardKitLicensing`: licensing is part of the SDK** (ENG-197) — `PressboardLicence.activate(key:appGroup:)`
  in your app and `.withLicence(appGroup:application:)` on the configuration in your extension.
  Fetching, storage and the daily cadence are the SDK's, not an integrator's to reimplement. The
  module is linked by the **app** and never by the keyboard, so the extension still contains no
  code that can open a connection (LICENSE §5).

### Changed
- **A lease that has run out is now refused** (ENG-194) — `exp` was decoded and then ignored, so
  a lease verified on its `soft`/`hard` dates alone. Those sit years out for a healthy licence,
  which meant an application that stopped refreshing, or whose licence was removed the day after
  it last called home, kept everything indefinitely. The engine still never refuses to type: an
  expired lease is the free tier.
- **An unregistered application now gets the free tier** (ENG-192) — the engine carries a root
  public key, so a lease is verified for real: supply one and it decides what is unlocked, supply
  none and the keyboard types in English with every paid behaviour quiet. The gate an integrator
  could previously inject (`PressboardConfiguration.licence`) is **internal**; a licence anybody
  can overrule in one line is not a licence. See INTEGRATION.md §15b.
- **Fixed: the bundled dictionaries were never actually gated by a lease** (ENG-192) — the check
  read the injected gate rather than the resolved one, so a host running on a lease had the paid
  word lists open regardless of what the lease said.
- **Renamed the namespace types so they no longer shadow their modules** (ENG-191) —
  `PressboardKit` → `PressboardEngine` (carries `version`), `PressboardKitLayouts` →
  `PressboardLayouts` (carries `defaultLocales`), and the empty `PressboardKitAutocomplete`
  namespace is gone. A public type named after its module cannot be expressed in a
  `.swiftinterface`, which rules out library evolution and with it binary distribution.
  Breaking, and deliberately done while there is one integrator rather than several.

### Added
- **The engine can read a licence lease** (ENG-189) — `LeaseVerifier` checks a root-signed trust
  bundle against a key compiled into the binary, takes the lease-signing key it names, and
  verifies the lease with it. `PressboardConfiguration` gains `loadLicence`, `loadTrustBundle`,
  `applicationBundleID` and `onLicenceStatus`; the gate is derived from the result.

  The kit still opens no sockets. A keyboard extension has no network until the user grants Full
  Access, so the containing app fetches and this only ever reads — which is why the whole thing
  is closures over bytes rather than a client.

  **Every failure lands on the free tier.** No lease, a malformed one, a foreign signature, an
  unknown key, the wrong application, an expired trust bundle, a clock wound backwards: all of
  them produce a working keyboard and a reason, never a crash and never a blank keyboard.

  Enforcement is off until a root key is compiled in, and an absent key means `.unrestricted`
  rather than locked — a build that shipped without one behaves like today's builds.

### Added (earlier)
- **The free tier exists and can be run** (ENG-188) — `PressboardConfiguration` takes a `licence`,
  and `.free` gives you the tier `FREE-VS-PRO.md` describes: English, core typing, rendering,
  gestures, callouts, styling and accessibility, with the paid features quiet. The default stays
  `.unrestricted`, so nothing changes for anyone who does not ask for it.

  `PaidFeature` is the vocabulary — `dictionaries`, `autocorrect`, `prediction`, `slide-to-type`,
  `key-targeting`, `emoji-panel`, `emoji-suggestions`, `themes`, `key-preview`, `math`,
  `spell-checking`, `layouts`, plus `locale.<code>`. Those strings are a contract with the service
  that will issue licences, so they are fixed now rather than invented twice.

  **Every character preview is paid**, the measured native key-pop included: the free tier shows
  nothing on key press. The selected style is reset alongside the switch, so a bubble style left
  armed cannot keep the bubble field alive on a tier that has no preview.

  **Locking degrades, it never breaks.** A locked language falls back to English rather than
  emptying the list; a locked dictionary falls back to the small built-in English word list rather
  than to nothing; and a locked language, dictionary or layout each fall back rather than empty out.
  The keyboard types under every combination, which is `LICENSE` §8a
  and is now a test rather than an intention.

  `LicenseGate.unlicensed` is deprecated in favour of `.unrestricted`. The old name read backwards:
  it unlocked everything, while `FREE-VS-PRO.md` specified tests asserting that paid features do
  nothing "with `.unlicensed`".

  A host's own word list is not gated. The licence covers the 39 lists we ship, not data an
  integrator brought with them.

### Removed
- **Georgian no longer ships a word list** (ENG-185) — it types without predictions now, as Armenian
  and Belarusian already do. The list we had was corrupt at source, in the 2016 and 2018 editions
  alike: 20,956 of its 40,000 entries used letters dropped from the Georgian alphabet in the 19th
  century, and `და` — the commonest word in the language — sat at position 110 beneath the noise.
  Wrong suggestions and wrong autocorrections in a language are worse than none. A sound list puts
  it straight back; ENG-186 records what has been looked at and why it was not good enough.

  Counts move with it: 39 bundled lists, 53 of the 73 locales resolving to one.

  The test that should have caught this passed for months. It asked whether the list was long and
  whether it contained `და` — and it was, and it did. It now asks whether each language's commonest
  word appears in the first fifty entries, across ten languages, because a frequency list that
  buries its own commonest word is not a frequency list.

### Changed
- **The kit is now called PressboardKit** (ENG-184) — `Swift` is a registered Apple trademark, and
  Apple's rules for third parties forbid using one "as or as part of a … product name", including
  variations and takeoffs; their own worked example, `iPodMart`, is the same shape `SwiftboardKit` was.
  The kit is meant to be sold, and you cannot build a trademark of your own on someone else's.

  **What changed for an integrator.** Every module is renamed (`import PressboardKit`,
  `PressboardKitLayouts`, `PressboardKitAutocomplete`, `PressboardKitEmoji`, `PressboardKitApp`), as is
  every public type: `PressboardKeyboardView`, `PressboardController`, `PressboardConfiguration`,
  `PressboardInputViewController`, `PressboardRootView`, `PressboardTextDocument`, and
  `KeyboardBehavior.pressboardDefault`. The behaviour behind them is untouched — this is a rename, not
  a redesign, and no shims are provided because the kit is in early access with one integrator.
  The `os_log` subsystem is now `PressboardKit`, so saved Instruments filters and any
  `log show --predicate 'subsystem == "SwiftboardKit"'` recipe need updating.

  **What deliberately did not change.** `EmojiRecentsStore`'s default key is still
  `"swiftboardRecentEmoji"`. It is the key everyone's recent emoji are already stored under; renaming
  it would not migrate them, it would silently lose them. Older entries in this file also keep the old
  name, because that is what the code was called when they were written.

  **The demo app did change identity**: bundle ids moved to `com.pressboardkit.demo*`, the App Group to
  `group.com.pressboardkit.demo`, and the keyboard's display name to "PressboardKit Demo". Any
  simulator or device with the old demo installed needs it removed and the keyboard re-enabled — a
  bundle id is immutable once submitted, so this was the last free moment to fix it.

### Added
- **The keyboard appears sooner on a cold start** (ENG-182) — 303 ms to on-screen instead of 378 ms,
  with the work inside `viewDidLoad` halved from 162 ms to 74 ms (measured on the simulator). The merged
  dictionary is derived on a background task instead of on the way to the first frame; until it lands,
  roughly 160 ms after a cold start, the keyboard types normally and simply has no suggestions,
  autocorrect or predictive targeting — the same "no dictionary" fallback those features already
  describe. Every appearance after the first takes the finished dictionary from the process cache as
  before (ENG-160), so this costs nothing warm. Hosts that want to hold something back until suggestions
  work can watch `SwiftboardController.isDictionaryReady` or await `dictionaryReady()`.
- **Auto-correction in every language the system has a dictionary for — Czech among them** (ENG-171,
  ENG-173). `UITextChecker` has two halves that ship independently: on a Czech phone (iOS 27.0) its misspelling
  *finder* never flags anything, not even "codelas", while its *guesses* answer perfectly well
  ("vwcer" → "večer"). We asked through the finder, so Czech got no corrections at all. The two are now
  separate (`SystemSpellChecker.flagsMisspellings` / `.guesses(for:)`), and `LayeredSpellChecker` takes
  them in turn: the system's verdict where its finder works, otherwise its guesses, and otherwise the
  bundled word list, which is what remains where the system has no data. The Czech finder cannot be woken
  — six spellings of the language, seven inputs and both wrap settings produced not one hit (ENG-174) —
  so where it is dead its two jobs are done by the other parts: the completion lexicon says whether the
  word is real at all (it knew every correct Czech word tried, including ones a 40k list misses), and a
  guess is only applied when it is a letter on the wrong key or two letters in the wrong order. A guess
  that adds or drops a letter is left to the word list: for a word the system cannot call wrong, that is
  the shape that turns an unfamiliar word into a different one.
  The slip test (`TypingSlip`, `KeyNeighbors`) judges against the arrangement on screen, so a Czech
  QWERTZ keyboard treats QWERTZ neighbours as neighbours. Diacritics nobody types are restored in the
  same pass ("delas" → "děláš"), and a word valid in any enabled language is never rewritten.
- **The keyboard predicts the next word, and learns the pairs you write** (ENG-158, first step of ENG-73;
  KK#452/#495/#749/#993). Until now the bar sat empty the moment a word was finished — we completed the
  word being typed and ignored everything before it. New `NextWordPredicting` seam and a built-in
  `NextWordPredictor` that merges **learned** pairs (ranked first) with a **seed** bigram table a host can
  supply. Learning is what makes this useful with no shipped corpus, and it is what native does; it is
  bounded (`maxLearnedContexts`, oldest context evicted first) so it can't grow into the extension's
  memory budget, and it is handed to the host through `SwiftboardConfiguration.saveLearnedWordPairs` so it
  survives the extension being killed. Gated by `KeyboardBehavior.nextWordPrediction` (default on; a no-op
  until something is learned or seeded). Proven end-to-end in the real extension, through the App Group,
  by `NextWordPredictionUITests`.
- **Suggestion bar entries are readable and pickable by VoiceOver** — each is its own element, named by
  the word it inserts, with the highlighted correction announced as such. ENG-152 closed this hole for the
  keys; the bar had the same one.
- **The domain key offers every enabled language's TLD** (ENG-157). A Czech keyboard offers `.cz`, a
  German one `.de`, and a keyboard with both offers both — before this only Czech had a country TLD at
  all, and only when it was the primary language. New `KeyboardLocale.topLevelDomain` (nil where a
  language doesn't map to one country — English, Arabic, the Sámi languages — because guessing a country
  from a language is worse than offering nothing), `DomainKey.callouts(for: [KeyboardLocale])` and
  `DomainKey.maxCallouts`. The enabled set reaches the layout through the new
  `KeyboardContext.enabledLocales` and `StandardLayoutResolver.layout(…, locales:)`.
- **VoiceOver can now name and press every key** (ENG-152). Keys carried an automation identifier but no
  accessibility name, no keyboard-key trait and no activation path — the same hole that left
  KeyboardKit's keyboard unusable under VoiceOver (KK#644). Each key is now a single accessibility
  element with a readable name (`space`, `delete`, `shift`, `return`, the letter in its displayed case),
  the `.isKeyboardKey` trait, and an activation action that types directly — assistive activation
  delivers no touches, so neither input pipeline would otherwise see anything. The element is declared on
  the *composed* key (`KeyAccessibility` + `KeyGrid`), not inside `KeyboardButton`: declared on the inner
  view, SwiftUI kept publishing the key's own `Text` as a separate `staticText` element, so a letter key
  was announced only by accident and the space bar not at all. Guarded in the real extension by
  `KeyboardAccessibilityUITests`.
- **Suggestions in an empty field** (ENG-150, KK#546/#710/#712). Native offers three words in a field
  nobody has typed into; we showed an empty bar. `DictionarySuggestionProvider.openingWords` defaults to
  the primary language's three most frequent words, and a host can pass a curated set via
  `init(words:openingWords:)`. New `KeyboardBehavior.emptyFieldSuggestions` (default on = native).
- **English `i` → `I`** (ENG-151, KK#623) via `AutoCorrection.explicitCorrection`, applied before the
  "don't correct a known word" guard and exempt from the minimum-length rule. Deliberately English-only —
  Czech `i` is a word.

### Fixed
- **Auto-correction works in languages other than English** (ENG-170). We asked `UITextChecker` for the
  keyboard's bare locale code (`cs`, `de`), but it names its languages with a region (`cs_CZ`, `de_DE`)
  and matches them literally. An unlisted language isn't refused — the checker calls every word correctly
  spelled — so auto-correction silently did nothing outside English, which resolved by luck. The language
  is now resolved against the checker's own list, preferring the user's region between variants
  (`en_GB` vs `en_US`), and a language it genuinely has no dictionary for corrects nothing instead of
  pretending to (`SystemSpellChecker.resolvedLanguage` / `.isUsable` say which). Note that iOS ships no
  dictionary at all for some languages, Czech among them — see ENG-171.
- **The extension's memory grew ~10 MB on every keyboard appearance, never coming back** (ENG-160):
  measured on a real device, one process walked 23.5 -> 174 MiB over seven appearances — about four times
  a keyboard extension's budget — and was then killed. iOS builds a fresh input view controller each time
  the keyboard appears, so a fresh `SwiftboardController`, so a fresh `DictionarySuggestionProvider`; the
  source word lists are `static let` and load once, but each provider *derives* its merged list,
  `rankByLower` map and folded/sorted forms from them — ~10 MB of small string allocations for a merged
  CZ+EN keyboard. The old controller is released (its host's `deinit` log proves it), but freed small
  allocations do not hand their pages back. `makeProvider` now keeps a process-wide cache keyed by
  language set, bounded to three, bypassed when the host supplies its own words. Typing itself added
  ~1 MiB per 300 keystrokes, which is why every typing-based measurement missed this.

### Fixed
- **Long-pressing the domain key typed `.com` before you had picked anything** (ENG-156). On a URL field,
  holding ".com" to reach ".eu" wrote ".com" the moment the key went down, so the pick landed after it —
  ".com.eu" instead of ".eu". Two causes: the key is `.character(".com")`, so `insertOnKeyDown` treated it
  as an ordinary character key (key-down commit exists to make *typing* feel instant; a multi-character
  key is neither typed fast nor typed by accident, and native commits nothing on key-down), and the
  callout's undo sent a single backspace for a four-character insert. `KeyDownCommitLogic.
  shouldCommitOnKeyDown`'s `isCharacterKey` parameter is now `isSingleCharacterKey` to make the rule
  explicit. Proven in the real extension against Safari's address bar (`DomainKeyUITests`).
- **A suggestion bar appeared over number pads** (ENG-154, KK#499). The toolbar slot rendered whenever
  `predictiveText`/`showMathResults`/`emojiSuggestions` was on, regardless of the field — but native shows
  no QuickType row at all over a `numberPad`/`phonePad`/`decimalPad`, so we were making the keyboard
  taller than native for nothing. ENG-146 then made it visibly wrong, since `currentWord("123")` returns
  `123` and the bar started showing it. The bar is now suppressed there and no suggestions are computed
  per keystroke; gated on `adaptToInputType`, so a host that ignores the field type is unaffected. New
  read-only `SwiftboardController.suppressesSuggestionBar`.
- **Adding a `KeyboardBehavior` parameter threw away every saved setting** (ENG-153). With synthesized
  `Codable` a stored JSON must carry *all* keys, so settings a host persists (typically in the App Group
  it shares with its keyboard extension) failed to decode after any SDK update that added a flag, and
  silently reset to defaults — surfaced by this wave adding three parameters at once. `KeyboardBehavior`
  now has a hand-written `init(from:)` where every key is optional and falls back to the memberwise
  default.
- **The apostrophe didn't bring the letters back on the numeric page** (ENG-144, KK#332/#808).
  `LayoutAutoSwitch` returned to letters after a space or newline only. Measured against native on iOS
  26.4 by typing every key of the `123` page in turn (`NativePageReturnUITests`): of
  `' " - / : ; ( ) $ @ . , ? !`, the apostrophe is the **only** one that pages back — which fits what it
  is for (`don't`, `it's`). Straight and curly forms both count, since smart punctuation may have already
  swapped the typed one.
- **The "two spaces → `. `" shortcut had no time limit** (ENG-145, KK#161). It decided purely from text,
  so it fired after an arbitrarily long pause — or on a trailing space that was pasted rather than typed.
  Measured on iOS 26.4: native still fires at 0.75 s and no longer at 0.85 s. `PeriodShortcut.shouldExpand`
  now takes the time since the user's last space keystroke, with the window exposed as
  `KeyboardBehavior.periodShortcutWindow` (default 0.8 s).
- **The word being typed ended at an apostrophe or a digit** (ENG-146, KK#1030/#820).
  `SuggestionText.currentWord` took only `isLetter`, so `"don't"` came back as `t` and `"abc123"` as
  empty — English contractions could never be completed or corrected. A word now runs over letters,
  digits and *inner* apostrophes, while an opening quote is still not swallowed (`'hello` → `hello`).
- **A single character could be auto-corrected** (ENG-147, KK#456). Typing `g` offered a highlighted
  `get`, and `g` is not a known word, so the spell checker got a crack at it too — the defect that turned
  "Add 100 g of flour" into "Add 100 G of flour" for KeyboardKit. Auto-correction now leaves words shorter
  than `KeyboardBehavior.autoCorrectMinimumWordLength` (default 2) alone, and never rewrites a token
  containing a digit ("H2O", a flight number).
- **An orphaned backspace repeat could keep deleting after the finger lifted** in the per-key gesture path
  (ENG-149, KK#933). The multi-touch layer has guarded this since ENG-136; `KeyboardButton` still looped on
  `!Task.isCancelled` alone, which doesn't cover a touch that ends without either release or cancel (SwiftUI
  tearing the gesture down mid-press while the host re-lays out). It now also stops when the key is no
  longer pressed, and `reset()` cancels the tasks instead of just dropping the handles.

### Fixed
- **A multi-character key's pop had no room around its text** (ENG-169). The native preview is a square
  sized for one glyph, and ".com" filled it edge to edge. Such a pop now widens to its text, with padding
  derived from the point size rather than fixed; single-character pops are untouched, since theirs is
  already sized for one glyph and padding it would make ours larger than native's.

### Fixed
- **Callout selection did not follow the finger** (ENG-168). The highlighted cell was derived from the
  drag's displacement from where the touch landed, while the cells are laid out around the *key's centre*
  — so pressing a wide key off-centre put the highlight a visible distance from the finger, most obviously
  on ".com". Selection is now measured from the key's centre. The multi-touch path was also mixing a
  global key centre with a grid-local width, putting it a few points out of step with the per-key path;
  both are grid-local now.

### Fixed
- **The domain key's pop showed a truncated ".c…"** (ENG-166). A key-pop draws the key's glyph magnified,
  which is right for one character and impossible for four. A multi-character key now starts from the
  smaller control size — the same rule its key label and callout bar already use — via the new
  `KeyPopText.fontSize(for:metrics:magnification:)`. Whether native shows a pop on that key at all is
  still unmeasured; this fixes the rendering, not the parity question.

### Fixed
- **The host's space-bar label sat off-centre on the URL keyboard** (ENG-165). Reserving room only on the
  badge's side stopped the wrapping but pushed the label off the key's axis, which reads as a mistake.
  The reservation is symmetric again, paid for by a smaller badge on a short bar (8pt, tighter padding):
  every point the badge gives up, the label gets back twice over, and a minimum scale factor keeps the
  label on one line either way.

### Fixed
- **The host's space-bar label wrapped onto two lines on the URL keyboard** (ENG-164). Room for the badge
  was reserved on both sides to keep the label optically centred, which costs twice the width and left
  nothing for the text on a short space bar. It is now reserved only on the side the badge is actually on,
  and only when the bar is short — a wide space bar keeps its original centred look, since the two never
  collided there. The host's overlay also gets `lineLimit(1)` and a minimum scale factor, so anything
  drawn inside a key shrinks rather than wrapping or clipping.

### Fixed
- **The domain callouts still overflowed, and the space bar's badge moved to the middle** (ENG-163,
  correcting ENG-161). Fitting the bar by total width is not the same question as fitting it with its base
  anchored over the key: the room on each side is capped by the key's position, in whole cells, so
  rounding could still push the outermost TLD off-screen — which is what a device screenshot showed. The
  width now shrinks until the *actual* layout clears both margins, and the pitch is decided once at
  press-time and carried on the touch so selection and rendering cannot disagree. The language badge is
  back in its corner (a `.frame(width:height:)` without an alignment had centred it once the host's label
  was dropped), and the host's label is no longer dropped at all — on a short space bar the badge shrinks
  instead, so both stay visible.

### Fixed
- **The ".com" key's callouts overflowed the screen, and the space bar's brand label collided with the
  language badge** (ENG-161). The domain key offers up to six TLDs, which need the wide multi-character
  cell — seven cells at 58pt is 406pt on a 402pt phone, and `layout` overflows rather than dropping a
  character, so the far TLDs landed off-screen. New `CalloutLogic.fittedCellWidth` narrows the cells just
  enough to fit (never below 24pt, where the labels stop being readable). Separately, the space bar
  composed the host's overlay and the language badge in a plain `ZStack`, which is fine at full width and
  collides on the URL keyboard's narrower space bar; the badge is now measured and its width reserved on
  both sides, with the host's decoration dropped entirely below 44pt of remaining room.

### Fixed
- **Emoji category tabs only responded where their glyph was drawn, and the panel had no haptics**
  (ENG-141). The tabs measured 32x44pt but reacted in a ~16x18pt island at their centre: hosted
  out-of-process, a fully transparent region is dropped from the synchronized render tree and stops
  receiving touches (the ENG-120 trap), so only the drawn glyph was live. Fixed with the same rendered
  0.001-alpha fill `KeyGrid` uses. Separately, `SwiftboardRootView` never passed `onFeedback` to
  `EmojiKeyboard`, so the entire panel — tabs, ABC, backspace and the emoji themselves — was silent; it
  now uses the same `feedbackSettings` as the letter keys.
- **The emoji panel's category row didn't match native** (ENG-140). Measured against the real iOS 26 emoji
  keyboard and corrected: "abc" → "ABC"; the active tab is now marked by a filled 30pt circle
  (`rgb(43,43,46)` dark / `rgb(182,184,198)` light) with a brightened glyph instead of an accent-coloured
  icon; category glyphs are 12.5pt rather than 16pt (native renders them 15.7pt tall, ours were 20.0pt);
  and all four tints are set to their measured values. New `EmojiPanelView.selectedTabFill`,
  `selectedTabDiameter`, `tabIconColor`, `tabIconSelectedColor` and `controlKeyColor` expose them.
  `accentColor` no longer tints the active tab — native does not — and now only feeds the emoji search
  header's caret.
- **`Perf.logFootprint` wrote at `.info`, which the system never persists** (ENG-137): `log collect` from
  a device returned nothing at all after a full day of real use. Now `.notice`.
- **The space bar lost touches that native still accepts** (ENG-138): aiming at the space bar and getting
  the letter above it ("Ahoj jak" → "Ahojcjak") was a typo our keyboard made and native did not. Measured
  against the real iOS 26 keyboard: native's space bar claims touches up to **12.2pt above its own top
  edge**, while the boundary between two *letter* rows sits exactly on the geometric midpoint — so this is
  a space-bar-specific enlargement, not a global vertical nudge. New
  `KeyboardBehavior.spaceBarTopExtension` (default `12`); ours now measures 12.1pt.

### Fixed
- **A held backspace kept deleting after the finger lifted** (ENG-136): it wiped the whole message and
  kept ticking haptics. The repeat loop only stopped on `cancel()`, which is reached from the touch being
  released — but a touch can also leave without that, either under a recycled `UITouch` identity or when
  the gesture recognizer is torn down mid-touch as the host re-lays out (which deleting a long message
  reliably provokes). The loop now also stops as soon as its touch is no longer tracked, a stale entry is
  torn down rather than overwritten, and a new/reset recognizer drops any touch it inherited.

### Fixed
- **A long-press callout bar's pre-selected base character didn't sit above the pressed key** (ENG-135):
  on keys with many accents (e/u/o) it ended up at the far end of the bar, ~1.5cm from the finger, and the
  outermost accents couldn't be reached at all because the finger ran into the screen edge. The bar grew
  in one direction only and was then clamped to the screen, which broke the anchor it had just set. It now
  spills to **both** sides of the base cell, which stays over the key — measured against the real iOS 26
  keyboard, which does the same.

### Changed
- `CalloutLogic.selectedIndex(translationX:itemWidth:count:)` is replaced by
  `cellOffset(translationX:cellPitch:layout:)` + `itemIndex(atCellOffset:layout:)` (ENG-135), alongside
  new `layout(...)`, `displayItemIndices(layout:)`, `barCenterOffset(layout:cellPitch:)` and the
  `CalloutLayout` type. Cell offsets are now signed — a drag left selects as meaningfully as a drag right
  — and both selection and layout take the cell **pitch** (width plus inter-cell spacing) rather than the
  bare width, which used to drift by 2pt per cell across a wide bar.

### Fixed
- **Keys with no key-pop didn't visibly change when held** (ENG-134): backspace, return, space, emoji,
  123, globe and shift kept their idle fill apart from a `brightness(-0.06)` that is invisible in dark
  mode and the wrong direction in light. They now take a measured pressed fill, like native. Letters are
  unchanged — their key-pop is the feedback, and native doesn't recolour them either.

### Added
- `NativeColors` and `NativeEmojiRowStyle` (ENG-143), both carried by `NativeDesign`: the colours the
  keyboard paints with and the emoji category row's measured style now travel in the snapshot too, so a
  design snapshot covers the whole measured look rather than only its geometry. `NativeKeyboardStyle`
  takes a `colors:` (default `.iOS26`), `EmojiPanelView` takes an `emojiRow:`, and `KeyboardColor.uiColor()`
  bridges a measured colour into the UIKit-drawn parts. Closes the limit recorded against ENG-142.
- `NativeDesign` and `KeyboardBehavior.nativeDesignID` (ENG-142): the native look is now a dated,
  selectable snapshot rather than an unlabelled set of constants. Every parity number in the engine was
  measured against one iOS release, and Apple redraws the keyboard between releases; a snapshot names the
  release it describes so a future one can be added beside it. `nil` (the default) resolves automatically
  to the newest snapshot measured on an iOS no newer than the device's, and an unknown pinned id falls
  back to that rather than failing. One snapshot ships today (`NativeDesign.iOS26`) — deliberately; this
  is the seam, not a stock of designs. `KeyboardTheme.style(design:)` renders a theme in a given snapshot.
- `KeyboardBehavior.memoryDiagnosticsInterval` (default `0` = off) and `Perf.logFootprint(keystrokes:)` /
  `Perf.physFootprint()` (ENG-137): the keyboard can log its own memory footprint every N keystrokes.
  Diagnostic seam for the open question of whether prolonged typing grows an extension toward its memory
  limit — from outside, that is impractical to measure (Instruments' Allocations cannot attach to a
  process it did not launch, and a keyboard extension cannot be launched by it).
- `KeyboardStyleProvider.keyFillPressed(for:)` (ENG-134), defaulting to `keyFill(for:)` so existing
  custom styles are unaffected. `NativeKeyboardStyle` returns values measured against the real iOS 26
  keyboard: `rgb(193,194,199)` light, `rgb(127,127,128)` dark.

### Fixed
- **The space bar typed a space on key-down and deleted it again when a drag engaged** (ENG-132): visible
  as a flicker. Native writes the space only on lift-off, which is how it can still tell a tap from a
  cursor-drag. ENG-107's ordering fix is preserved by flushing a still-held space the instant another key
  commits, so real touch order is kept ("jak se" never becomes "jaks e").

### Added
- `KeyboardBehavior.spaceCommitsOnRelease` (default `true` = native, ENG-132) and
  `KeyboardBehavior.cursorDragEngageHaptic` (default `true`, ENG-132), plus
  `KeyboardFeedback.cursorDragEngaged()` (optional, default no-op) — native ticks when a spacebar press
  becomes a caret drag, **regardless of `FeedbackSettings.isHapticEnabled`**, because it signals a mode
  change rather than a keystroke. Needs Full Access in an extension, like any haptic.
- `KeyDownCommitLogic.shouldFlushHeldSpace(spaceIsHeld:spaceAlreadyCommitted:spaceDragActive:)` (ENG-132).

### Fixed
- **The caret vanished during a fast spacebar drag** (ENG-131): it stayed hidden while the finger kept
  moving and only reappeared once it stopped (slow drags were fine). A drag pushes a caret move per touch
  event — up to ~120/s on a ProMotion display — and each is a cross-process `adjustTextPosition`, which
  never left the host's text view an idle moment to repaint the caret. Moves are now accumulated and pushed
  at most once per `KeyboardBehavior.cursorDragUpdateInterval`. Pending movement is flushed before anything
  that depends on the live caret (a sentence jump, the end of the drag), so nothing lands stale or out of
  order.
- **A spacebar cursor-drag flooded the host app** (ENG-129): the caret stopped repainting mid-drag, taps
  into the text field were ignored, and the whole backlog applied at once on lift-off. `syncAfterInput()`
  reads `documentContextBeforeInput` — a *synchronous cross-process round-trip* on a real host — and ran on
  every drag step (one per 8pt, many per second), saturating the host's main thread. It is now skipped
  entirely while `isMovingCursor`; the caret still moves via `adjustTextPosition`, and
  `cursorMoveChanged(false)` does one re-sync for the final position.

### Changed
- `KeyDownCommitLogic.shouldCommitOnKeyDown` takes a new `deferSpaceToRelease:` argument (ENG-132).
- **Spacebar-drag vertical movement now moves by sentence, not by an estimated visual line** (ENG-128).
  `KeyboardAction.moveCursorRow(Int)` → `.moveCursorSentence(Int)`, `SpacebarDragLogic.row(translationY:
  pointsPerRow:)` → `.sentenceStep(translationY:pointsPerSentence:)`, `SpacebarDragLogic.defaultPointsPerRow`
  → `.defaultPointsPerSentence` (and `24` → `40`pt), `SpacebarDragLogic.characterOffset(forRows:before:
  after:estimatedVisualLineLength:)` → `.characterOffset(forSentences:before:after:)`. A keyboard extension
  sees only a *truncated* window of the document through `UITextDocumentProxy` and no text geometry at all,
  so a character-count estimate of a wrapped line mis-fires against real hosts; sentence stops are derived
  purely from punctuation that is actually visible, so the move is deterministic. Same trade-off KeyboardKit
  10.6 makes for its vertical spacebar drag. The raised threshold keeps the natural vertical drift of an
  ordinary sideways drag from firing a jump mid-drag. Horizontal dragging is unchanged.

### Removed
- `KeyboardBehavior.estimatedVisualLineLength` and `SpacebarDragLogic.defaultEstimatedVisualLineLength`
  (ENG-128) — the guessed parameter they existed to tune is gone, not re-tuned. Nothing replaces them;
  vertical drag needs no geometry estimate now.

### Added
- `KeyboardBehavior.cursorDragUpdateInterval` (default `1/30` s, ENG-131): how often, at most, a spacebar
  cursor-drag pushes the caret to the host. `0` pushes every step immediately (the old behaviour).
- `SpacebarDragLogic.sentenceBoundaries(in:)` (ENG-128): every caret offset that counts as a sentence stop
  in a string — the start, the position after each run of sentence-ending punctuation (`.` `!` `?` `…`,
  where "?!" is one ending), the position after each hard newline, and the end.
- `SpacebarDragLogic.step`'s new `acceleratedPointsPerStep`/`accelerationDistance` parameters (defaults
  `4`pt and `120`pt, ENG-127): a horizontal spacebar drag now moves faster the further it travels, instead
  of a flat linear rate — see the "Fixed" entry below. Pass `accelerationDistance: .greatestFiniteMagnitude`
  to fully revert to the old flat-linear mapping.
- `KeyboardBehavior.slideToTypeRestoresDiacritics` (default `true`, ENG-126): a decoded slide-to-type
  match is inserted with its dictionary diacritics restored ("máš") rather than the folded, base-letter
  form actually swiped ("mas"). Both are deliberate: `SwipeDecoder` now matches diacritic-insensitively
  (a glide only ever crosses a keyboard's base-letter keys — diacritics are a long-press callout, never
  swiped through), so a Czech glide over "m", "a", "s" finds the dictionary entry "máš". Measured
  against the native iOS keyboard, native inserts the folded "mas" — never restores diacritics on a
  swipe; this default deliberately diverges (the accented word is grammatically correct and free to
  restore once matched), and the flag is there to reproduce native exactly when a host wants that
  instead. `DictionarySuggestionProvider.candidateWordsRanked` — a swipe candidate list paired with each
  word's rank *within its own source language list* (not its position in the merged/concatenated
  array). `SwiftboardController.makeProvider` now builds the multi-language dictionary via the new
  `DictionarySuggestionProvider.init(languageLists:)` (one list per language, unchanged merge/dedup
  behaviour) instead of pre-concatenating, so per-language rank survives the merge.
- `KeyboardBehavior.suppressPunctuationSpacingAfterEdit` (default `true`): once the user backspaces a
  `punctuationSpacing` fixup all the way back to the plain word it moved a space past and retypes it
  their own way, that spot stops refixing — whatever mark they retype, not just an emoticon. Survives a
  retyped space (unlike `autoSpaceSuppressed`/`autocorrectSuppressed`, on purpose: redoing a rejection is
  naturally "space, then the mark", and resetting on that space would erase it first). Cleared by moving
  the cursor elsewhere. See ENG-125.
- `KeyboardBehavior.emoticonAwarePunctuation` (default `true`): keeps a typed emoticon intact when
  `punctuationSpacing` is on. `:` and `;` are sentence marks to that fixup, so "hi :)" used to come out
  "hi: )" — and retyping it hit the same fixup again. The mouth can't be known at the mark, so the move
  is undone when the very next character completes a face (new `Emoticons` pure logic; `:` / `;` openers,
  `) ( ] [ } { D P p O o 0 3 S s X x b B c L , # +` mouths plus the `- ^ '` nose/tear — "v" deliberately
  left out: it's a complete, very high-frequency Czech preposition on its own, not just a common first
  letter, so a false positive is meaningfully more likely there than for the rest). A space between mark
  and mouth closes the one-action window, so real punctuation is untouched; the window also survives a
  page switch/shift between the mark and the mouth ("hi :D" needs both) — only an action that actually
  writes/deletes text or moves the cursor closes it. See ENG-123, ENG-125.
- `KeyboardBehavior.predictiveKeyTargeting` (default `false`, unchanged behavior): biases a borderline
  hit-test toward whichever same-row letter key keeps the word typed so far a viable dictionary prefix —
  measured on-device, typing "keyboar" then tapping just left of "d" (geometrically closer to "s") still
  resolves to "d" on native, since "keyboard" is a real word and "keyboars" isn't. Only reachable under
  `multiTouchInputLayer` (`KeyHitTestGeometry.key`); never reassigns a touch that lands comfortably inside
  a key, and is a no-op at the start of a word or with no dictionary for the active language. New
  `KeyTargetBias` (pure logic) and `DictionarySuggestionProvider.isViablePrefix` (O(log n)). See ENG-119.
- `KeyboardBehavior.touchHitTestVerticalOffset` (default `0`, unchanged behavior): shifts the effective
  hit-test point up from the raw touch location before matching it to a key, under `multiTouchInputLayer`.
  Native reportedly applies a similar correction; the value here is deliberately left at `0` (unmeasured)
  rather than guessed — see ENG-110.
- `DemoApp/DemoAppUITests` — a real XCUITest harness (ENG-121) that taps exact key-grid coordinates
  (dead-zone gaps, a seam sweep, predictive-targeting boundary points) and reads back what got typed,
  against both DemoApp's in-process live preview and a **real, out-of-process keyboard extension**
  (switched to via another app). `KeyboardTestGeometry` looks keys up by the stable identifier every
  `KeyboardButton` now carries (`accessibilityIdentifier`, e.g. `key.character.s`) and taps their actual
  rendered rects — reusable by an integrator's own UI tests, not SwiftboardKit-specific. See
  `INTEGRATION.md` §19.
- `DemoApp/DemoAppUITests/GestureTimingThresholdUITests` (ENG-122) — measures the long-press callout
  threshold and the slide-to-type glide-engage distance against the real native keyboard, in the same
  host app (Reminders) as our own real, out-of-process extension, so the two are directly comparable
  (screenshot-diffing mid-hold for the timing one via a `Timer` on the main run loop — `press(forDuration:)`
  pumps it while waiting — and plain typed-text-length deltas for the distance one). Exploratory/slow
  measurement tooling, not a fixed pass/fail spec; `SlideToTypeEngageUITests` is the fast, permanent
  regression guard for the distance side once tuned.
- `KeyboardButton.accessibilityIdentifier`: every key now exposes a stable identifier
  (`key.character.<lowercased letter>`, `key.shift`, `key.space`, `key.backspace`, `key.primary`,
  `key.nextKeyboard`, `key.dictation`, `key.type.<page>`, `key.custom.<name>`) — no effect on rendering
  or VoiceOver, just UI-test automation (built for the harness above, useful for a host's own tests too).

### Changed
- `KeyTouchStateMachine.glideEngageKeyCount` (multi-touch slide-to-type engage gate): `3` → `2`, measured
  against native (ENG-122): a straight-line drag along the home row flips from "tap" to "glide" on native
  at ~19–24pt, vs. our previous ~45pt (same measurement method, in the same host app, against native and
  our real out-of-process extension). `glideAbandon` (12pt) was never the binding constraint and is
  unchanged. Closes most of the gap in the horizontal direction (now ~26pt); a straight-down drag didn't
  move (~45pt still, vs. native's ~38pt) for reasons not yet root-caused — see ENG-124.
- `NativeMetrics.calloutLongPressThresholdNanoseconds` (new): the long-press-for-diacritics delay,
  previously an untraceable `300_000_000` duplicated in both `KeyboardButton.scheduleCallouts` and
  `KeyTouchTracker.scheduleCallouts`. Value unchanged (300ms) — measured against native (ENG-122) via a
  new XCUITest harness and found statistically indistinguishable from native at this harness's own
  resolution once its ~280–300ms self-measured overhead is accounted for (see the constant's doc comment
  for the full self-calibration).
- `KeyboardBehavior.swiftboardDefault` now enables `predictiveKeyTargeting` too. `.standard` still
  doesn't — it changes what a borderline touch types, so integrators opt in.
- **Breaking**: `CharacterPreviewStyle.native` — the old shape (a key that flares out of the pressed key
  via a connecting neck, blanking the key's own letter) is renamed to `.connected`. A **new** `.native`
  takes the `native` name and matches the real iOS key-pop, re-measured against a device across two
  passes (ENG-110): a standalone **square** (`NativeMetrics.nativePreviewSize` = `keyRowHeight * 1.35`,
  ~60×60pt — fixed regardless of the pressed key's width, matching native holding it square even over a
  narrow key) floating `nativePreviewGap` (`keyRowHeight * 0.14`, ~6.3pt) above the pressed key, no neck,
  with a lighter fill (`nativePreviewLighten` — measured, `0.087`); the pressed key itself stays visible
  (letter + a lightened fill, `nativePressedKeyLighten` = `0.2`, also measured — native's preview is
  *darker* than its pressed key, not lighter) instead of blanking. Since this pop is taller than the
  connected styles', the host now reserves headroom per active style (`NativeMetrics.popOverflow(for:)`)
  rather than the flat `keyPopOverflow`, which stays exactly as it was for `.connected`/`.curvy`/
  `.square`. Persisted raw string `"native"` now resolves to the new shape — intentional, since it's the
  actually-native one; hosts that specifically want the old connected shape should switch to
  `.connected`.
- `NativeMetrics.iPhonePortrait.horizontalPadding`: `3` → `6.5`pt, re-measured against a device — the old
  value put every key edge up to ~3.4pt off native, which read as mis-hits near the edges of letters.
  `iPhoneLandscape`'s padding is still the old unverified `3` (not remeasured in this pass).
- `KeyboardBehavior.swiftboardDefault` now enables `multiTouchInputLayer`. `.standard` still doesn't, so
  integrators stay on the per-key gesture path until the new layer has more mileage.

### Fixed
- **Spacebar cursor-drag: a vertical "row" move jumped to the very start/end of the whole text instead of
  moving one line, and horizontal movement was flat/linear instead of speeding up with distance** (ENG-127).
  `SpacebarDragLogic.characterOffset(forRows:)` only recognized real `"\n"` as a line break — a plain
  message typed in one run (the common case: no `"\n"` at all) has none, so an up/down drag jumped straight
  to the very beginning/end of the entire text instead of moving one visual line. A long `"\n"`-delimited
  paragraph had the same issue in miniature: it could skip several of its own wrapped visual lines in one
  move. Fixed by modeling "visual lines" as real newline-delimited runs further chopped into fixed-size
  chunks of `KeyboardBehavior.estimatedVisualLineLength` characters (a necessary estimate — the engine has
  no text geometry) — both branches (with and without real newlines) are now bounded by the same estimate,
  and the column is preserved across the move as before. Separately, `SpacebarDragLogic.step` (horizontal
  drag → character count) mapped `translationX` to characters at one flat rate; it now stays at the
  existing native-calibrated rate for the first `accelerationDistance` (120pt) of the drag, then moves
  twice as fast beyond it — approximating native's "slow = fine per-character control, fast = swift
  traversal" feel via distance travelled (a pure, deterministic proxy) rather than measured velocity, which
  would need wall-clock time and per-frame smoothing threaded into what's deliberately a pure, UIKit-free,
  fully unit-tested type.
- **Slide-to-type decoded noticeably worse than the native keyboard on a multi-language (CZ+EN)
  keyboard** (ENG-126): a real device swipe of "ahoj jak se mas co delas" decoded as "ahoj jfk ser mass
  chi deeds" — almost entirely wrong, English, and consistently *longer* than the intended word.
  `SwipeDecoder.decodeCandidates` ranked matches by **length first, frequency only as a tie-break**
  ("a glide crosses every letter, so the fullest word is the most likely") — since a glide's crossed-key
  path tends to be a superset-ish sequence, this let any longer subsequence match beat the actually
  intended, more frequent, shorter word ("co" → "chi", "se" → "ser"). Compounding it, the multi-language
  provider concatenates language word lists (primary language first) into one flat, frequency-ordered
  array — so with English listed first on an EN+CZ keyboard, *every* English word ranked ahead of *every*
  Czech word regardless of real frequency, and Czech words with diacritics (the dictionary's actual
  spelling, e.g. "máš") were never reachable at all by a glide over their un-accented keys ("m", "a",
  "s"), since matching compared literal, unfolded letters.
  - Ranking is now **frequency-first, length only a weak signal**: a log-scaled score
    (`-log(rank + 2)`, steepest at low ranks — matches how skewed real frequency lists are) with a small
    penalty for how far a candidate's length deviates from the number of keys crossed
    (`lengthDeviationPenalty`), tuned so it only breaks a near-tie in frequency and can never let an
    implausible length outrank a much more frequent word.
  - Matching is now **diacritic-insensitive** (`SwipeDecoder.fold`), so an accented dictionary word is
    reachable by a glide over its base-letter keys; the decoder still returns the word's actual spelling.
  - Multi-language fairness: candidates now carry a rank **within their own source language list**
    (`DictionarySuggestionProvider.candidateWordsRanked` / `init(languageLists:)`,
    `SwipeDecoder.RankedCandidate`), not a position in the concatenated array — a common Czech word can
    now beat a rare English one even with English listed first. A word present in more than one enabled
    language (rare, but real — e.g. "se" also appears, very rarely, in the English corpus) keeps its
    **best** rank across the languages it's in, not whichever language happened to be scanned first.
  - Verified against the real bundled 40k EN/CS dictionaries (not a toy word list) — see
    `SwipeRealDictionaryTests`, including the exact regression sentence.
- `emoticonAwarePunctuation`'s one-action undo window (ENG-123) was closing on *every* action, including
  ones that don't touch the document — so "hi :D" only worked if no page switch/shift stood between the
  mark and the mouth. "hi :)" / "hi :-)" survived (mark and mouth both live on the numeric page), "hi :D"
  didn't (needs a switch back to letters plus shift for the capital D). The window now only closes on an
  action that writes/deletes text or moves the cursor. See ENG-125.
- Touches landing in the gaps between keys are no longer swallowed under `multiTouchInputLayer`. The key
  grid is a bare `Layout`, hit-testable only where its keys are, so SwiftUI never delivered those touches
  and the snap-to-nearest hit-test never ran — a dead strip between every pair of keys that native
  doesn't have. The grid now carries its own hit area (`GridHitArea`, ENG-120) — **not**
  `.contentShape(Rectangle())`: that (and a `Color.clear` background) closed the gap in DemoApp's
  in-process live preview, but measurably **not** in a real, out-of-process keyboard extension — see the
  new `DemoAppUITests.KeyboardExtensionUITests` (ENG-121) harness, which reproduced the dead zone there
  and pinned it down to exactly `keySpacing`'s width. A `Rectangle` with a near-zero-but-nonzero opacity
  fill closes it in both; a fully transparent shape/hit-test-only hint appears to get pruned from the
  render tree that extension hosting synchronizes across processes.
- The `#+=` page label is sized like native's, which draws it smaller than `ABC` on the same key. The
  descender-free page labels (`ABC`/`123`/`#+=`) also take the smaller caps nudge — the letter nudge left
  them a measured 0.66pt high.
- Uppercase letters match native, which steps the font size down when shifted (its W/w ratio is 1.22
  against the typeface's natural ~1.4) — ours stood ~14% taller. New `uppercaseFontSize`, plus a smaller
  optical nudge for caps, which have no descenders to compensate for.
- Digits and symbols on the number/symbol pages are sized like native, which draws them smaller than
  letters — ours stood 17.7pt against native's 15.4. New `NativeMetrics.numericFontSize` applies to
  character keys off the alphabetic page.
- Control-key glyphs match native's size: shift/delete stood 14.0pt against native's 17.0/16.7, and the
  "123" label 11.7 against 13.0. SF Symbols now have their own `symbolFontSize` — one point size can't
  render a symbol and text at the same visual size.
- Key letters are optically centred like native. SwiftUI centres `Text` on its line box, which reserves
  descender room even for glyphs that have none, so letters sat a measured 2.33pt low; text labels now
  carry a font-size-proportional nudge (SF Symbols, already optically centred, don't).
- Key corners now match native: measured against a real iOS 26 key (7.5pt circular fit vs our 5.1pt) and
  calibrated by rendering, since a `.continuous` corner draws smaller than the declared radius.
- Dark-mode keys are no longer ~10 levels darker than the real iOS keyboard: the translucent key fill's
  alpha is calibrated (0.18) so it composites to the measured native shade, and control keys share the
  letter keys' alpha — iOS 26 no longer draws them darker.
- The key-pop and callout bar could sit offset from the key they belong to after the keyboard was
  re-hosted (switching to another keyboard and back). Their horizontal placement no longer depends on a
  measured global frame that goes stale; `KeyGrid` passes each key its own rect instead. Also clamps to
  the key area rather than the full screen, which is the correct bound in one-handed mode.
- Slide-to-type can now start on a key that has accents: the pending long-press callout timer is
  cancelled once the finger sets off, so the accent bar no longer opens mid-swipe and take over the
  touch. Holding still still opens it, matching native, where both features share the same keys and are
  told apart by movement.
- Multi-touch input layer (`multiTouchInputLayer`): fast typing no longer drops characters. The
  gesture's backing view (and its subtree) now has `isMultipleTouchEnabled` set, the recognizer is
  built through its designated initializer so `cancelsTouchesInView`/`delaysTouches*` actually apply,
  the competing keyboard-level slide-to-type gesture is not attached under this layer, and a touch
  landing in the gap between keys snaps to the nearest key instead of being ignored.
- Added an experimental multi-touch input layer, slice 1 (ENG-108): fast multi-touch typing on-device
  still reordered/dropped characters even after ENG-107's ordering fix, because SwiftUI's per-key
  `DragGesture` can't keep up with genuinely overlapping touches (they can be delivered out of real order,
  or dropped). New `KeyboardBehavior.multiTouchInputLayer` (default **false** — the existing per-key
  gesture pipeline, unchanged, stays the default in both `.standard` and `.swiftboardDefault`) routes key
  presses through a single shared multi-touch gesture (`KeyTouchGesture`, a custom `UIGestureRecognizer`
  attached via `UIGestureRecognizerRepresentable` — a self-inserted UIKit view/recognizer never receives
  touches inside a real keyboard extension's cross-process hosting, only SwiftUI's own gesture channel
  does) instead, processing raw `UITouch` events ordered by `UITouch.timestamp` rather than delivery
  order (`KeyTouchTracker`, `KeyTouchStateMachine`). Keys still render in SwiftUI (identical look) but read
  their press/pop/pulse state from a new per-key `KeyVisualBox` instead of their own `@State`, driven by
  the tracker. **Slice 1** (this change): key-down character/space commit (`KeyDownCommitLogic`, unchanged
  — this is what makes ENG-107's ordering fix actually land in the right order under fast multi-touch),
  backspace press-down + accelerating hold, drag-off cancel, and the key's own visual/haptic/bubble
  feedback. **Not yet wired** under this flag (later slices): long-press callouts, the spacebar
  cursor-drag, and slide-to-type — a key that would normally use one of those behaves like a plain
  tap+release for now. New pure, unit-tested `KeyHitTestGeometry` (hit-test rects for every key,
  generalizing `KeyGrid.letterFrames`) and `KeyTouchStateMachine` (tap/drag-off/ordering decisions, shared
  with the existing per-key path). All-or-nothing per `KeyGrid` — the two pipelines are never mixed within
  one keyboard.
- Multi-touch input layer, **slices 2–4** (ENG-108, still `multiTouchInputLayer`): the three behaviors
  slice 1 left as a plain tap+release now work under the flag too, each covered by the same per-touch
  state machine (`KeyTouchTracker`'s `ActiveTouch`) rather than a second gesture. **Slice 2** (callouts):
  the long-press timer, drag-to-select (`CalloutLogic.selectedIndex`, plus a new `CalloutLogic.
  extendsRight` shared with the per-key path's bar-direction check), and release-commit (replacing the
  key-down base character with the picked accent) — driven through `KeyVisualBox.showCallouts`/
  `selectedCallout`/new `revealCallouts(keyAnimationsEnabled:)`. **Slice 3** (spacebar cursor-drag):
  `SpacebarDragLogic` + `CursorMoveState` blanking + the ENG-107 undo-on-engage semantics, wired through a
  new `cursorState`/`onCursorMove` pair on `KeyTouchGesture`/`KeyTouchTracker`. **Slice 4** (slide-to-type):
  a touch that starts on a letter, travels past the abandon distance, *and* crosses ≥3 distinct letters
  (`KeyTouchStateMachine.glideEngaged`, `SwipeKeys.keys(forPath:keyFrames:)`) switches into glide — undoing
  its key-down character, emitting `.slideChanged(keys:)` as new letters are crossed, and `.slideTyped
  (keys:)` on release. `KeyGrid`'s keyboard-level swipe `DragGesture` stays intentionally **un-attached**
  under this flag (see slice 1) — slide-to-type now lives entirely inside the tracker, so there is still
  never a second gesture competing for the same touch. 7 new tests (`CalloutLogic.extendsRight`,
  `KeyTouchStateMachine.glideEngaged`); `KeyTouchTracker`/`KeyTouchGesture`/`KeyVisualBox` remain the UIKit
  rendering-layer test exception (verified in DemoApp on-device). The old per-key gesture pipeline
  (`multiTouchInputLayer == false`) is unchanged.
- Fixed character ordering around a space under `insertOnKeyDown` (ENG-107): letters committed on
  key-down, but space only committed on release, so typing a letter's key-down before releasing the
  previous space could land it *before* that space ("jak se" → "jaks e"). The space bar now commits on
  key-down too (extending `insertOnKeyDown`'s existing semantics, no new parameter), while a spacebar
  cursor-drag still works correctly — the moment the drag engages, a key-down space is undone (mirrors the
  letter-glide undo), and the generic "dragged off the key cancels it" rule no longer applies to space (a
  real drag-off is already caught by the engage check, so a leftover "still inserted" there would only be
  a false reading from the key-down space's own re-layout, not an actual drag — and must not eat the space
  just typed). Pulled the decision logic into a new pure, unit-tested `KeyDownCommitLogic`.
- Added `KeyboardBehavior.keyAnimationsEnabled` (ENG-106, default **true** = native): an experimental
  switch to test whether the key-press *visual feedback animations* (the bubble-style "push in" pulse,
  the long-press callout bar's reveal) make typing read as slower than it is, even though character
  insertion (`insertOnKeyDown`) is already instant. When `false`, those transitions apply immediately
  instead of animating. Scoped to key-press interaction only (not the emoji panel). Demo toggle under
  "Feedback & previews".
- Fixed the bubble-style pressed-key shrink not showing on device (ENG-105): it was tied to `isPressed`
  via an implicit animation, which a fast `insertOnKeyDown` tap released before the spring could run (and
  which didn't re-fire reliably across renders — it read as "only the first key shrinks"). The shrink is
  now an explicit one-shot pulse fired from the character insert, so it plays fully on every key.
- Deeper pressed-key shrink under bubble styles (ENG-104): the pressed character key now scales to 0.8
  (was 0.92) for a clearer "pushed-in" feel. Still scoped to bubble preview styles.
- `.eu` in the domain-key callouts (ENG-103): the URL keyboard's ".com" long-press now offers `.eu`
  on every language (Czech already had it).
- Bubble preview can't get stuck (ENG-102, strengthens ENG-93): the field now also resets when the
  character-preview style changes (so switching away from a bubble style — e.g. to native — always takes
  effect and can't stay stuck showing nothing) and via a new public `resetBubblePreview()` that hosts call
  on `viewWillAppear` (a reliable recovery point after unlock / app-switch, more so than the lifecycle
  notifications alone). Unit-tested.
- `KeyboardLocale.hasBundledDictionary` (ENG-101): a public flag telling whether a locale ships a bundled
  word-frequency list (predictions + swipe work) vs. types without predictions. Guarded by a test that it
  matches `DictionarySuggestionProvider.builtInWords(for:)` being non-empty, so it can't drift. Lets a host
  gate dictionary-dependent UI/behaviour per the active languages.
- 18 more languages — everything on the request list except Cherokee (ENG-100): the remaining base
  languages Filipino, Uzbek, Inari Sami and Shughni Tajik (no frequency list — they type, no predictions
  yet; Shughni Tajik reuses the Russian rows with Tajik letters on callouts), plus the regional/layout
  variants Arabic (PC), Dutch (Belgium), English (Australia/Canada), French (Canada/Belgium/Switzerland),
  German (Austria/Switzerland), Kurdish Sorani (Iraq/PC), Norwegian Nynorsk and Spanish (Latin America/
  Mexico) — each reuses its base language's layout, dictionary and callouts. **Cherokee** remains out (its
  85-glyph syllabary doesn't fit the 3-row model). SwiftboardKit now covers 73 languages / 7 scripts.
- 15 more languages — regional variants, niche Latin, extended Cyrillic, Kurdish (ENG-99): English (UK),
  English (US), Portuguese (Brazil), Serbian (Latin) and Vietnamese ship bundled lists (Serbian Latin
  and pt-BR fetched, en-GB/US reuse the English list); Faroese, Maltese, Swahili, Welsh, Irish, Northern
  Sami and Hawaiian reuse a Latin layout (no source list yet, so they type without predictions). Kazakh
  and Chuvash reuse the Russian rows with their extra letters (ә/ғ/қ…, ӑ/ӗ/ҫ/ӳ) on callouts; Kurdish
  Sorani gets its own Arabic-based rows. **Cherokee is intentionally not added** — its 85-glyph syllabary
  doesn't fit the 3-row keyboard model and needs a different keyboard UI. SwiftboardKit now covers 55
  languages across 7 scripts.
- Armenian, Georgian, Persian (ENG-98): three more scripts. Georgian and Persian ship bundled 40k
  FrequencyWords lists; Armenian has no source list (types, no predictions). Each gets its own alphabet
  rows (Armenian և and Persian hamza/variants on callouts); Persian is Arabic-based (پ چ ژ گ), LTR keys /
  RTL text like Arabic. Unit-tested. SwiftboardKit now covers 40 languages across 6 scripts (Latin,
  Cyrillic, Greek, Arabic, Hebrew, Armenian, Georgian).
- Greek + more Cyrillic + RTL scripts (ENG-97): Greek, Belarusian, Mongolian, Arabic and Hebrew.
  Greek/Arabic/Hebrew ship bundled 40k FrequencyWords lists; Belarusian and Mongolian have no source
  list, so they type but don't predict yet. Each gets its own alphabet rows (Greek accents + Arabic
  hamza on callouts). **No engine RTL is needed** — the keys are laid out LTR like the native iOS Arabic/
  Hebrew keyboards and the host text field renders the inserted text RTL. Unit-tested (alphabet coverage
  + list loading). SwiftboardKit now covers 37 languages across Latin/Cyrillic/Greek/Arabic/Hebrew.
- Cyrillic script — 5 languages (ENG-96): Russian, Ukrainian, Bulgarian, Macedonian and Serbian
  (Cyrillic). The letter rows are now **locale-driven** (`CharacterRows.rows(for:layoutType:)`) so a
  non-Latin script gets its own alphabet table regardless of the (Latin) `layoutType`; each language's
  full alphabet is on keys, with the few extras on callouts (ё/ъ on Russian е/ь, ґ on Ukrainian г).
  Bundled 40k FrequencyWords lists; Serbian is transliterated Latin→Cyrillic at build time (deterministic
  for Serbian). Locale-aware quotes (« » for ru/uk, „ “ for bg/mk/sr). No RTL needed (the host renders the
  field). Unit-tested (alphabet coverage + list loading).
- 22 more languages — Latin-script wave 1 (ENG-95): new `KeyboardLocale` cases for Italian, Polish,
  Portuguese, Dutch, Swedish, Danish, Norwegian, Finnish, Turkish, Romanian, Croatian, Slovak, Slovenian,
  Hungarian, Estonian, Latvian, Lithuanian, Catalan, Albanian, Icelandic, Indonesian and Malay. Each ships
  a bundled 40k FrequencyWords (OpenSubtitles) list for predictions + swipe, reuses an existing key
  arrangement (QWERTY, or QWERTZ for hr/sk/sl/hu) and the shared accent callouts (which already cover every
  letter these languages need — long-press reaches them all), and gets locale-aware smart quotes. No engine
  or API change beyond the new enum cases. Follow-ups: front-load each language's primary accent for native
  parity, and non-Latin scripts (Cyrillic / Greek / Armenian / Georgian / Arabic / Hebrew) as their own
  layout tables.
- Bubble preview keys now read as pressed (ENG-94): under the bubble preview styles
  (`bubble`/`bubbleWord`/`bubbleSentence`) the letter rides a keyboard-level bubble instead of a per-key
  pop, so the pressed key never merged and kept showing its glyph unchanged — the press didn't register.
  Now, while a character key is held in a bubble style, its glyph is hidden (it's in the bubble), the key
  darkens (a firmer `-0.12` vs. the native control-key `-0.06`), its bottom depth is dropped, and it
  shrinks slightly (`scale 0.92`, a quick spring — the standard "pushed-in" press affordance) so it reads
  clearly pressed — matching what the connected pop styles already do (minus the pop). Restores instantly
  on release (the bubble floats on). The scale is scoped to the single pressed key, so it doesn't
  reintroduce the per-keystroke animation cost the key-pop avoids. Rendering-only (`KeyboardButton`); no API change.
- Bubble preview survives lock/unlock (ENG-93): `BubbleField.reset()` and `SwiftboardController` now
  observe `NSExtensionHostDidEnterBackground` / `WillEnterForeground` and clear the field on each. While
  the device is locked the media clock and SwiftUI's animation timeline stall; without a reset the bubble
  preview (esp. `roam`) could come back with the timeline stuck "running" (a frame's work every tick →
  laggy keys) and stale bubbles that never render. Clearing on background means the next keypress starts a
  clean animation. No API change beyond the new public `BubbleField.reset()`.
- Keep the user's edit after auto-correct (ENG-92): new `KeyboardBehavior.suppressAutocorrectAfterEdit`
  (default `true` = native). Once a word has been auto-corrected, backspacing into it to change it no
  longer re-applies the correction — the keyboard keeps what the user typed until they move on (space /
  return / cursor move), matching the native "I rejected the correction" behaviour. Only relevant when
  `autoCorrection` is on; set `false` for the old always-correct behaviour. New DemoApp toggle; unit-tested.
  No breaking changes (new field defaults to native).
- Colour themes (ENG-91): new `KeyboardBehavior.theme` (`KeyboardTheme`) — a pickable, `Codable`,
  `CaseIterable` enum with `.native` (default, unchanged system look incl. liquid glass) plus `.midnight`,
  `.ocean`, `.sunset`, `.forest`, `.graphite`. Each non-native theme recolours the *same* renderer from a
  single `ThemePalette` (light+dark) via `ThemedKeyboardStyle`, so keys, text, accent, bubbles, callouts,
  the key-pop and the suggestion bar all follow the theme; themed keyboards are solid (opt out of the
  translucent backdrop). Resolve with `theme.style()`, or build a custom look via `ThemePalette` +
  `ThemedKeyboardStyle`. A non-native theme also tints the extension's container view (the rounded top
  "tray" that is otherwise the clear system backdrop) via a dynamic `UIColor` in `SwiftboardInputViewController`,
  so the whole keyboard reads as themed, not just the keys; both the solid SwiftUI background and the
  container tray round their top corners (radius 12) so a themed keyboard has the same rounded top as the
  system tray. Also fixes accent dynamic dispatch: `accentFill()`/`accentTextColor()` are now
  protocol requirements (defaults kept in the extension), so an injected style's overrides are honoured
  through the `any KeyboardStyleProvider` existential. New DemoApp theme picker; unit-tested. No breaking
  changes (new field defaults to `.native`).
- Two new bubble preview styles (ENG-90): `characterPreviewStyle.bubbleWord` and `.bubbleSentence`,
  built on the existing bubble overlay (shared `Bubble` + rendering + pop; only the motion/lifecycle
  differ). Instead of roaming, bubbles settle into a row on the keys↔autocomplete boundary (upper half
  over the bar) and spell what you type. **Word**: letters accumulate; ending the word (space/punctuation)
  bursts the whole word at once and drops a separator bubble that pops when the next word starts.
  **Sentence**: same lineup, but every character pops on its own short (typing-speed) timer — a sliding
  tail of the text. Bubbles emerge directly in the band (above the pressed key's column) and slide into
  place; the row is centred while it fits and shrinks to span the full width once it overflows; backspace
  pops the last bubble. Sentence bubbles are longer-lived than the roaming machine so the tail lingers.
  New pure
  `BubblePhysics.settle` (lineup spring) + `BubbleMotion` are unit-tested; the field lifecycle is
  exercised via the DemoApp (new picker entries). No breaking changes.
- Faster typing with the character preview on (ENG-89): the key-pop was the main per-keystroke render
  cost (confirmed on device — typing is noticeably snappier with the preview off). It now appears and
  disappears instantly — the per-press transition + implicit animation are gone — so a keystroke does no
  animation work; `showPop` no longer defers via a `Task`; and the hide is a single step (short linger,
  then removed) instead of a two-phase fade. The connected pop also draws its shape + glyph in one
  `Canvas` node instead of a `Shape` + `overlay(Text)` view tree (fewer nodes to create each keystroke).
  The pop still shows under the finger and lingers a beat.
- Faster keyboard render (ENG-88): keys are no longer each wrapped in an `AnyView`. A device profile
  showed `DynamicViewContainer<AnyView>` (the per-key type erasure from the `buttonContent` replacement
  slot) was a large slice of the SwiftUI update cost on every re-render. Each key is now a concrete
  `KeyboardButton` (it renders a `.none` spacer itself; the globe long-press is a modifier), and the
  per-key customization slot changes from `buttonContent: (item, AnyView) -> AnyView` (full replacement)
  to `keyOverlay: (item) -> AnyView?` (an optional overlay) — so only the rare overlay is type-erased.
  **Breaking**: hosts using `buttonContent` move to `keyOverlay`.
- Faster keyboard render (ENG-87): the key grid is now positioned by a single custom `Layout`
  (`KeyboardKeysLayout`) from the already-solved key widths, instead of nested `VStack{HStack{…}}` with a
  per-key `.frame`. A device profile showed the nested stacks' layout computers (HStack/VStack/ZStack/
  geometry) cost ~170 ms of SwiftUI time — one layout pass removes that. Visually identical to native.
- Profiling infrastructure (ENG-86): the input hot path now emits `os_signpost`s (`Perf`, subsystem
  `SwiftboardKit`/`input`) — `input`, `readContext`, `suggestions`, `layout`, `swipeDecode` — so a device
  trace in Instruments (Time Profiler + SwiftUI + os_signpost) shows exactly where per-keystroke time goes.
  They're near-free when nothing is recording. `PROFILING.md` documents how to record and read a trace.
- Slide-to-type is ~7× faster to decode (ENG-85): the decoder did a full `lowercased()` + `[Character]`
  collapse for every one of the ~40k candidates before checking anything; a cheap first/last-letter anchor
  prefilter (highly selective) now runs first, so the heavy per-word work only touches the handful that
  can match. ~61 ms → ~9 ms per glide over a 40k list, and it already runs off the main thread (ENG-84).
- Typing performance (ENG-84):
  - Every key's depth is now a cheap 1px bottom edge instead of a blurred `.shadow`, which forced an
    offscreen render pass per key every frame (~30 keys) — a real cost while typing/animating.
  - The per-key bubble-spawn geometry reader (a named coordinate space, tracked on every layout pass) is
    attached only for the `.bubble` character-preview style, not the default.
  - Slide-to-type decoding (a full-dictionary scan, tens of ms) now runs **off the main thread** for both
    the live glide preview and the word insertion (latest-glide-wins), so gliding and lift-off no longer
    freeze the keyboard for a frame. The model layer was already fast (~0.03 ms/suggestion) and re-renders
    are isolated, so these render/threading wins are where the latency was.
- Fix: auto-correction no longer changes a word that's valid in another enabled language into a primary-
  language word (the spell checker only knows the primary language). On an EN+CZ keyboard, Czech "jak" is
  now kept instead of being "corrected" to English "oak" — any word known in the merged dictionary is left
  alone (`DictionarySuggestionProvider.isKnown`).
- Added **French, Spanish and German** (ENG-83): new `KeyboardLocale` cases with their layouts
  (AZERTY / QWERTY / QWERTZ), `fr` / `es` / `de` identifiers and native display names. Each ships a
  bundled 40k frequency word list (`fr/es/de_words.txt`, same MIT OpenSubtitles/FrequencyWords source
  as en/cs), so suggestions + swipe work out of the box and multi-language keyboards can mix them. Their
  long-press callouts inherit the full accent set with the language's primary accent first (`a`→`ä` for
  German, `n`→`ñ` for Spanish, `e`→`é` for French); smart punctuation uses « » for French/Spanish and
  „ “ for German.
- The keyboard now supports **multiple languages** (ENG-82). `SwiftboardConfiguration.locale` becomes
  `locales: [KeyboardLocale]` (ordered; first = primary). The primary language drives the key layout,
  callouts, smart punctuation and spell-check; **all** enabled languages are searched for suggestions +
  swipe — their dictionaries are merged (primary first, so a shared word keeps its primary rank and the
  active language's completions come first). New `DictionarySuggestionProvider.builtInWords(for:)`,
  `SwiftboardController.setLocales(_:)` / `locales`, `KeyboardLocale.displayName`, and `loadLocales` /
  `saveLocales` persistence hooks. Demo Settings gains a primary-language picker + "Also suggest …"
  toggles for the extra dictionaries. (Breaking: `locale`/`loadLocale`/`saveLocale` → the plural forms.)
  The space-bar badge now shows every added language, in order, concatenated — e.g. "CZEN" / "CZENDE"
  (`KeyboardLocale.badgeCode`). The demo Settings language section is now an "added languages" list (add
  via menu, swipe to remove, drag to reorder — top = layout) instead of per-language suggest toggles.
- Fix: with `insertOnKeyDown`, picking an accent from a key's long-press callout now **replaces** the
  base character typed on key-down instead of appending the accent after it ("a" then pick "á" → "á",
  not "aá"). The replace is keyed on `insertOnKeyDown` (the base is provably still in the document at that
  point) rather than the `didInsertOnDown` @State, which a mid-hold re-render could reset.
- New parametric feature `spaceAfterPunctuation` (ENG-81, off by default — not native): fills in a missing
  space around punctuation. Any mark `. , ? ! :` typed right after a word inserts the space **immediately**
  (“Hi” + “.” → “Hi. ”); that space then drives native behavior on its own — after `. ? !` the next
  sentence auto-capitalizes (comma/colon stay lowercase) and the keyboard returns to the letters page,
  exactly as if you’d typed the space. After a digit (so decimals “3.14”/times “12:30” are left alone) the
  space is instead added only when a letter actually follows (“42.Next” → “42. Next”). A backspace
  immediately after **undoes** the auto-space and won’t re-add one at that spot until you move on (space /
  return / cursor move) — so abbreviations, URLs and emoticons (“e.g”, “file.txt”, “Hi.jpg”, “:D”) stay
  typable. Pure `SpaceAfterPunctuation` decision logic + a little controller state; demo toggle. When the
  feature is on, mark keys `. , ? ! :` commit on release (like the space bar) so their space + page switch
  don’t fire mid-touch under `insertOnKeyDown` (which re-laid-out the small key and cancelled the space).
- The 40k EN/CZ frequency word lists now ship **inside** SwiftboardKit (`SwiftboardKitAutocomplete`
  resources), loaded lazily per locale via `Bundle.module`. Autocomplete + swipe now work out of the box
  for any host — no need to bundle your own `.txt`. `SwiftboardConfiguration.words` becomes an optional
  override (empty = use the built-in list); the demo app no longer ships its own copies. Attribution
  (OpenSubtitles / Hermit Dave FrequencyWords, MIT) travels with the data in the module.
- Fix: backspace on the emoji panel now repeats (accelerating) while held, so you can delete several
  emoji/characters — like the main keyboard. The emoji preview pop is also snappier and its overlays no
  longer re-render the whole keyboard on each tap (owned by the controller).
- New character-preview style `.bubble` — a "bubble machine" (ENG-80): each key press releases a
  translucent bubble carrying the letter that roams randomly across the keyboard, comes in a varied
  size, bounces off the walls and off other bubbles, and pops after a moment. The bubble is released
  when the character is committed (not on press-down, so holding for a diacritic doesn't pop one early),
  and its lifetime tracks typing speed so you can read what you pressed. It emerges upward from the key
  (like the key-pop), so you register the letter, then bounces (the first wall bounce is randomized, the
  rest physical) off the edges and other bubbles. A freshly-pressed bubble is tinted with the system
  accent (rim + glow) and a touch larger for ~0.35s so you can tell which key it was; the rim otherwise
  shimmers with a soft iridescent (soap-film) gradient. Each bubble is an organic, gently-morphing blob
  (not a circle), squashes where it collides, and bursts into an expanding ring + droplets when it pops.
  Replaces the key-pop, is see-through, and doesn't
  block tapping the keys underneath. Character preview now works on the **emoji panel** too — tapping an
  emoji hides its glyph (it "moved" into the pop) and a preview grows from the cell: a connected key-pop
  (`.native`/`.curvy`/`.square`) — a tall rectangular head on a neck *narrower* than the cell so it reads
  as emerging from within, no stroke/shadow/tint, shifted inward at the screen edge to stay on-screen — or
  the floating bubble (`.floating`), or a bubble (`.bubble`). Appears instantly, very brief hold. (The emoji grid is host-rendered and not made of `KeyboardButton`s, so
  `EmojiKeyboard`/`EmojiPanelView` gained an `onSelectAt` reporting the tap location, which drives a
  shared `BubbleField` / `EmojiPreviewModel`.) Pure `BubblePhysics` +
  `BubbleField`/`BubbleFieldView` (a paused-when-idle `TimelineView` + `Canvas`). Tap-to-pop is a later
  phase.
- `KeyboardBehavior.swiftboardDefault` — the kit's recommended default, used by `SwiftboardConfiguration`
  when a host doesn't pass a behavior: native but with a uniform key colour, no globe/mic keys, and
  key-down input. `.standard` stays pure native. So an integrator gets these defaults without configuring.
- New `SwiftboardKitApp` module — a batteries-included assembly so a host ships a keyboard in ~10 lines
  (ENG-78). It adds `SwiftboardController` (the full keyboard brain + text editing), `SwiftboardRootView`
  (the whole SwiftUI keyboard), `SwiftboardInputViewController` (a base `UIInputViewController` to
  subclass), `SwiftboardConfiguration` (inject behavior, word lists, persistence, dictation),
  `SwiftboardTextDocument` (+ `ProxyTextDocument`/`StringTextDocument`, so one text-editing path serves
  both the extension proxy and an in-app buffer), and `SwitchKeyboardRow`. A real extension is now just
  `class KeyboardViewController: SwiftboardInputViewController { override func makeConfiguration() … }`.
  The demo's `DemoKeyboardModel`/`DemoKeyboardRootView` were deleted and its controller shrank from 256
  to 12 lines; no host logic changed.
- Input-type adaptation: the keyboard now adapts to the edited field, like native (ENG-77). Email fields
  get "@" and "." around the space (and no auto-capitalization); URL and browser search/address bars
  (`.URL`/`.webSearch` — Safari/Chrome omnibox) get "." "/" and a locale-aware ".com" domain key
  (long-press for .cz/.net/.org/…); number/phone/decimal fields get a digit pad; and the
  return key shows its purpose (Go/Search/Send/Done/Next/…) in the accent colour. New `KeyboardInputType`,
  `ReturnKeyType`, `DomainKey`, `KeyboardContext.inputType`/`returnKeyType`, and UIKit mappings
  (`KeyboardInputType(_:)`/`ReturnKeyType(_:)` from `UIKeyboardType`/`UIReturnKeyType`). Gated by
  `KeyboardBehavior.adaptToInputType` (default on). Note: iOS never presents a third-party keyboard for
  number/phone/decimal pad fields, so those pads apply only to an in-app/embedded host.
- Fix: auto-correction now also fires on terminating punctuation and return, not just space (ENG-76).
  Typing "delas." now yields "děláš." (and likewise for "," "!" "?" ";" ":" and Enter), matching native.
  Both hosts share `DemoKeyboardModel.autocorrectTerminators`.
- Fix: Czech + diacritics — suggestions, space-bar autocorrect, and cleaner data (ENG-75).
  - Completion matched the prefix case-insensitively but not diacritic-insensitively, so typing base
    letters without háčky/čárky ("cas", "mas", "vecer") matched nothing while English worked. Bucketing
    and prefix matching are now diacritic-folded ("cas" → "čas", "mas" → "máš", "vecer" → "večeře").
  - Space-bar autocorrect only used UITextChecker, which won't turn "delas" into "děláš", so the bar
    offered the accented word but space kept the bare one. `DictionarySuggestionProvider` now exposes
    `diacriticRestoration(of:)` (the most-frequent known word that is the typed word with diacritics
    added — never changes a letter), and the host tries it before the spell checker.
  - The Czech list carried diacritic-free "shadow" spellings from subtitles ("cas", "kdyz", "dekuji")
    that read as known words and blocked restoration. Those were dropped when not valid Czech words
    (checked against the Czech Hunspell dictionary at build time); real homographs stay ("byt", "rada",
    "lze", "zabit").
- Better predictions: cleaner word-frequency data + case-aware completion (ENG-72). The bundled 50k
  lists were replaced with cleaned 40k lists from OpenSubtitles (Hermit Dave FrequencyWords) — real
  frequencies all the way down (the old lists degraded into an alphabetically-ordered low-frequency
  tail) and no proper-noun/junk noise (the old English list was ~31% capitalised proper nouns).
  `DictionarySuggestionProvider` now re-cases completions to match the typed prefix ("Ho" → "Home",
  "HO" → "HOME"). Next-word (n-gram) context and personalisation are tracked as follow-ups.
- Snappier typing: optional key-down input (ENG-71): a new `KeyboardBehavior.insertOnKeyDown` inserts a
  character key the instant it's pressed instead of on release, so the letter appears immediately (engine
  default `false` = native key-up; the demo defaults it on, with a Settings toggle). A long-press for
  accents replaces the key-down character, a glide undoes it, and dragging off the key cancels it;
  `didInsertOnDown` stops it being typed again on release. The per-key haptic tap is also dialled down
  to a lighter intensity (`keyImpactIntensity`), so fast typing with haptics on doesn't feel heavy.
- Character preview: the key's letter is removed first, then the pop appears (ENG-70): on touch-down
  `showPop()` now hides the key's character immediately (this frame) and shows the pop on the next
  runloop tick, so the touched key goes blank *before* the preview draws — never both at once. The
  post-release linger is keyed on the synchronous `keyMerged` (not the deferred `popVisible`), so a very
  fast tap still holds the pop.
- Character preview shape is now an angular taper with softly rounded corners; the square style is
  angular again (ENG-69): `KeyPopShape` was rewritten from curves to a **straight taper** — each side is
  vertical (head) · small fillet · **straight diagonal** · small fillet · vertical (key). The neck now
  roots deeper inside the key (`totalH = keyPopOverflow + 0.7·keyRowHeight`), so the widening emerges from
  within the key (its key-width bottom hidden inside it). The profile sets the corner rounding — `rTop`
  rounds the head's top corners, `fillet` the bottom corners (the shoulders where the head meets the
  neck), leaving the straight diagonal middle straight: native = rounded top + softly rounded shoulders
  (`fillet 4`); **square (`.angular`) = sharp** (it had lost its angular look when the shape went to
  curves); curvy = rounder. The top stays fixed, so headroom is unchanged.
- Character preview sits lower (closer to the key) and the keyboard is shorter (ENG-68): two parts —
  (a) `NativeMetrics.keyPopOverflow` dropped from `1.56·keyRowHeight` to `1.2`, so the pop grows a shorter
  neck right above the pressed key (min ≈ its head height, `1.05·keyRowHeight`); (b) the reserved headroom
  (`DemoKeyboardModel.popTopInset`) now also subtracts the key area's own `verticalPadding` (the pop draws
  up into it), which ENG-59 forgot — it was over-reserving ~8pt. The keyboard's extra height is now ~4pt
  (was ~26pt in ENG-59, 10pt after part (a)), with the pop unchanged (2pt clearance, no clipping). The
  seamless flow into the key (ENG-65) is preserved.
- Spacebar cursor-move is now 2D (ENG-67): dragging the spacebar up/down moves the caret by whole
  **lines**, not just left/right by characters — like the native trackpad. Since the proxy can only move
  by characters, the host maps a row delta to a character offset from the document context via the new
  pure `SpacebarDragLogic.characterOffset(forRows:before:after:)` (keeps the column, clamps to the ends of
  the context). New `.moveCursorRow(Int)` action + `SpacebarDragLogic.row(...)`. (Works on explicit
  newlines; a third-party keyboard can't see soft-wrapped visual lines.)
- Spacebar cursor-move clears the keys and freezes state (ENG-66): while you press-and-hold the spacebar
  to reposition the cursor, every key now blanks its label (no letters/symbols, and the EN/CZ badge
  hides) and the predictions and keyboard case stop changing — exactly like the native keyboard. A new
  observable `CursorMoveState` (watched by every key) is flipped by the space key during the drag; the
  new `SwiftboardKeyboardView(onCursorMove:)` reports the drag start/end so the host can freeze
  `updateSuggestions` / `updateCase` (gated on the model's `isMovingCursor`) and re-sync once on release.
  The in-app preview now edits at a real **caret** (index-based insert/delete/move, drawn as a coloured
  bar) so the spacebar cursor move is visible there too; the extension moves the caret via the system's
  `adjustTextPosition`. The spacebar-drag state is held off `@State` (`SpaceDragState`) and the space key
  is excluded from the blank, so it doesn't re-render mid-drag — a re-render was interrupting its gesture
  in the extension's `UIHostingController`, so the cursor only moved one character (looked stuck).
- Softer, symmetric character-preview (key-pop) shape (ENG-65): the native connected pop looked angular,
  its bottom edges sharp and close to the key, and the letter crowded the top. The head is now rounded
  with a larger radius and the **bottom corners are generously rounded** (`KeyPopShape.bottomCornerRadius
  = keyWidth*0.44`); the large flare arc and the large bottom arc are opposing curves that sweep into one
  graceful S on each side (a slender, elongated neck). The transition from the smaller key to the larger
  pop is a **single continuous curve**: the flare reaches key width exactly at the key's top edge (`totalH
  = keyPopOverflow + rBot`) and the rounded bottom curves in *below* it, hidden inside the same-coloured
  merged key — so the neck doesn't bulge or pinch at the key and the **bottom edge is absorbed by the
  curve**. The **top edge stays fixed** at `keyPopOverflow`, so the ENG-59 headroom / keyboard height is
  unchanged.

### Performance
- Character preview no longer causes dropped letters when typing fast (ENG-64): with the preview on,
  every keystroke additionally built *and animated* the pop on the key that also carries the touch
  gesture; that extra per-press main-thread work occasionally caused the next fast tap's gesture to be
  missed (a dropped letter). The pop now appears **instantly** (no insertion animation) so a keystroke
  commits no animation work; it still holds briefly and fades out on release (off the touch handler).
- Haptics and key sound no longer delay the next keystroke (ENG-63): key feedback ran synchronously on
  the main thread. Now (A) the sound plays on a background queue — `AudioServicesPlaySystemSound` is a
  thread-safe C API with real invocation cost, so keeping it off the main thread stops it stalling the
  next tap; (B) the haptic generators are `prepare()`d on key-down (and re-prepared after firing) so the
  Taptic Engine is warm and the impact isn't delayed — the `impactOccurred()` call itself is cheap and
  non-blocking, so it stays on the main thread (UIFeedbackGenerator is main-thread only); (C) the
  generators are `static` (one shared handle) so their prepared state survives a view rebuild. Adds an
  optional `prepare()` to `KeyboardFeedback` (default no-op).

### Changed
- Numeric/symbolic page's third row now lines up with the alphabetic shift/backspace (ENG-62): the `#+=`
  (and `123`) page toggle and the backspace on the number/symbol pages were `.available`, so their width
  depended on how many keys sat between them (5 punctuation) and didn't match the alphabetic shift/backspace
  (7 letters). They now take the shift's exact width (`shiftKeyWidth`, shared with the bottom-row toggle),
  and the punctuation keys between them are `.available` so they share the leftover space evenly — a touch
  wider than a standard key, like native.
- Character preview (key-pop) timing now matches native (ENG-61): the pop's lifetime is decoupled from
  the press, so it snaps up quickly on press (0.05s), then — even on a quick tap — holds fully visible
  briefly after release (0.06s) before fading/retracting toward the key (0.13s). The pressed key stays
  merged with the pop (its letter hidden, pop colour) through the whole fade and its letter/colour
  restore only once the pop has *fully* disappeared — never while the preview is still on screen.
  Previously the pop was tied to `isPressed`, so a quick tap barely showed the finished preview.

### Fixed
- Character preview (key-pop) for the edge keys was clipped by the screen edge (ENG-60): the pop's head
  is wider than its key, so for the leftmost/rightmost keys (q/p, 1/0, or whatever the layout puts
  there) it ran off the screen when centred. The head now shifts inward (clamped to the edge) while the
  neck stays anchored to the key and skews to reconnect — like the native keyboard. (`ConnectedKeyPop.
  headOffsetX` + `KeyPopShape.keyCenterOffsetX`.)
- Character preview (key-pop) for the top row was clipped in the keyboard extension (ENG-59): the pop
  extends above the keyboard's top edge, and a keyboard **extension cannot draw outside its frame** (the
  system clips it — unlike the native/system keyboard, which is privileged). The keyboard now reserves
  the pop's overflow as headroom *inside* a slightly taller frame: `SwiftboardKeyboardView` gains a
  `topInset` (filled by the translucent backdrop, so it reads as a marginally taller keyboard, not a
  gap), sized from the new `NativeMetrics.keyPopOverflow` minus the toolbar height, and the extension's
  height includes it. The top-row pop now shows in full. (The in-app preview already rendered it
  un-clipped, since a single app view hierarchy doesn't clip the overflow.)

### Added
- Emoji suggestions (ENG-58): the word being typed is mapped to an emoji (e.g. "Hi"/"ahoj" → 👋)
  and offered in the suggestion bar as a highlighted slot; picking it replaces the word. New
  `EmojiSuggestion` (bilingual EN/CZ, case- and diacritic-insensitive) and a `emojiSuggestions`
  behavior flag (default on) + Settings toggle. The bar shows whenever predictions, math results
  **or** emoji suggestions are on.
- Math results now offer three bar entries like predictive text (ENG-57): the expression ("1+1"),
  the highlighted `expression=result` form ("1+1=2"), and the bare result ("2"). Picking any of them
  replaces the trailing expression. New `MathResult.expressionAndResult(...)`, `.suggestionVariants(_:)`
  and `.variantTexts(...)` build the three slots from one source of truth.

### Fixed
- Math results didn't show with predictive text off (ENG-56): the suggestion bar (and its reserved
  height in the extension) was gated only on `predictiveText`, so with predictions off but
  `showMathResults` on, the computed result had nowhere to appear. The bar now shows whenever
  `predictiveText || showMathResults`. Verified: typing "2+2" with predictive text off shows "4".
- Key sound never played even with `keySound` on (ENG-55): the engine used `UIDevice.playInputClick()`,
  which is a silent no-op in a keyboard extension unless the input view adopts `UIInputViewAudioFeedback`
  and is granted Full Access — and even then it's unreliable. It now plays the system's own keyboard
  sound IDs directly via `AudioServicesPlaySystemSound` (1104 tap / 1155 delete / 1156 modifier), which
  works from an extension **without Full Access and with no host setup**. Respects the silent switch;
  note the Simulator doesn't play these clicks (test on device).
- Keyboard no longer resizes when tapping the emoji search field (ENG-54): while browsing emoji the
  header showed an empty row (reserved for search results), and tapping search grew the whole keyboard
  by that row. The `EmojiSearchHeader` now renders just the field while browsing (no empty row), and the
  emoji panel is taller by exactly the results-row height (`resultsRowHeight`), so the total keyboard
  height is identical whether browsing or searching — no resize. Verified the keyboard's top edge stays
  put across both states (iPhone 17 Pro).
- Emoji too small / too many rows on large phones (ENG-53): the emoji grid sized its cells from the
  collection view's height inside `sizeForItemAt`, which could lag on some devices (e.g. iPhone Pro
  Max) — giving small emoji in 7 rows instead of 4. The cell size is now computed in `layoutSubviews`
  from the panel's own frame (reliable) and set on the layout, so exactly `rowCount` (4) rows fill the
  panel with large emoji on every device. Verified on iPhone 17 Pro Max.

### Changed
- Bottom row lines up with the letter grid (ENG-52): the space-row keys now align to the letters
  above like native. The page-toggle (123/ABC) matches the shift key's width; the first special key
  (emoji, else globe/mic) right-aligns with "z"; globe and mic are one key column each (sitting under
  "x", "c", …); the return key's left edge meets "m" and it runs to the edge; and the space bar fills
  what's left — so it spans exactly **3 key columns with globe+mic, 4 with one, 5 with neither**. New
  `KeyboardLayoutItemWidth.grid(inputs:gaps:points:)` expresses widths on the letter grid; the special
  keys are ordered `123 · emoji · globe · mic · space · return`.

### Fixed
- Key height (ENG-51): keys were a touch shorter than native. Re-measured against the native
  keyboard (both at the same scale: our key ~128px vs native ~133px) and bumped `NativeMetrics.
  iPhonePortrait.keyRowHeight` 43 → 45pt so key height matches native. The keyboard sizes itself from
  this metric, so the overall height follows.

### Changed
- Key-pop & callout colour = key colour, one source of truth (ENG-50): the character key-pop and the
  long-press callout (diacritics) bar now derive their fill from the key's fill (`KeyboardButton.
  keyFillColor`) instead of a separate colour, so they always match the keys (incl. `uniformKeyColor`
  and glass). Being floating UI they render it **opaque** — on glass the key's translucent fill is
  composited over the keyboard backdrop (new `RGBA.composited`), giving the key's real surface colour.
  Measured 0.27 == 0.27 vs the keys. The colour bridge (`RGBA.color`, `KeyboardColor.color(for:)`,
  `KeyboardAppearance(_:)`) is now public; the demo's in-app preview gains a backdrop standing in for
  the extension's system glass so translucent keys show their true surface colour. While the key-pop
  shows, the pressed key takes the **same colour as the pop** and drops its shadow, so key + pop merge
  into one seamless connected shape (no junction seam/border). The letter hides on the key instantly
  and the pop grows out over a short beat (~0.11 s) then vanishes instantly on release — so the pop is
  animated enough to notice, but the letter never shows on both the key and the pop at once.
- Wider gap by shift/backspace (ENG-50): shift and backspace (and the numeric page's `#+=`/`123`
  toggle, which sits in the shift position) are a touch narrower so the gap to the adjacent letters is
  ~2.5× the normal inter-key spacing, like the native keyboard (an invisible `.points` spacer).

### Added
- Demo: language selection moved to Settings + input-language indicator on the space bar (ENG-49):
  the English/Čeština picker moved from the main screen into Settings; the chosen language persists
  (App Group) and is shown as EN/CZ in the **space bar's bottom-right corner** (positioned via a
  GeometryReader), like native. Switching to Czech also flips the layout to QWERTZ.
- Hide the "space" label (ENG-49): new `KeyboardBehavior.showSpaceLabel` (default on = native "space")
  leaves the space bar blank when off, so a host can show just a corner indicator. Settings toggle; the
  demo defaults it off to show only the EN/CZ language badge.
- Uniform key colour (ENG-48): new `KeyboardBehavior.uniformKeyColor` (default off = native darker
  control keys) fills every key — shift, backspace, return, globe, mic, 123, … — with the standard
  letter-key colour. Settings toggle "Uniform key color". Set the flag on for the flat look.

### Changed
- Coalesce the suggestion recompute off the critical path (ENG-47): the demo host now updates the
  keyboard case immediately (it drives the shift state + visible case) but schedules the suggestion
  recompute coalesced onto the next runloop tick, so a burst of fast keystrokes doesn't run the
  dictionary lookup between touches (only the latest wins). Small follow-up to ENG-44/45/46.

### Fixed
- Duplicate per-keystroke work (ENG-46): the demo host recomputed suggestions + case twice per
  keystroke — once after its own edit and again from the system's `textDidChange` for the same edit.
  `syncAfterInput` now caches the last document context and returns early when unchanged, halving the
  per-keystroke logic. Documented as a host tip in INTEGRATION.md §18. (Measured touch-up→insert
  latency is ~1 frame; the SwiftUI gesture path is near its floor, so a custom UIKit touch layer
  wasn't warranted.)
- Typing latency, take 2 — the key grid re-rendered on every keystroke (ENG-45): a host updating its
  suggestion bar per keystroke re-evaluated the whole `SwiftboardKeyboardView`, and because
  `KeyboardButton` isn't `Equatable` all ~33 keys re-rendered (~10 ms in Debug). Fixes:
  (1) the key area is now an **`Equatable` `KeyGrid`** gated on `layout`/`appearance`/`behavior`, so
  it's skipped when a plain letter doesn't change them — measured 0 grid re-renders while typing 8
  mid-word letters; (2) each key no longer carries a `GeometryReader` (its width is passed in; its
  global x is read via `.onGeometryChange`), cutting a grid render from ~10 ms to ~6 ms. The demo
  also moves suggestions into their own `ObservableObject` so a keystroke re-renders only the bar
  (documented as the recommended host pattern in INTEGRATION.md §18).
- Typing latency & dropped keystrokes (ENG-44): `DictionarySuggestionProvider.suggestions(for:)`
  ran on every keystroke and, on a miss, scanned the whole ~50k word list calling `.lowercased()`
  twice per candidate (up to ~100k String allocations per keystroke) — a main-thread stall that
  delayed the character and dropped fast taps. Now the words are indexed by lowercased first
  character (built once) and matched with an allocation-free anchored case-insensitive compare, so
  a keystroke touches one bucket. ~3.6× faster in Debug worst case (3.5 ms → 0.98 ms/keystroke);
  suggestion results are unchanged.

### Added
- Integration guide (ENG-43): a detailed `INTEGRATION.md` for SDK consumers — mental model, quick
  start, `KeyboardAction` routing, the full `KeyboardBehavior` reference, Liquid Glass transparency,
  styling (`keyFill`/`keyFillOnGlass`), autocomplete/emoji/slide-to-type, and App Group setup. README
  trimmed to point at it. Made the `StandardLayoutResolver.layout(for:)` context convenience `public`.
- Translucent keyboard background (ENG-42): the keyboard is no longer an opaque rectangle — a keyboard
  extension already renders over the system's translucent keyboard backdrop (iOS 26 "liquid glass"), so
  the engine's backdrop is now fully **clear** (any material/colour fill of our own just adds a grey veil
  that reads as an opaque rectangle over it). New `KeyboardBehavior.translucentBackground` (default on)
  with a Settings toggle; the extension sets its view background to clear so the system backdrop and the
  host app show through. On the glass the keys are translucent too (`keyFillOnGlass`, alphas measured
  from the native iOS 26 dark keyboard — letters ≈ 0.33, control keys ≈ backdrop), so they blend with
  the app instead of reading as darker opaque blocks.
- Callout includes the base letter (ENG-41): like native, the long-press callout now includes the
  plain (un-accented) character as the pre-selected entry directly above the key — releasing without a
  drag types it, and you slide sideways to pick an accent. The bar grows right for keys on the left
  half and left for keys on the right half so the base stays over the key and the accents run toward
  the roomier side; the callout cell width dropped to 30.
- Full accent callouts (ENG-40): the long-press callout now offers the complete set of accents (native
  iOS + Czech) for every letter in both locales — e.g. "a" → à á â ä ǎ æ ã å ā ă ą, and diacritics added
  for d/g/h/j/k/l/n/r/s/t/u/w/y/z. The callout bar's cells are narrower so a full set (~11) fits, and it
  now clamps to the screen edges (offset from the key's global x) so an edge key's callout stays visible.
- Selectable key layouts (ENG-39): QWERTY, AZERTY, QWERTZ, Dvorak and Colemak. `KeyboardLayoutType`
  gains `.dvorak` / `.colemak` (authentic letter rows in `CharacterRows`), and a new
  `KeyboardBehavior.layoutTypeOverride` (nil = the locale's default) selects the arrangement — the
  resolver already framed the rows generically (shift/backspace) and callouts are keyed by character,
  so any layout works. Demo Settings adds a layout picker. For Dvorak the backspace sits on the top
  row (right of "l"). All letter keys are the same width and the rows are staggered by half a key (like
  a classic keyboard / the native iOS Dvorak) so each key nests in the gap of the row above/below: the
  1.5-key shift shifts the bottom letters, the home row gets a trailing spacer so "s" sits off the right
  edge, and the top row is right-aligned via a leading spacer with a ~2-key-wide backspace. `.none`
  layout items render as transparent spacers.
- Choosable key-pop style (ENG-38): the character preview now has four styles via the new
  `CharacterPreviewStyle` enum + `KeyboardBehavior.characterPreviewStyle` (default `.native`):
  `.native` — a larger key that flares smoothly out of the pressed key with tangent-vertical curves at
  both the key and the head, so there are no edges (grows out of the key "like a river widening");
  `.curvy` — a more pronounced neck; `.square` — angular (no rounding); `.floating` — a detached bubble
  above the key. New `KeyPopShape` (smooth/curvy/angular profiles) + `ConnectedKeyPop`; `KeyboardButton`
  measures its width so the neck meets the key. While the pop is showing, the key itself goes blank (the
  character is only in the preview), like native. Demo Settings adds a style picker. The extension no
  longer clips the preview at its top edge (clipsToBounds = false), so a top-row key's pop shows in full.
- Auto-return to letters after a space (ENG-37): on the numeric (123) or symbolic (#+=) page, typing
  a space or newline flips the keyboard back to the letters page — native behavior, since after
  whitespace you usually start a new word. New pure `LayoutAutoSwitch.nextType(after:on:enabled:)`
  (triggers = space/newline; digits/symbols keep the page); the shared action handler calls it after
  a character/space/return. Gated by `KeyboardBehavior.returnToLettersAfterSpace` (default on) with a
  Settings toggle.
- Space-before-punctuation fixup (ENG-36): typing a sentence mark after a stray space that follows a
  word moves the space to after the mark — " ." → ". ", " ," → ", ", and likewise ! ? ; :. New pure
  `PunctuationSpacing.fixup` (fires only after a letter/number, not on chained spaces or after
  punctuation); the host deletes the space and inserts the mark + a space. Gated by
  `KeyboardBehavior.punctuationSpacing` (default on), with a Settings toggle. Wired into both hosts
  ahead of smart punctuation. Fully unit-tested.
- Slide-to-type shows live suggestions while gliding (ENG-35): the suggestion bar now updates as the
  finger moves, not only on lift-off. New `KeyboardAction.slideChanged(keys:)` is emitted from the
  swipe gesture on each newly-crossed key; the host routes it to a preview that fills the bar without
  inserting. On lift-off `slideTyped` still inserts the best word and keeps the alternatives (ENG-34).
  (Mid-glide suggestions are sparse with the strict first-pass decoder — a geometric decoder for
  continuous, high-quality live suggestions is tracked as ENG-31.)
- Slide-to-type offers word choices (ENG-34): a glide now inserts the best word and shows the
  alternatives in the suggestion bar (best highlighted), like predictive text — tapping an
  alternative replaces the inserted word. New `SwipeDecoder.decodeCandidates(…limit:)` returns the
  top-N matches (longest → frequency, de-duplicated); `decode` is now its first element. The demo
  model fills the bar with the candidates on swipe and keeps them until you type past the word; a
  picked alternative replaces the inserted word (+ trailing space).
- Large frequency dictionary in the demo (ENG-33): predictions and slide-to-type now run off a
  50,000-word-per-language list (English + Czech) instead of the tiny built-in seed. Derived from the
  Leipzig Corpora Collection (eng_news_2024_1M / ces_news_2024_1M, CC-BY 4.0): alphabetic tokens only,
  case variants merged (proper nouns keep their capitalisation), top 50k by frequency. Shipped as
  `DemoApp/Resources/{en,cs}_words.txt` bundled into both the app and the keyboard extension;
  `DemoKeyboardModel` loads them from the bundle (falling back to the built-in list). Attribution in
  the demo Settings footer and `Resources/CREDITS.txt`. The engine is unchanged — the host supplies
  the dictionary via `DictionarySuggestionProvider(words:)`.

### Fixed
- Predictive text missing common words (ENG-32): the built-in `WordFrequencies` list had only ~78
  English words, so everyday words (are, how, where, thinking, …) were never suggested. Expanded it
  to a few hundred frequency-ordered common words (English + Czech) including inflections
  (think/thinks/thinking/thought, are/is/was/were, where/when/what/who/how, …).
  `DictionarySuggestionProvider` now de-duplicates its word list case-insensitively (keeping the
  first, most-frequent occurrence) so a word listed in several sections isn't offered twice.
- Slide-to-type "wasn't detected" (ENG-30): two bugs. (1) The main one — a glide triggered the
  per-key **long-press callout** (accent bar), and the drag then selected accents instead of the
  swipe, so the user saw only the key-pop/accents and no word. A letter key now **yields to the
  keyboard-level swipe** as soon as the finger travels (before the callouts open) or the swipe
  engages: it cancels the callout, drops the key-pop, and doesn't type on release; a hold-in-place
  still opens callouts. (2) `SwipeDecoder` was too strict (exact first/last key + frequency-only),
  so real glides that over-/under-shoot decoded to nil — it now tolerates start/end overshoot (the
  anchor letter may be within the first/last two crossed keys) and prefers the longest matching word,
  frequency as the tie-break. Verified: an injected glide fires slide-to-type with no accent garbage,
  and a "hello" key path decodes to "hello".

### Added
- One-handed picker on globe long-press (ENG-29): long-pressing the globe key opens a menu with a
  one-handed left/off/right picker above a "Switch keyboard" row (the system keyboard picker), like
  the native keyboard. Gated by `KeyboardBehavior.oneHandedGlobeMenu` (default on). The engine draws
  the menu and the one-handed picker (`onSelectOneHanded`); the system picker can't be opened purely
  from SwiftUI, so the host supplies the switch-keyboard row via the new `nextKeyboardMenuRow` slot
  (the demo wires a `UIButton` to `handleInputModeList`). Our one-handed row can't be added to the
  system menu itself (private to Apple's keyboards), hence the custom menu.
- Parametric one-handed shift amount (ENG-28): `KeyboardBehavior.oneHandedScale` controls how far
  the keyboard shrinks + shifts in one-handed mode — a fraction of the full width, default
  `OneHandedLayout.defaultScale` (0.75 = a 25% shift). The key area passes it to
  `OneHandedLayout.metrics`; the demo Settings adds a slider (shown only when one-handed is on).
  Globe + mic live in the keyboard's own bottom row, so they shift with it as one block (native).
  (Note: iOS's own Settings → Keyboard one-handed choice isn't readable from a keyboard extension —
  no public API — so this stays app-controlled.)

### Fixed
- One-handed orientation (ENG-26): `.left` again puts the keys on the left (arrow on the right),
  `.right` on the right — the ENG-24 swap was backwards. The expand arrow sets `oneHandedMode = .off`
  (persisted); the app reloads settings on foreground so extension changes show up.

### Added
- Shared settings via App Group (ENG-25): `KeyboardBehavior`/`OneHandedMode` are now `Codable`;
  the demo persists the behavior to a shared App Group so settings chosen in the app apply to the
  keyboard extension in other apps. The extension reloads on appear.
- Globe key toggle (ENG-23): `KeyboardBehavior.nextKeyboardKey` (default on) shows/hides the
  globe / next-keyboard key.
- One-handed expand button (ENG-24): a chevron in the gap returns the keyboard to full width
  (`onExpandOneHanded`); `.left`/`.right` swapped so `.right` = keys left / arrow right.

### Fixed
- Single shift acting like caps lock (ENG-22): a single shift now drops back to lowercase after
  one character (the extension no longer relies on textDidChange, which can miss self-inserted
  text). The auto-drop is parametric — new `KeyboardBehavior.autoShiftToLowercase` (default true;
  off = sticky shift). Caps lock (double-tap) is unchanged.

### Added
- Interactive emoji search (ENG-20): `EmojiSearchHeader` (a search field + results row for the
  emoji panel's toolbar) plus the demo's search mode — typing feeds the query, results update live,
  the return key closes search. Gated by `emojiSearch`. Completes the emoji feature.
- Emoji search (ENG-19): a curated 822-entry keyword map (EN+CZ) + `EmojiSearch.search` —
  diacritic- and case-insensitive full-text emoji search ("stastny" finds "šťastný"). Reusable
  API; the interactive search UI is a follow-up.
- Emoji panel (ENG-18): a new `SwiftboardKitEmoji` module — catalog, skin-tone variants, a
  host-injectable recents store, and a polished UIKit emoji panel (`EmojiKeyboard` for the
  `emojiKeyboard` slot). A smiley key (gated by `emojiKey`) switches to the panel; hosts may
  supply their own emoji view in the slot instead. Ported from Rewordo, made host-agnostic.
- Dictation (ENG-17): a mic key in the bottom row (added only when enabled) + a `.dictation` action
  and host `onDictate` hook; the host supplies speech-to-text. Gated by `dictation`.
- Slide-to-type (ENG-16): glide across the keys to type a word — `SwipeDecoder` + `SwipeKeys`
  (pure, tested) with a keyboard-level swipe gesture that coordinates with per-key taps; the host
  decodes with its dictionary. New `.slideTyped` action. Gated by `slideToType`. First-pass decoder.
- One-handed keyboard + math results (ENG-15): `oneHandedMode` (.off/.left/.right) shrinks and
  shifts the keys via `OneHandedLayout`; `showMathResults` offers the result of a trailing
  arithmetic expression (`MathEvaluator` + `MathResult`) in the suggestion bar.
- Auto-correction + spell check (ENG-14): `SpellChecking` protocol + `AutoCorrection.correction`
  (pure), with an iOS `SystemSpellChecker` (UITextChecker). On space the completed word is
  corrected. Gated by `autoCorrection` / `checkSpelling`. (Underlining misspellings in the host is
  owned by iOS and not possible from an extension.)
- Delete by word (ENG-13): accelerating backspace hold that escalates to word deletion
  (`DeleteByWord.wordDeletionLength` + a new `.deleteWord` action), gated by `deleteByWord`.
- Smart punctuation (ENG-12): `SmartPunctuation.transform` — straight quotes → locale-aware curly
  quotes (EN “ ” / ‘ ’, CS „ “ / ‚ ’), apostrophe → ’, and "--" → em dash. Gated by `smartPunctuation`.
- ". " shortcut (ENG-11): `PeriodShortcut.shouldExpand` — double-space after a word becomes
  ". " (period + space); gated by `periodShortcut`. Doesn't chain or fire after punctuation.
- Auto-capitalization (ENG-10): `AutoCapitalization.shouldCapitalizeNext` (start of field, after
  a sentence terminator + space, or after a newline), gated by `autoCapitalization`. Fully tested.
- `KeyboardBehavior` config (ENG-9): a single parametric surface mirroring the native iOS
  keyboard settings (character preview, haptics, sound, auto-caps, caps lock, smart punctuation,
  predictive/autocorrect, one-handed, slide-to-type, …) so SDK consumers can toggle each behavior.
  Defaults reproduce native. Wired the already-implemented features (key-pop, feedback, caps lock,
  predictive bar) to it. See NATIVE-SETTINGS.md.

### Changed
- Native parity pass (ENG-8): measured the native iOS 26 keyboard (iPhone 17 Pro, light+dark)
  by pixel-sampling and locked in `NativeMetrics`/`NativeKeyboardStyle` — key height 43pt, key
  gap 6pt, row gap 11pt; measured colors (light bg 0.886, light keys white incl. control; dark
  bg 0.161, standard 0.420, control 0.271); added the native key shadow; neutral suggestion
  highlight. Side-by-side indistinguishable in light and dark.

### Added
- Autocomplete (ENG-6): `SuggestionProvider` + a `DictionarySuggestionProvider` that completes
  the current word from a frequency word list (built-in EN/CS), plus a native 3-slot
  `SuggestionBar` for the toolbar slot. `nil` provider = no suggestions. Logic fully tested.
- Spacebar-drag + feedback (ENG-5): slide across the spacebar to move the caret (new
  `KeyboardAction.moveCursor`), plus injectable haptic/audio feedback (`KeyboardFeedback` /
  `SystemKeyboardFeedback`, gated by `FeedbackSettings`). Drag/feedback logic fully tested.
- Shift/caps + callouts + key-pop (ENG-4): native shift state machine (single toggle,
  double-tap caps lock, auto-drop after a character) and callout selection math, both fully
  tested; iOS key-pop bubble on press, long-press callout bar with slide-to-select, and an
  active-state shift key (shift.fill / capslock.fill).
- DemoApp (ENG-7, dev tooling): xcodegen-based project with an in-app live keyboard preview
  and a real keyboard extension, both driven by the shared engine. Runs on the simulator with
  no App Group/entitlements; render confirmed side-by-side close to native in light and dark.
- Rendering layer (ENG-3): `SwiftboardKeyboardView` (SwiftUI) with `toolbar` / `buttonContent`
  / `emojiKeyboard` slots, a native-alignment key-width solver, and an injectable
  `KeyboardStyleProvider` whose default (`NativeKeyboardStyle`) reproduces the native iOS look
  (host overrides only what it wants). Geometry/colors are provisional pending ENG-8 measurement.
  Cross-platform logic is 100% covered; SwiftUI views verified by an iOS simulator build.
- Keyboard model & layout engine (ENG-2): `KeyboardContext` state, action/type/case/locale
  value types, and a data-driven `StandardLayoutResolver` producing native-structured layouts
  for EN (QWERTY) and CS (QWERTZ) — letter/number/symbol pages, shift, and per-locale
  long-press callouts. Adding a language is a data change. 100% test coverage.
- Scaffold repozitáře: SPM balíček (moduly `SwiftboardKit`, `SwiftboardKitLayouts`,
  `SwiftboardKitAutocomplete`), dokumentace (PLAN/DESIGN/README/TODO/FEATURES),
  `LicenseGate` no-op seam.
