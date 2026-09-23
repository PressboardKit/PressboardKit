# PressboardKit — Integration Guide

How to embed PressboardKit in your own iOS app's keyboard extension. This is the
detailed, hands-on companion to [`README.md`](README.md) (overview) and
[`FEATURES.md`](FEATURES.md) (feature list). Everything here reflects the **actual public
API**; the code in [`DemoApp/`](DemoApp) is the working reference for all of it.

> **Design promise:** the keyboard is meant to be **indistinguishable from the native iOS
> keyboard**, and **every behavior is a parameter** (`KeyboardBehavior`) whose default
> reproduces native. You opt out of a behavior; you never have to opt in to get native.

- [1. What you get](#1-what-you-get)
- [2. Requirements & installation](#2-requirements--installation)
- [3. Mental model](#3-mental-model)
- [4. Quick start — a working keyboard in ~120 lines](#4-quick-start--a-working-keyboard-in-120-lines)
- [5. The render view: `PressboardKeyboardView`](#5-the-render-view-pressboardkeyboardview)
- [6. State: `KeyboardContext` + `StandardLayoutResolver`](#6-state-keyboardcontext--standardlayoutresolver)
- [7. Routing `KeyboardAction`](#7-routing-keyboardaction)
- [8. Configuration: `KeyboardBehavior` (every toggle)](#8-configuration-keyboardbehavior-every-toggle)
- [9. Appearance & Liquid Glass transparency](#9-appearance--liquid-glass-transparency)
- [10. Styling: `KeyboardStyleProvider`](#10-styling-keyboardstyleprovider)
- [11. Text-level behaviors (host applies)](#11-text-level-behaviors-host-applies)
- [12. Autocomplete module](#12-autocomplete-module)
- [13. Emoji module](#13-emoji-module)
- [14. Slide-to-type](#14-slide-to-type)
- [15. Sharing settings with your app (App Group)](#15-sharing-settings-with-your-app-app-group)
- [16. Sizing the keyboard](#16-sizing-the-keyboard)
- [17. Full behavior reference](#17-full-behavior-reference)
- [12b. Next-word prediction](#12b-next-word-prediction)
- [17b. The URL keyboard's domain key](#17b-the-url-keyboards-domain-key)
- [18b. Accessibility (VoiceOver)](#18b-accessibility-voiceover)
- [18. Performance & the render path](#18-performance--the-render-path)
- [19. UI testing (XCUITest harness)](#19-ui-testing-xcuitest-harness)

---

## 1. What you get

| Product | Import | Purpose |
|---|---|---|
| `PressboardKit` | core | Render view, layout state, callouts, shift/caps, spacebar-drag, feedback, all text-transform logic |
| `PressboardKitLayouts` | layouts | `StandardLayoutResolver` + the CZ/EN data tables and QWERTY/AZERTY/QWERTZ/Dvorak/Colemak arrangements |
| `PressboardKitAutocomplete` | optional | `SuggestionProvider` protocol + a dictionary-backed provider |
| `PressboardKitEmoji` | optional | Emoji panel, full-text emoji search, recents store |

The kit **has no dependency on any host app**. You inject your theme, your suggestion
source, and your settings; the kit imports none of your code.

## 2. Requirements & installation

- **iOS 26+, Swift 6, Swift Package Manager.**

Add the package (Xcode → *File ▸ Add Package Dependencies…*, or `Package.swift`):

```swift
.package(url: "https://your.git/PressboardKit.git", from: "0.1.0")
```

Then link the products your **keyboard extension target** needs (at minimum
`PressboardKit` + `PressboardKitLayouts`):

```swift
.target(
    name: "MyKeyboardExtension",
    dependencies: [
        .product(name: "PressboardKit", package: "PressboardKit"),
        .product(name: "PressboardKitLayouts", package: "PressboardKit"),
        .product(name: "PressboardKitAutocomplete", package: "PressboardKit"), // optional
        .product(name: "PressboardKitEmoji", package: "PressboardKit"),        // optional
    ]
)
```

> **Memory:** a keyboard extension has a tight (~30–48 MB) budget. Autocomplete and emoji
> are separate products precisely so you can leave them out if you don't use them.

## 3. Mental model

PressboardKit is split into two halves, and this split is the key to using it correctly:

1. **A SwiftUI render view** (`PressboardKeyboardView`) that draws a *resolved layout* and
   reports **user intents** (`KeyboardAction`: a character was tapped, backspace, shift,
   the caret moved, a glide finished, …). It never touches the text document.
2. **Pure, testable logic** (shift state machine, auto-cap, smart punctuation, delete-by-word,
   autocorrect, math, swipe decode, …) that you call from your `UIInputViewController` to
   mutate the `textDocumentProxy`.

**You own the text document.** The kit tells you *what the user did*; you decide *what to
insert/delete*. This keeps the engine host-agnostic and lets you intercept anything.

A tiny **host state object** (an `ObservableObject`) ties them together: it holds a
`KeyboardContext` (locale, page, shift case), republishes a resolved `KeyboardLayoutDefinition`
whenever state changes, and exposes helpers. `PressboardKitApp`'s `PressboardController` is a
complete implementation of this object (used by the demo and reusable directly).

```
  UIInputViewController (yours)
        │  hosts
        ▼
  UIHostingController( PressboardKeyboardView )   ← draws model.layout
        │  onAction: KeyboardAction
        ▼
  Host model (ObservableObject)  ── KeyboardContext ──► StandardLayoutResolver ──► layout
        │  helper logic (shift, auto-cap, …)
        ▼
  textDocumentProxy  (insert / delete / move caret)
```

## 4. Quick start — a keyboard in ~10 lines (`PressboardKitApp`)

The `PressboardKitApp` module assembles the whole keyboard for you — the controller, the SwiftUI view,
text editing (smart punctuation, auto-correction, cursor moves), sizing, and the input-trait adaptation
(email/URL/number pad/return label). A real extension **subclasses `PressboardInputViewController`** and
returns a configuration — that is the entire extension:

```swift
import PressboardKitApp

final class KeyboardViewController: PressboardInputViewController {
    override func makeConfiguration() -> PressboardConfiguration {
        PressboardConfiguration(
            // behavior defaults to `.pressboardDefault` (native + uniform keys, no globe/mic, key-down
            // input). Pass `.standard` for pure native, or your own `KeyboardBehavior`.
            locale: .english,
            words: [.english: myEnglishWords,       // your bundled frequency lists (most-frequent first)
                    .czech:   myCzechWords]
        )
    }
}
```

`KeyboardBehavior` reproduces the native keyboard and every feature is a toggle (§8 / §17). Two presets:
`.standard` (pure native) and `.pressboardDefault` (the kit's recommended default, used when you don't
pass a `behavior`).

**Persist settings across the app + extension (App Group).** Pass the hooks and the controller reloads
them on `viewWillAppear` and saves on every change:

```swift
PressboardConfiguration(
    behavior: Store.load(), locale: Store.loadLocale(), words: …,
    emojiDefaults: UserDefaults(suiteName: "group.you.app")!,   // shared emoji recents
    loadBehavior: Store.load,  saveBehavior: Store.save,
    loadLocale:   Store.loadLocale, saveLocale: Store.saveLocale)
```

**Edit settings in your app.** Bind straight into the controller's published `behavior`, e.g.
`Toggle("Haptics", isOn: $controller.behavior.hapticFeedback)`; change language with
`controller.setLocale(_:)`.

**Embed in-app (no extension).** Point the same controller at any `PressboardTextDocument` — a
`StringTextDocument` for a preview, or your own type — and drop in `PressboardRootView`:

```swift
@StateObject private var controller = PressboardController(configuration: .init(behavior: .standard))
@StateObject private var document = StringTextDocument()
// …
PressboardRootView(controller: controller)
    .onAppear { controller.document = document }
```

> **Key sound** works out of the box when `KeyboardBehavior.keySound` is on — the engine plays the
> system keyboard clicks directly (`AudioServicesPlaySystemSound`), so **no host setup is required**
> and it works without Full Access. It respects the device's silent switch. Note: the **Simulator
> does not play** these clicks; test key sound on a physical device.

The sections below document the **lower-level API** that `PressboardKitApp` is built on — the render
view, context, action routing, styling — for a host that wants to assemble the keyboard itself instead
of subclassing `PressboardInputViewController`.

## 5. The render view: `PressboardKeyboardView`

```swift
public init(
    layout: KeyboardLayoutDefinition,
    style: any KeyboardStyleProvider = NativeKeyboardStyle(),
    behavior: KeyboardBehavior = .standard,
    onAction: @escaping (KeyboardAction) -> Void,
    onCursorMove: @escaping (Bool) -> Void = { _ in },               // spacebar cursor move start/end
    onExpandOneHanded: (() -> Void)? = nil,
    onSelectOneHanded: @escaping (OneHandedMode) -> Void = { _ in },
    topInset: CGFloat = 0,                                            // key-pop headroom (see §9b)
    @ViewBuilder toolbar: () -> some View = { EmptyView() },          // above the keys
    buttonContent: (KeyboardLayoutItem, AnyView) -> AnyView = { _, d in d }, // override any key
    @ViewBuilder emojiKeyboard: () -> some View = { EmptyView() },    // shown on the emoji page
    @ViewBuilder nextKeyboardMenuRow: () -> some View = { EmptyView() } // globe long-press menu row
)
```

- **`layout`** — the resolved layout from `StandardLayoutResolver` (see §6).
- **`style`** — visuals; `NativeKeyboardStyle()` by default (see §10).
- **`behavior`** — the parametric config (see §8). The view derives its own key feedback,
  character preview, callouts, glide, one-handed layout, and glass transparency from it.
- **`topInset`** — transparent headroom above the toolbar so the top-row key-pop isn't clipped by the
  keyboard's top edge in an extension (see §9b).
- **`onCursorMove`** — called with `true` when a spacebar drag starts moving the cursor and `false`
  when it ends. The view already blanks every key for the duration (native); use this to **freeze your
  predictions and keyboard case while moving** (skip `updateSuggestions` / `updateCase` between `true`
  and `false`) and to run one resync on `false` for the new caret position.
- **`toolbar`** — your suggestion bar / emoji-search header goes here (see §12, §13).
- **`buttonContent`** — return a replacement view for a given key (e.g. brand a specific key);
  return the passed-in default to keep native.
- **`emojiKeyboard`** — the view rendered when `layout.keyboardType == .emojis` (see §13).
- **`onExpandOneHanded` / `onSelectOneHanded` / `nextKeyboardMenuRow`** — one-handed mode &
  the globe long-press menu (see §8).
- **`isViableNextCharacter`** — `(Character) -> Bool`, only consulted under `behavior.
  predictiveKeyTargeting` (see §8/§17, ENG-119): whether appending that letter to the word typed so far
  is still a viable dictionary prefix, e.g. from `DictionarySuggestionProvider.isViablePrefix(_:
  followedBy:)`. `PressboardController`/`PressboardRootView` wire this for you automatically; supply it
  yourself only if you're driving a bare `PressboardKeyboardView`. `nil` (default) behaves like the flag
  being off.

## 6. State: `KeyboardContext` + `StandardLayoutResolver`

`KeyboardContext` is the mutable state a layout is resolved from:

```swift
public final class KeyboardContext {
    public var locale: KeyboardLocale                 // .english / .czech
    public var layoutTypeOverride: KeyboardLayoutType? // nil = locale default
    public var keyboardType: KeyboardType             // .alphabetic / .numeric / .symbolic / .emojis
    public var keyboardCase: KeyboardCase             // .lowercased / .uppercased / .auto / .capsLocked
    public var inputType: KeyboardInputType           // .normal / .email / .url / .numberPad / … (field traits)
    public var returnKeyType: ReturnKeyType           // .default / .go / .search / .send / … (return label)
}
```

**Feeding the field traits (`adaptToInputType`).** In `viewWillAppear` and `textDidChange`, read the
proxy's input traits and update the context so the layout adapts to the field (email "@"/"."; URL
".com"; number pad; Go/Search/… return). The engine ships the UIKit mappings:

```swift
context.inputType     = KeyboardInputType(textDocumentProxy.keyboardType ?? .default)
context.returnKeyType = ReturnKeyType(textDocumentProxy.returnKeyType ?? .default)
// then re-resolve (below). Pass `adaptToInputType:` so the toggle can force the plain keyboard.
```

Note: iOS never presents a third-party keyboard for `.numberPad`/`.phonePad`/`.decimalPad` fields (it
uses the system pad), so those pads only apply when you embed the engine as your own in-app keyboard.

`StandardLayoutResolver.layout` turns that state into a `KeyboardLayoutDefinition` the view
draws. There's a convenience overload that reads a whole `KeyboardContext`:

```swift
// Convenience — reads the context; pass the visibility toggles from your behavior.
layout = StandardLayoutResolver.layout(
    for: context,
    adaptToInputType: behavior.adaptToInputType,
    dictation: behavior.dictation,
    emojiKey: behavior.emojiKey,
    nextKeyboardKey: behavior.nextKeyboardKey)

// Explicit form (pure, no context object):
layout = StandardLayoutResolver.layout(
    locale: .english, layoutType: .qwerty,
    keyboardType: .alphabetic, keyboardCase: .lowercased,
    dictation: false, emojiKey: true, nextKeyboardKey: true)
```

**Re-resolve whenever any of those change** (case, page, locale, layout override) and publish
the new `layout` — that is what re-renders the keys. Adding a language is a data-table change
in `PressboardKitLayouts`, not an engine change.

## 7. Routing `KeyboardAction`

`onAction` delivers every user intent. Handle the ones your feature set needs:

| Action | Meaning | Typical handling |
|---|---|---|
| `.character(String)` | a key/callout produced text | insert (after smart-punctuation, §11) |
| `.space` | space key | insert space (after ". " shortcut, §11) |
| `.backspace` | backspace tap | `deleteBackward()` |
| `.deleteWord` | backspace hold escalated | delete a whole word (§11) |
| `.primary` | return/go/search/✓ | insert `"\n"` (or close emoji search) |
| `.shift` | shift tapped | `model.toggleShift()` |
| `.keyboardType(KeyboardType)` | page switch (123 / #+= / ABC / emoji) | `model.setPage(_:)` / open emoji |
| `.moveCursor(Int)` | spacebar-drag caret move (horizontal) | `adjustTextPosition(byCharacterOffset:)` |
| `.moveCursorSentence(Int)` | spacebar-drag caret move (vertical, − back / + forward) | `SpacebarDragLogic.characterOffset(forSentences:before:after:)` from the doc context, then `adjustTextPosition`. Sentence stops (start / after `.` `!` `?` `…` runs / after `\n` / end) rather than rendered lines: the proxy exposes no text geometry **and** only a truncated window, so wrap points are unknowable — see `characterOffset`'s doc (ENG-128) |
| `.slideChanged([Character])` | live glide in progress | preview suggestions, no insert (§14) |
| `.slideTyped([Character])` | glide finished | decode + insert word (§14) |
| `.nextKeyboard` | globe tapped | `advanceToNextInputMode()` |
| `.dictation` | mic tapped | start your speech-to-text |
| `.custom(String)`, `.none` | host-defined / spacer | ignore or handle as you like |

See `PressboardController.handle(_:)` (in `PressboardKitApp`) for a complete switch.

## 8. Configuration: `KeyboardBehavior` (every toggle)

`KeyboardBehavior` is one `Codable`/`Equatable`/`Sendable` value that mirrors the native
*Settings ▸ Keyboard* toggles. **`.standard` is the native default.** Pass it to
`PressboardKeyboardView` and to the logic helpers. Flip any field to opt out:

```swift
var behavior = KeyboardBehavior.standard
behavior.autoCorrection = false          // e.g. disable autocorrect
behavior.layoutTypeOverride = .dvorak    // force a key arrangement
behavior.oneHandedMode = .right
behavior.translucentBackground = false   // opaque background instead of glass
```

Behaviors are of two kinds:

- **View-level** (the engine view applies them itself): `characterPreview`,
  `characterPreviewStyle`, `hapticFeedback`, `keySound`, `keyAnimationsEnabled`, `enableCapsLock`, `slideToType`,
  `insertOnKeyDown`, `multiTouchInputLayer`, `touchHitTestVerticalOffset`, `predictiveKeyTargeting`, `adaptToInputType`, `oneHandedMode`/`oneHandedScale`/
  `oneHandedGlobeMenu`, `translucentBackground`, and the key-visibility toggles (`nextKeyboardKey`,
  `dictation`, `emojiKey`, `emojiSearch`).
- **Text-level** (pure logic the engine *provides* but the **host applies** to the document):
  `autoCapitalization`, `autoShiftToLowercase`, `smartPunctuation`, `periodShortcut`,
  `punctuationSpacing`, `deleteByWord`, `predictiveText`, `autoCorrection`, `checkSpelling`,
  `showMathResults`, `emojiSuggestions`, `returnToLettersAfterSpace`,
  `slideToTypeRestoresDiacritics`. See §11 for how to apply each.

The full field-by-field table is in [§17](#17-full-behavior-reference).

## 9. Appearance & Liquid Glass transparency

*(Latest feature — `translucentBackground`, default **on**.)*

The native iOS 26 keyboard is **not** an opaque rectangle: it floats on the system's
translucent "liquid glass" backdrop and blends with the app behind it. PressboardKit
reproduces this, and it takes **two cooperating pieces** — one in the engine, one in your
extension:

### 9a. Engine side (automatic)

With `behavior.translucentBackground == true` (the default):

- The keyboard's own background is drawn **fully clear**. A keyboard extension already renders
  over the system's translucent backdrop, so **any** material/colour fill of our own only adds
  a grey veil that reads as an opaque rectangle. Clear lets the real system glass show.
- The **keys themselves are translucent** on the glass (`KeyboardStyleProvider.keyFillOnGlass`),
  with alphas measured from the native iOS 26 keyboard, so they blend with the app instead of
  looking like darker opaque blocks. When `translucentBackground` is off, keys use the opaque
  measured `keyFill` and the background is the solid measured colour.

You get all of this for free by leaving `translucentBackground` at its default.

### 9b. Extension side (you must do this)

The glass only shows if **nothing behind the SwiftUI view is opaque**. In your
`UIInputViewController`:

```swift
host.view.backgroundColor = .clear   // the UIHostingController's view
view.backgroundColor = .clear        // the input view controller's own view
host.view.clipsToBounds = false      // your own views must not clip
view.clipsToBounds = false
```

**Top-row key-pop.** The character preview for the top key row extends above the keyboard. A keyboard
extension **cannot draw outside its frame** (the system clips it — only the native keyboard is
privileged to draw over the app), and un-clipping the system's container views does *not* help. Instead
reserve the overflow as headroom *inside* a slightly taller keyboard: pass
`PressboardKeyboardView(topInset:)` the value `max(0, style.metrics.keyPopOverflow - toolbarHeight)`
(0 on the emoji page, whose taller header already has room) and add the same amount to your input
view's height. The inset is filled by the translucent backdrop, so it reads as a marginally taller
keyboard, not a gap. (In an in-app preview the overflow isn't clipped, so the inset is only needed in
the extension — but it's harmless to pass in both.)

If you skip this, the extension's default opaque view sits in front of the system backdrop and
you're back to a rectangle — even though the engine is drawing clear.

> **Why not `.regularMaterial` / a `UIVisualEffectView`?** A custom keyboard is **not** given
> the host app's rendered content, so a material can't blur it — it just paints a flat grey
> veil that reads as a rectangle. The correct approach is *clear* everything and rely on the
> **system-provided** keyboard backdrop. (This is exactly what the native keyboard and a
> correct KeyboardKit setup do.)

To opt out (draw a solid background, e.g. for a fully custom opaque theme):

```swift
behavior.translucentBackground = false
// and give your extension's view an opaque backgroundColor instead of .clear
```

## 10. Styling: `KeyboardStyleProvider`

Visuals come from a `KeyboardStyleProvider`. `NativeKeyboardStyle()` is the default and
reproduces the measured native look (light + dark). Supply your own to re-theme — you only
override what you want; metrics/fonts keep native proportions.

```swift
public protocol KeyboardStyleProvider {
    var metrics: NativeMetrics { get }                      // row heights, spacing, radius, fonts
    func backgroundColor() -> KeyboardColor                 // used only when translucentBackground == false
    func keyFill(for role: KeyRole) -> KeyboardColor        // opaque key fill (solid background)
    func keyFillOnGlass(for role: KeyRole) -> KeyboardColor // translucent key fill (glass); defaults to keyFill
    func keyTextColor(for role: KeyRole) -> KeyboardColor
    func keyFontSize(for action: KeyboardAction) -> CGFloat // has a native default
}
```

- `KeyRole` is `.standard` (letters/space) or `.control` (shift, backspace, 123, globe, …).
- `KeyboardColor` carries a `light` and `dark` `RGBA` (alpha supported → translucency).
- **If you provide a custom style and want native-accurate glass**, implement
  `keyFillOnGlass` with translucent colours. If you don't, it falls back to your opaque
  `keyFill`, which will read heavier on the glass.

```swift
struct BrandStyle: KeyboardStyleProvider {
    var metrics: NativeMetrics = .iPhonePortrait
    func backgroundColor() -> KeyboardColor { .init(light: .init(0.90,0.90,0.92), dark: .init(0.11,0.11,0.12)) }
    func keyFill(for role: KeyRole) -> KeyboardColor { /* your opaque colours */ }
    func keyTextColor(for role: KeyRole) -> KeyboardColor { .init(light: .init(0,0,0), dark: .init(1,1,1)) }
    // optional but recommended for glass:
    func keyFillOnGlass(for role: KeyRole) -> KeyboardColor {
        role == .standard ? .init(light: .init(1,1,1,1), dark: .init(1,1,1,0.13))
                          : .init(light: .init(1,1,1,1), dark: .init(1,1,1,0.04))
    }
}
```

## 11. Text-level behaviors (host applies)

The engine emits `.character` / `.space` / `.backspace` etc.; **you** run the matching pure
helper before touching the document. All helpers take the relevant `behavior` flag and no-op
when it's off, so you can wire them unconditionally. Pattern (from `DemoApp`):

```swift
// Smart punctuation & the " ." → ". " fix, on character insert:
func insertCharacter(_ c: String) {
    let before = textDocumentProxy.documentContextBeforeInput ?? ""
    if let ch = c.first, c.count == 1,
       let fixed = PunctuationSpacing.fixup(for: ch, textBeforeCursor: before,
                                            enabled: behavior.punctuationSpacing) {
        textDocumentProxy.deleteBackward(); textDocumentProxy.insertText(fixed)
    } else if let ch = c.first, c.count == 1,
       let r = SmartPunctuation.transform(for: ch, textBeforeCursor: before,
                                          locale: context.locale, enabled: behavior.smartPunctuation) {
        switch r {
        case .insert(let s):      textDocumentProxy.insertText(s)
        case .replaceLast(let s): textDocumentProxy.deleteBackward(); textDocumentProxy.insertText(s)
        }
    } else {
        textDocumentProxy.insertText(c)
    }
    sync()
}

// Space: autocorrect the finished word, then the ". " double-space shortcut:
func insertSpace() {
    let before = textDocumentProxy.documentContextBeforeInput ?? ""
    if let fix = autocorrection(forTextBeforeCursor: before) {          // AutoCorrection + SystemSpellChecker
        let word = SuggestionText.currentWord(in: before)
        for _ in 0..<word.count { textDocumentProxy.deleteBackward() }
        textDocumentProxy.insertText(fix)
    }
    let now = textDocumentProxy.documentContextBeforeInput ?? ""
    // `sinceLastSpace` is the time since the user's previous space keystroke (nil if there wasn't one) —
    // the shortcut only fires inside `periodShortcutWindow`, like native.
    if PeriodShortcut.shouldExpand(textBeforeCursor: now, sinceLastSpace: sinceLastSpace,
                                   window: behavior.periodShortcutWindow, enabled: behavior.periodShortcut) {
        textDocumentProxy.deleteBackward(); textDocumentProxy.insertText(". ")
    } else {
        textDocumentProxy.insertText(" ")
    }
    sync()
}

// Delete-by-word on an escalated backspace hold (.deleteWord action):
func deleteWord() {
    let before = textDocumentProxy.documentContextBeforeInput ?? ""
    let n = DeleteByWord.wordDeletionLength(textBeforeCursor: before)
    for _ in 0..<n { textDocumentProxy.deleteBackward() }
    sync()
}
```

Other helpers you'll use:

- **`AutoCapitalization.shouldCapitalizeNext(afterTextBeforeCursor:enabled:)`** — drive the
  shift case after each change (see §4a `updateCase`).
- **`ShiftLogic.afterShiftTap(current:sinceLastShiftTap:capsLockEnabled:)`** — the shift/caps
  state machine.
- **`LayoutAutoSwitch.nextType(after:on:enabled:)`** — after a space/newline on the 123/#+=
  page, return `.alphabetic` to flip back (call on space/return/character).
- **`MathResult.expressionAndResult(...)` / `.suggestionVariants(_:)` / `.variantTexts(...)` /
  `.expressionLength(...)`** — offer a typed arithmetic result in the suggestion bar (gated by
  `showMathResults`). `suggestionVariants` builds the native three slots — the expression ("1+1"),
  the highlighted `expression=result` ("1+1=2"), and the result ("2") — and `variantTexts` lets you
  recognise a picked one so it replaces the trailing expression (`expressionLength`). The result
  appears in the same bar as predictions, so show/size the bar when **either** `predictiveText` **or**
  `showMathResults` is on — otherwise it has nowhere to render. (`.result(...)` remains for the bare
  result.)
- **`EmojiSuggestion.emoji(forTextBeforeCursor:)` / `.emoji(forWord:)`** — the emoji for the word
  being typed (e.g. "Hi"/"ahoj" → 👋), gated by `emojiSuggestions`. Surface it in the bar (a
  highlighted `Suggestion`); picking it replaces the current word like any suggestion. Bilingual
  (EN/CZ), case- and diacritic-insensitive. Include it in the bar's show/size condition too.
- **`SuggestionText.currentWord(in:)`** — the word under the cursor (used by autocorrect &
  suggestion replacement).

## 12. Autocomplete module

`import PressboardKitAutocomplete`.

```swift
public protocol SuggestionProvider { func suggestions(for textBeforeCursor: String) -> [Suggestion] }

// Built-in dictionary provider — feed it a word list (frequency-ordered is best):
let provider = DictionarySuggestionProvider(words: myWords)   // or .english / .czech built-ins
let suggestions = behavior.predictiveText ? provider.suggestions(for: before) : []
```

Render the native 3-slot bar in the `toolbar` slot and apply a pick:

```swift
PressboardKeyboardView(layout: model.layout, behavior: model.behavior, onAction: handle,
    toolbar: {
        if model.behavior.predictiveText {
            SuggestionBar(suggestions: model.suggestions, onPick: applyPick)
        }
    })

func applyPick(_ s: Suggestion) {
    let before = textDocumentProxy.documentContextBeforeInput ?? ""
    let word = SuggestionText.currentWord(in: before)          // or MathResult.expressionLength(...)
    for _ in 0..<word.count { textDocumentProxy.deleteBackward() }
    textDocumentProxy.insertText(s.text + " ")
    sync()
}
```

### The dictionary arrives late on a cold start

`PressboardController` does not build its dictionary on the way to the first frame — deriving the merged
lists costs ~160 ms, and a keyboard extension that is slow to appear loses races with hosts that ask for
it while they activate. A fresh controller starts with an empty provider and adopts the real one when the
background build lands (ENG-182); a controller for a language set this process has already built is ready
immediately.

Nothing in the keyboard waits for it: typing, layout, feedback and the emoji panel never needed the
dictionary. Suggestions, autocorrect and predictive key targeting are inert until it arrives — the same
"no dictionary" behaviour they document for a host that ships no word list at all. If you want to hold
something back until then, `isDictionaryReady` is `@Published`, and `await controller.dictionaryReady()`
returns once it is in place (immediately when it already was).

Autocorrect uses a spell checker behind the `SpellChecking` protocol; feed it to
`AutoCorrection.correction(for:using:enabled:)`. Use `LayeredSpellChecker(language:provider:neighbors:)`
unless you have a reason not to — it is what `PressboardController` builds:

```swift
LayeredSpellChecker(
    language: locale.identifier,                 // "cs" — resolved to the checker's own "cs_CZ"
    provider: provider,                          // the merged dictionary of every enabled language
    neighbors: KeyNeighbors(rows: CharacterRows.rows(for: locale, layoutType: layoutType))
)
```

It layers three sources, because `UITextChecker` is not all-or-nothing per language — its misspelling
finder and its guesses ship independently. On a Czech phone the finder never flags a word, not even
"codelas", while `guesses` answers correctly ("vwcer" → "večer"); ask only through the finder, the
obvious reading of the API, and Czech gets nothing (ENG-170, ENG-173).

1. `SystemSpellChecker.correction(for:)` where `flagsMisspellings` is true — the system's own verdict,
   taken as is, including when it declines to correct.
2. Otherwise `SystemSpellChecker.guesses(for:)`, behind two guards that stand in for the missing finder:
   the word must be one `knowsWord(_:)` does **not** know (its completion lexicon is installed for Czech
   even where its spelling data is not), and the guess must be a letter on the wrong key or two letters
   in the wrong order. Both are load-bearing — without the first, the real word "vrčel" becomes "vrčeli";
   without the second, "codelas" becomes "odělas".
3. Otherwise `DictionarySpellChecker`, correcting from the bundled word list. Guesses that add or drop a
   letter land here rather than in step 2, so they are only applied when our own list carries them.

The pieces are public if you want to assemble them differently. `SystemSpellChecking` is the protocol —
`flagsMisspellings`, `guesses(for:)`, `knowsWord(_:)` — and `LayeredSpellChecker(system:provider:neighbors:)`
takes any implementation of it, including a fake, on any platform. `SystemSpellChecker(language:)` is the
`UITextChecker` one and also exposes `resolvedLanguage` (bare `"cs"` is not one of its languages — it
names them `cs_CZ` and matches literally). `TypingSlip.between(typed:intended:neighbors:)` is the slip
test, and `KeyNeighbors` is built from the rows on screen.

## 13. Emoji module

`import PressboardKitEmoji`. Render `EmojiKeyboard` in the `emojiKeyboard` slot; it shows when
`context.keyboardType == .emojis` (set that when you receive `.keyboardType(.emojis)`).

```swift
emojiKeyboard: {
    EmojiKeyboard(
        recents: recentsStore.recents(),
        onSelect: { insert($0); recentsStore.markUsed($0) },
        onBackspace: { self.delete() },
        onABC: { model.setPage(.alphabetic) })
}
```

- **Recents:** `EmojiRecentsStore(defaults:)` — pass your App Group `UserDefaults` to share
  across app/extension.
- **Full-text search** (gated by `behavior.emojiSearch`): `EmojiSearch.search(query) -> [String]`
  (diacritic/case-insensitive, EN+CZ), rendered via `EmojiSearchHeader` in the `toolbar` slot.
  Route `.character`/`.backspace` into the query while search is active (see
  `PressboardController`'s emoji-search methods).

## 14. Slide-to-type

Gated by `behavior.slideToType` (on by default). The engine detects a glide across ≥3 letters
and emits:

- **`.slideChanged([Character])`** repeatedly during the glide → show a *live preview* in the
  bar without inserting:
  ```swift
  let candidates = SwipeDecoder.decodeCandidates(keySequence: keys, candidates: provider.candidateWordsRanked, limit: 3)
  model.suggestions = candidates.enumerated().map { Suggestion(text: $1, isAutocorrect: $0 == 0) }
  ```
- **`.slideTyped([Character])`** on lift-off → decode, insert the best word + a space, and keep
  the alternatives in the bar so a tap can replace it (native behavior). See
  `PressboardController` (it handles the swipe internally).

`SwipeDecoder` needs the candidate word list, ranked within each word's own source language
(`DictionarySuggestionProvider.candidateWordsRanked`) — **not** `candidateWords`. On a
multi-language provider (`init(languageLists:)`, what `PressboardController.makeProvider` builds),
`candidateWords` is one flat, concatenated array (primary language first), so a plain
frequency-ordered decode would let the first-listed language's words always outrank every other
enabled language's, regardless of actual frequency (bugfix/ENG-126). `candidateWordsRanked` pairs
each word with its rank inside its own language instead, so e.g. a very common Czech word can beat
a rare English one even on an EN-primary keyboard. (`candidateWords`/`decodeCandidates(…
candidates: [String]…)` still work — for a single-language provider, or a caller that doesn't care
about cross-language fairness — but default to `candidateWordsRanked` for anything multi-language.)

**Diacritics:** matching is diacritic-insensitive (a glide only ever crosses a keyboard's
base-letter keys — diacritics are a long-press callout, never swiped through), so a Czech glide
over "m", "a", "s" matches the dictionary entry "máš". What gets **inserted** is controlled by
`behavior.slideToTypeRestoresDiacritics` (default `true`: insert "máš"). Set it `false` to insert
the folded, base-letter form instead ("mas") — this is what the native iOS keyboard does (measured
on-device, ENG-126); the default here deliberately diverges because "máš"/"děláš" are the
grammatically correct words and restoring them is free once the decoder already found that exact
entry. `PressboardController` applies this itself; a custom integration decoding manually should
fold `SwipeDecoder`'s result the same way when the flag is off.

## 15. Sharing settings with your app (App Group)

Let your containing app's settings screen edit a `KeyboardBehavior` and have the extension pick
it up. `KeyboardBehavior` is `Codable`; persist it in an **App Group** so both processes see it.
`DemoApp/Shared/BehaviorStore.swift` is the reference:

```swift
enum BehaviorStore {
    static let appGroup = "group.com.yourcompany.app"
    static var defaults: UserDefaults { UserDefaults(suiteName: appGroup) ?? .standard }
    static func save(_ b: KeyboardBehavior) { defaults.set(try? JSONEncoder().encode(b), forKey: "behavior") }
    static func load() -> KeyboardBehavior {
        guard let d = defaults.data(forKey: "behavior"),
              let b = try? JSONDecoder().decode(KeyboardBehavior.self, from: d) else { return .standard }
        return b
    }
}
```

- The app writes on change (`BehaviorStore.save`); the extension reloads in `viewWillAppear`
  (`BehaviorStore.load`) and re-resolves the layout.
- Add the **App Group capability** to *both* targets and use the same group id.
- Haptics need **Full Access** (`RequestsOpenAccess = YES`); the key-click sound needs the host
  to be input-click-capable.

## 15b. Licensing: registering your application

**Without a licence the keyboard types, and does nothing else.** English only, no autocorrect,
no predictions, no bundled dictionaries, no slide-to-type, no emoji panel, no themes. Nothing
crashes and nothing nags — it degrades, because a keyboard that stops working is a keyboard
somebody cannot use to ask for help.

What unlocks it is a **lease**: a short-lived, signed statement naming your application. You
register the application once, your containing app exchanges its licence key for a lease, and the
keyboard extension verifies that lease **offline**.

```swift
let config = PressboardConfiguration(
    // The extension never fetches anything — it has no network until the user grants Full
    // Access. Your app fetches and stores; this only reads, on every appearance, so a renewed
    // lease is picked up without the extension being rebuilt.
    loadLicence: { LicenceStore.lease },          // Data? from your App Group
    loadTrustBundle: { LicenceStore.trustBundle }, // Data? from your App Group
    // Your *containing app's* bundle id, not the extension's. Inside the extension
    // `Bundle.main` is `com.acme.app.keyboard`, which is not what the licence was issued for.
    applicationBundleID: "com.acme.app",
    onLicenceStatus: { status in LicenceStore.lastStatus = status })
```

`onLicenceStatus` is how you find out where you stand — `.licensed(plan:softExpiry:)`,
`.degrading(plan:hardExpiry:)` or `.free(reason:)` — so your settings screen can say "expires in
five days" instead of leaving somebody to notice what stopped working.

**A lease is short-lived, and refreshing it is the host's job.** The service issues one good for
about a week and asks to be called daily; the exact dates are in the lease rather than in this
SDK, so they can be widened during an incident without you shipping anything. Six failed days —
no network, an outage of ours — cost nothing. The seventh drops the application to the free tier.

That means a person who installs your keyboard and never opens your app again stops being
licensed a week later. **Refresh in the background** (`BGAppRefreshTask`) rather than only on
launch; a keyboard is used daily and its containing app is opened once.

**Every failure lands on the free tier, never on an error.** No lease, an unreadable one, a
signature we do not trust, a lease issued to another application, an expired one, or a clock that
has moved backwards: all of them type. The only thing that changes is what is unlocked.

The bundle identifier is checked twice: the lease must name your app, and the process asking must
belong to it (`com.acme.app.keyboard` against a lease for `com.acme.app`). A lease issued for
somebody else's application is worth nothing in yours.

## 16. Sizing the keyboard

The extension controls its own height. Compute it from the style metrics and update it when the
page changes (emoji is taller; a suggestion/search bar adds its height):

```swift
func desiredHeight() -> CGFloat {
    let keyArea = NativeKeyboardStyle().metrics.keyAreaHeight(rowCount: 4)
    if model.layout.keyboardType == .emojis { return (behavior.emojiSearch ? EmojiSearchHeader.height : 0) + keyArea + 44 }
    return (behavior.predictiveText ? SuggestionBar.height : 0) + keyArea
}
// Activate a height constraint on `view` at priority 999 and update its constant on mode change.
```

See `KeyboardViewController.desiredHeight()` / `updateHeight()` in `DemoApp`.

## 17. Full behavior reference

Every `KeyboardBehavior` field, its default (= native), where it's applied, and how you honour
it. **V = the engine view applies it for you; H = you apply it in the host.**

| Field | Default | Where | Notes / how to honour |
|---|---|---|---|
| `characterPreview` | `true` | V | Enlarged key-pop bubble on press. |
| `characterPreviewStyle` | `.native` | V | Per-key pop: `.native` (the real iOS shape, re-measured against a device across two passes, ENG-110 — a standalone **square**, `NativeMetrics.nativePreviewSize` = `keyRowHeight * 1.35` regardless of the pressed key's width, no neck, floating `nativePreviewGap` above the key; the pressed key stays visible, letter and all, lightened *more* than the pop above it — native's own pop is darker than its pressed key) / `.connected` (the engine's previous `.native` — a neck-connected pop that blanks the pressed key's letter) / `.curvy` / `.square` (both connected, like `.connected`) / `.floating` (a detached bubble, blanks the key). `.native`'s pop is taller than the connected styles', so the host reserves headroom per active style via `NativeMetrics.popOverflow(for:)` instead of the flat `keyPopOverflow` (unchanged for the connected styles). Overlay styles (no per-key pop, non-blocking): `.bubble` (letters float up & roam), `.bubbleWord` (letters line up on the keys↔autocomplete boundary spelling the word; the whole word pops at the boundary and a separator bubble is dropped, which pops when the next word starts), `.bubbleSentence` (same lineup but each character pops on its own short timer — a sliding tail of what you typed). **Breaking rename** (ENG-110): persisted raw value `"native"` now resolves to the new shape; if you specifically want the old connected shape, store `"connected"` instead. |
| `hapticFeedback` | `false` | V | Needs Full Access. Derived into the view's feedback. |
| `keySound` | `true` | V | System key click, played by the engine (`AudioServicesPlaySystemSound`). No host setup; works without Full Access. Silent on the Simulator (test on device). |
| `keyAnimationsEnabled` | `true` | V | Animate key-press visual feedback (bubble-style push-in pulse, callout-bar reveal). `false` applies those instantly instead — an experiment for testing whether the animations, not the (already-instant) character insert, are what reads as slow. Doesn't touch the emoji panel. |
| `autoCapitalization` | `true` | H | `AutoCapitalization.shouldCapitalizeNext(...)` in `updateCase`. |
| `enableCapsLock` | `true` | V/H | Passed to `ShiftLogic.afterShiftTap(capsLockEnabled:)`. |
| `autoShiftToLowercase` | `true` | H | Drop single shift after one char (else sticky). |
| `smartPunctuation` | `true` | H | `SmartPunctuation.transform(...)` on insert. |
| `periodShortcut` | `true` | H | `PeriodShortcut.shouldExpand(...)` on space. |
| `periodShortcutWindow` | `0.8` s | H | How long after the first space a second one still expands to ". ". Native has such a window (measured 0.75-0.85 s on iOS 26.4, ENG-145); a trailing space that was pasted or typed a minute ago is not expanded. Only matters when `periodShortcut` is on. |
| `punctuationSpacing` | `true` | H | `PunctuationSpacing.fixup(...)` on insert. |
| `deleteByWord` | `true` | V/H | View escalates the hold → `.deleteWord`; host runs `DeleteByWord`. |
| `insertOnKeyDown` | `false` | V | Type a character — and, since ENG-107, the space bar — on press-down (snappier) instead of release; keeps ordering correct around a space during fast typing. Callout replaces, glide/drag-off undo the character; a spacebar cursor-drag undoes the space instead of typing one (and the generic drag-off cancel doesn't re-apply to space — see `KeyDownCommitLogic`). |
| `adaptToInputType` | `true` | V/H | Adapt to the field's `inputType`/`returnKeyType`: email "@"/"."; URL ".com" (its long-press TLDs follow `KeyboardContext.enabledLocales` — ENG-157); number pad; Go/Search/… return. Also suppresses the suggestion bar over a number/phone/decimal pad, where native shows none (`PressboardController.suppressesSuggestionBar`, ENG-154). Host feeds the traits (see below). |
| `predictiveText` | `true` | H | Show `SuggestionBar` + provider suggestions. |
| `autoCorrection` | `true` | H | `AutoCorrection.correction(...)` on space. |
| `autoCorrectMinimumWordLength` | `2` | H | Shortest word auto-correction may rewrite. Native never corrects a lone character ("Add 100 g of flour" must not become "...100 G...", ENG-147); the locale rules in `AutoCorrection.explicitCorrection` (English `i` -> `I`) are exempt. Tokens containing digits are never corrected either. |
| `nextWordPrediction` | `true` | H | Once a word is finished, offer likely continuations instead of an empty bar. What is offered comes from the injected `NextWordPredicting` — the built-in `NextWordPredictor` learns from what the user writes and can also carry seed bigrams. A no-op until something is learned or seeded, so it is safe to leave on. Only matters when `predictiveText` is on. |
| `emptyFieldSuggestions` | `true` | H | Offer a few words in a field nobody has typed into yet, as native does. The words come from the provider (`DictionarySuggestionProvider.openingWords`, default = the primary language's three most frequent; pass your own via `init(words:openingWords:)`). Only matters when `predictiveText` is on. |
| `suppressAutocorrectAfterEdit` | `true` | H | After a word is auto-corrected, if the user backspaces into it, don't re-correct that word until they move on (space/return/cursor move). Native "I rejected the correction" behaviour. Only matters when `autoCorrection` is on. |
| `checkSpelling` | `true` | H | Feeds the system spell checker. |
| `showMathResults` | `true` | H | `MathResult.suggestionVariants(...)` into the bar (3 slots). |
| `emojiSuggestions` | `true` | H | `EmojiSuggestion.emoji(...)` for the typed word into the bar. |
| `theme` | `.native` | V | Colour theme (`KeyboardTheme`): `.native` (system look, keeps glass) / `.midnight` / `.ocean` / `.sunset` / `.forest` / `.graphite`. Recolours the whole keyboard (keys, text, accent, bubbles, callouts, suggestion bar) via one palette; non-native themes are solid (opt out of glass). Resolve to a style with `theme.style()`, or build your own via `ThemePalette` + `ThemedKeyboardStyle`. |
| `translucentBackground` | `true` | V | Liquid glass (§9). Also set the extension views clear. Forced off by non-native `theme`. |
| `uniformKeyColor` | `false` | V | Fill every key (shift/backspace/return/globe/mic/123/…) with the letter-key colour instead of the darker native control colour. |
| `showSpaceLabel` | `true` | V | Show the word "space" on the space bar; off leaves it blank (e.g. to show only a language indicator in the corner via `buttonContent`). |
| `layoutTypeOverride` | `nil` | V | QWERTY/AZERTY/QWERTZ/Dvorak/Colemak; `nil` = locale default. Set `context.layoutTypeOverride`. |
| `returnToLettersAfterSpace` | `true` | H | `LayoutAutoSwitch.nextType(...)` after space/return. |
| `nextKeyboardKey` | `true` | V | Globe key visibility; pass to the resolver + handle `.nextKeyboard`. |
| `slideToType` | `true` | V | Glide detection → `.slideChanged`/`.slideTyped` (§14). |
| `slideToTypeRestoresDiacritics` | `true` | H | A decoded slide-to-type match is inserted with its dictionary diacritics restored ("máš") rather than the folded, base-letter form actually swiped ("mas") — native does the latter (measured on-device, ENG-126); default here deliberately diverges since the accented word is grammatically correct and free to restore once matched. §14. |
| `cursorDragUpdateInterval` | `1.0 / 30.0` | H | ENG-131: how often, at most, a spacebar cursor-drag pushes the caret to the host. Each push is a cross-process `adjustTextPosition`; a drag produces one per touch event (~120/s on ProMotion), which leaves the host's text view no idle moment to repaint the caret — it visibly vanishes during a fast drag. Moves are accumulated and flushed at this interval (and immediately before anything needing the live caret). `0` restores per-step pushing. |
| `spaceCommitsOnRelease` | `true` (native) | H | ENG-132: hold the space bar's space until lift-off, so a press can still turn out to be a cursor-drag instead of typing a space and deleting it again. Ordering is kept by flushing a still-held space as soon as another key commits. Only meaningful with `insertOnKeyDown`. |
| `cursorDragEngageHaptic` | `true` (native) | H | ENG-132: haptic tick when a spacebar press becomes a caret drag. Fires **even when `FeedbackSettings.isHapticEnabled` is off** — it marks a mode change, not a keystroke. Implement `KeyboardFeedback.cursorDragEngaged()` to customise; needs Full Access in an extension. |
| `memoryDiagnosticsInterval` | `0` (off) | H | ENG-137: log the keyboard's own `phys_footprint` to the unified log every N keystrokes (`Perf.logFootprint`). Diagnostic only — measuring a keyboard extension's memory from outside is unreliable (Instruments' Allocations cannot attach to a process it did not launch, and an extension cannot be launched by it), so the keyboard reporting its own is the dependable way to tell whether prolonged typing walks it toward the extension memory limit. Read back with `log collect --device` + `log show --predicate 'subsystem == "PressboardKit"'`. |
| `spaceBarTopExtension` | `12` (measured) | H | ENG-138: how far above its own top edge the space bar still claims a touch. Measured against the real iOS 26 keyboard, where a tap up to ~12pt inside the bottom letter row still types a space. Space-bar-specific on purpose — the letter-row boundary measured as exactly geometric, so this is not a global vertical nudge. `0` restores a purely geometric space bar. |
| `nativeDesignID` | `nil` (automatic) | H | ENG-142: which measured snapshot of the native design to render (`NativeDesign.all`). Automatic picks the newest snapshot measured on an iOS no newer than the device's; pin an id to hold one look deliberately. An id this build doesn't ship resolves back to automatic. |
| (style seam) `keyFillPressed(for:)` | measured native greys | S | ENG-134: fill of a held key that has no key-pop of its own (backspace, return, space, emoji, 123, globe, shift). Defaults to `keyFill(for:)`, so a custom style that doesn't override it simply has no pressed state. Override on your `KeyboardStyleProvider` to restyle it. |
| `oneHandedMode` | `.off` | V | `.left`/`.right` shrink & shift; handle `onExpandOneHanded`. |
| `oneHandedScale` | `OneHandedLayout.defaultScale` | V | Fraction of full width when one-handed. |
| `oneHandedGlobeMenu` | `true` | V | Globe long-press menu; supply `nextKeyboardMenuRow`. |
| `dictation` | `true` | V | Mic key visibility; handle `.dictation` (host does STT). |
| `emojiKey` | `true` | V | Smiley key visibility; pass to the resolver. |
| `emojiSearch` | `true` | V/H | Emoji full-text search header (§13). |
| `multiTouchInputLayer` | `false` (`.standard`) / `true` (`.pressboardDefault`) | V | ENG-108: route key presses through a single shared multi-touch gesture (`KeyTouchGesture`/`KeyTouchTracker`, via `UIGestureRecognizerRepresentable` — a self-inserted UIKit view/recognizer never receives touches inside a real keyboard extension's cross-process hosting, only SwiftUI's own gesture channel does) instead of each key's own SwiftUI `DragGesture`, to fix ordering/dropped touches under fast multi-touch typing. Full parity with the per-key path: key-down commit, backspace, drag-off cancel, press/pop/bubble feedback, long-press callouts, the spacebar cursor-drag, and slide-to-type (the last one is a single per-touch state machine here — no second competing gesture, unlike the per-key path's separate keyboard-level swipe `DragGesture`). Both pipelines are visually identical. On in `.pressboardDefault` after a full round of on-device verification; still off in `.standard`, so existing integrators keep the per-key path until this has more mileage — set it explicitly either way to pin the behaviour. |
| `touchHitTestVerticalOffset` | `0` | V | ENG-110: only under `multiTouchInputLayer` (`KeyHitTestGeometry.key`, not the per-key gesture path, which has no comparable nearest-key hit-test to hook into). Shifts the effective hit-test point up from the raw touch location by this many points before matching it to a key — native reportedly applies a similar correction (a finger's contact point reads lower than where the user is aiming). `0` (default) is today's unshifted geometry; the correct native value hasn't been measured, so don't set this without on-device verification. |
| `predictiveKeyTargeting` | `false` (`.standard`) / `true` (`.pressboardDefault`) | V | ENG-119: also only under `multiTouchInputLayer` (same reason as above). Biases a borderline `KeyHitTestGeometry.key` result toward whichever same-row letter key keeps the word typed so far a viable dictionary prefix (`KeyTargetBias`, backed by `isViableNextCharacter`/`DictionarySuggestionProvider.isViablePrefix`) — measured on-device: typing "keyboar" then tapping just left of "d" (geometrically closer to "s") still resolves to "d" on native, since "keyboard" is a real word and "keyboars" isn't. Never reassigns a touch comfortably inside a key; a no-op at the start of a word or with no dictionary for the active language. `false` (default) is today's pure-geometry hit-test, unchanged; how aggressively native itself applies this hasn't been measured. |

## 18. Performance & the render path

Typing must never hitch. Rendering the whole key grid (~33 keys) costs a few ms, so it must not
re-render on every keystroke. Two things keep it cheap — one built into the engine, one you do:

**Engine side (automatic).** The key grid is an **`Equatable` view** (`KeyGrid`): SwiftUI skips
re-rendering it unless `layout`, `appearance`, or `behavior` actually change. A plain letter
keystroke doesn't change any of those (only your suggestion bar changes), so the grid is skipped
entirely. Measured on an iPhone 17 Pro (Debug): typing 8 mid-word letters → **0 grid re-renders**;
and when the grid *does* re-render (shift, page switch) it's ~6 ms (down from ~10 ms after keys
stopped each carrying their own `GeometryReader`).

> Caveat: `KeyGrid`'s equality compares `layout`/`appearance`/`behavior` and ignores closures and
> the `style`. If you pass a `buttonContent` that depends on external **value** state, or swap the
> `style` dynamically, also change something in `behavior`/`layout` (or accept the grid updating on
> the next such change) so the fast-path doesn't show stale keys.

**Host side (recommended).** Keep state that changes on **every keystroke** — the suggestion
bar's contents — in its **own `ObservableObject`, observed only by the toolbar**. Then a keystroke
publishes only that object and re-renders only the bar, without republishing the model that drives
the keys. The demo does exactly this:

```swift
@MainActor final class SuggestionsModel: ObservableObject {
    @Published var suggestions: [Suggestion] = []
}

@MainActor final class KeyboardModel: ObservableObject {
    let suggestionsModel = SuggestionsModel()      // NOT @Published — its changes don't republish us
    @Published private(set) var layout: KeyboardLayoutDefinition
    // …update suggestionsModel.suggestions on each keystroke; update `layout` only on case/page change
}

// A tiny view that observes only the suggestions, used in the `toolbar` slot:
private struct SuggestionsToolbar: View {
    @ObservedObject var model: SuggestionsModel
    let onPick: (Suggestion) -> Void
    var body: some View { SuggestionBar(suggestions: model.suggestions, onPick: onPick) }
}
```

The two are complementary: the engine's equatable grid protects you even if your model is coarse,
and the split-state pattern keeps the per-keystroke re-render down to just the bar. Also relevant:
the built-in `DictionarySuggestionProvider` indexes its word list so `suggestions(for:)` stays
sub-millisecond — if you write your own provider, keep that call cheap (it runs on every keystroke).

**Don't do the per-keystroke work twice.** You'll typically recompute suggestions/case both after
your own edit *and* from the system's `textDidChange` (which fires for the same edit). Guard against
it by caching the last document context and returning early when it's unchanged:

```swift
private var lastSyncedContext: String?
private func syncAfterInput() {
    let before = textDocumentProxy.documentContextBeforeInput ?? ""
    guard before != lastSyncedContext else { return }   // same edit — already handled
    lastSyncedContext = before
    model.updateSuggestions(for: before)
    model.updateCase(forTextBeforeCursor: before)
}
```

**Build/measure in Release.** The engine's Swift-level work (suggestion lookup, string transforms,
ARC) is many times faster optimized than in a Debug build; judge responsiveness on a Release build.
For reference, measured touch-up→insert latency in the demo is ~1 frame — the SwiftUI gesture path is
already close to its floor, so there's little to gain from a custom UIKit touch layer.

---

## 12b. Next-word prediction

The suggestion provider completes the word being typed. This is the other half: after "see you" the bar
should read "later / tomorrow / soon" rather than nothing.

```swift
PressboardConfiguration(
    // Optional: your own model. Omit it and you get the built-in learning predictor.
    nextWordPredictor: NextWordPredictor(seed: ["see": ["you", "ya"]]),
    // Strongly recommended: a keyboard extension is killed constantly, and without these the
    // predictor forgets everything every few minutes.
    loadLearnedWordPairs: { MyStore.learnedPairs },
    saveLearnedWordPairs: { MyStore.learnedPairs = $0 }
)
```

`NextWordPredictor` merges two sources and always ranks **learned** pairs above the **seed** table. It
learns a pair whenever a word is completed with a space or a return, using the text *after* the separator
landed — so it learns what the user kept, not what auto-correction replaced. The learned table is bounded
by `maxLearnedContexts` (oldest context evicted first) and by `maxFollowersPerContext`, so it cannot grow
into the extension's memory budget.

When nothing is known for the preceding word it returns **nothing**, and the bar stays empty — offering
the most frequent words of the language after every space is noise, not prediction.

Bring your own model by conforming to `NextWordPredicting` (two methods: `nextWords(after:limit:)` and
`learn(_:after:)`); it is kept off the per-keystroke completion path on purpose, so a network-backed
model can't slow typing down.

## 17b. The URL keyboard's domain key

A URL or web-search field gets a ".com" key next to the space bar. Its long-press callouts are built from
**every language you have enabled**, in that order, followed by a generic list:

```swift
controller.setLocales([.czech, .german])   // → .cz, .de, then .net, .org, .eu, …
```

The set reaches the layout through `KeyboardContext.enabledLocales`, which `PressboardController` keeps in
sync; a host driving `StandardLayoutResolver` itself passes `locales:` instead. A language with no single
country (English, Arabic, the Sámi languages) contributes nothing — see `KeyboardLocale.topLevelDomain` —
and the bar is capped at `DomainKey.maxCallouts` so it stays reachable with one thumb.

The key never commits on key-down, even with `insertOnKeyDown` on: a multi-character key isn't typed fast
or by accident, and committing it early made its own long-press unusable (ENG-156).

## 18b. Accessibility (VoiceOver)

Every key is published as **one** accessibility element carrying:

* a **name** — the character in its currently displayed case, or `space` / `delete` / `shift` /
  `return` / `letters` / `next keyboard` / `dictate` for the control keys (iOS substitutes its own
  canonical name for some symbol keys, e.g. a delete key reads as "Backspace" — that is the name a
  VoiceOver user already knows);
* the **`.isKeyboardKey` trait**, so iOS applies VoiceOver's keyboard typing modes;
* an **activation action** that types the key directly. Assistive activation delivers no touches, so
  neither input pipeline (the per-key gesture or the multi-touch layer) would otherwise see anything at
  all.

Nothing is required of a host for this — it comes with `PressboardKeyboardView`. If you pass a
`keyOverlay` (e.g. a language badge on the space bar), it is hidden from assistive tech automatically so
it can't turn the key into a container and swallow its name.

The names and the one-element-per-key shape are guarded in the real, out-of-process extension by
`DemoAppUITests/KeyboardAccessibilityUITests`. VoiceOver's own gesture handling is not something
XCUITest can drive, so the interaction itself still wants a manual pass with VoiceOver switched on.

## 19. UI testing (XCUITest harness)

`DemoApp/DemoAppUITests` is a real XCUITest target that taps **exact coordinates** in the key grid
and reads back what got typed — built for ENG-121 to answer a question unit tests can't: "does a
touch at this precise point on screen actually type the right character, in the real, out-of-process
keyboard extension a user actually types in?" Prior to it, the only way to check that was manual
taps on a simulator/device, which doesn't scale into a regression suite.

**Run it:**

```bash
cd DemoApp && xcodegen generate   # after any project.yml change; skip otherwise
xcodebuild test \
  -project PressboardKitDemo.xcodeproj -scheme DemoApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro'
```

**What it covers** (see the source for exact assertions — this is the map):
- `KeyboardTestGeometry` — the reusable helper. Looks up a key's on-screen `CGRect` by the stable
  identifier every `KeyboardButton` carries (`key.character.<lowercased letter>`, `key.shift`,
  `key.space`, …; see `KeyboardButton.accessibilityIdentifier` — free for your own app's UI tests
  too), then taps absolute points derived from those **real, rendered** rects — not a second,
  parallel re-implementation of `KeyHitTestGeometry`'s math, which could drift from the engine and
  prove nothing about what's actually on screen.
- `DeadZoneAndSeamUITests` — taps the dead-centre of the gap between two adjacent letter keys, and
  sweeps across the seam at ~2pt steps, asserting every point types something (no dead zone) and the
  sequence is monotonic (all of one key, then all of the other) — against **DemoApp's in-process
  live preview**.
- `KeyboardExtensionUITests` — the identical two checks against the **real, out-of-process keyboard
  extension**, switched to via another app (Reminders' quick-entry title field). This is the one
  that matters for parity: a fix that closes a dead zone in-process is not proven until it's
  measured here too — see the file's header comment for a real regression (bugfix/ENG-120) that
  passed in-process and failed here, plus the "stale extension" trap to watch for when iterating
  (a keyboard extension isn't always automatically reinstalled just because its source changed —
  `xcrun simctl uninstall <device> com.pressboardkit.demo` before rebuilding if a fix doesn't seem
  to take effect here).
- `PredictiveKeyTargetingUITests` — end-to-end version of the measured `predictiveKeyTargeting`
  (ENG-119) case: typing "keyboar" then tapping just inside "s", close to its boundary with "d",
  resolves to "d"; the identical tap after "qqqqqqq" resolves to the geometric "s".
- `GestureTimingThresholdUITests` (ENG-122) — measures (not asserts) the long-press callout threshold
  and the slide-to-type glide-engage distance against the **real native keyboard**, in the same host
  app as our own real extension (`RemindersKeyboardHarness`), so the two are directly comparable.
  Slow/exploratory — a binary search per value, screenshot-diffing mid-hold for the timing one via a
  `Timer` on the main run loop (confirmed empirically that `press(forDuration:)` pumps it while
  waiting) — not meant to run on every commit; re-run on demand when re-tuning `NativeMetrics.
  calloutLongPressThresholdNanoseconds` / `KeyTouchStateMachine.glideAbandon`/`glideEngageKeyCount`.
- `SlideToTypeEngageUITests` — the fast, permanent regression guard for the distance side of the
  above: a short drag still types one character (tap), a drag past the tuned engage distance doesn't
  — against the real extension.

**Requires the "PressboardKit Demo" keyboard enabled** for `KeyboardExtensionUITests` (Settings →
General → Keyboard → Keyboards, see `DemoApp/README.md`) — that test class re-registers it
automatically at the start of its run, but the extension must be installable at all (i.e. `DemoApp`
built at least once). If it can't reach the keyboard, it `XCTSkip`s rather than fails.

**Adapting this for your own app:** the same two ingredients — accessibility identifiers on your
keys (free from `KeyboardButton`) plus `XCUIApplication(bundleIdentifier:)` pointed at whatever host
app you type into — work in your own XCUITest target, no PressboardKit-specific tooling required.
`KeyboardTestGeometry` is plain `XCTest` + `CoreGraphics`; copy it in.

---

### Keeping this guide current

This is the integrator-facing contract. When a **user-visible feature or a public-API change**
lands, update this guide alongside `FEATURES.md` and `CHANGELOG.md` — a new `KeyboardBehavior`
field gets a row in [§17](#17-full-behavior-reference) and, if it needs host wiring, its own
short section. That keeps "how to use the latest feature" always answered here.
