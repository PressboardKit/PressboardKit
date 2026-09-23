# PressboardKit

A self-contained **iOS keyboard engine** in Swift — layout rendering, callouts,
shift/caps, spacebar-drag cursor, feedback, and pluggable autocomplete — with **no
dependency on any host app**. Built to replace the commercial KeyboardKit inside the
Rewordo app, and designed so it can later be offered as a standalone product.

## Primary goal — indistinguishable from the native iOS keyboard
The keyboard **must look and behave exactly like the native iOS keyboard**: same visuals,
same feel, same typing experience — users must not be able to tell the difference. This is
the overriding "done" criterion for every visual/interaction feature and takes precedence
over custom styling. See [`DESIGN.md`](DESIGN.md) for the concrete parity checklist
(metrics, timing, key-pop, callouts, sounds, haptics, autocorrect behavior).

> Status: **early scaffold (v0.0.1)**. Engine implementation lands in phase B — see
> [`PLAN.md`](PLAN.md) and [`TODO-ENGINE.md`](TODO-ENGINE.md).

## Why
Reduce reliance on a third-party keyboard SDK, own the roadmap, and keep the door open to
selling the engine. The public API is host-agnostic from day one (see [`DESIGN.md`](DESIGN.md)).

## Modules
| Product | Purpose |
|---|---|
| `PressboardKit` | Core engine: rendering, layout state, callouts, shift/caps, spacebar drag, feedback |
| `PressboardKitLayouts` | Data-driven layout tables (CZ/EN) + locale mapping; add a language = add a table |
| `PressboardKitAutocomplete` | `SuggestionProvider` + dictionary-based suggestions (optional; memory-friendly) |
| `PressboardKitEmoji` | Emoji keyboard panel — catalog, skin tones, recents, UIKit grid (optional) |

## Requirements
- iOS 26+, Swift 6, Swift Package Manager.

## Integration
The engine is a SwiftUI render view (`PressboardKeyboardView`) plus pure text-logic helpers;
your `UIInputViewController` owns the text document and wires the two together. A minimal working
keyboard:

```swift
import PressboardKit
import PressboardKitLayouts

let layout = StandardLayoutResolver.layout(for: context)   // context: KeyboardContext
PressboardKeyboardView(
    layout: layout,
    style: MyStyleProvider(),          // your theme; defaults to NativeKeyboardStyle()
    behavior: .standard,               // native defaults; flip any toggle to opt out
    onAction: handle                   // route .character / .backspace / .shift / …
)
```

Everything host-specific (theme, toolbar UI, suggestion source, settings) is injected; the kit
never imports host code. **See [`INTEGRATION.md`](INTEGRATION.md) for the full, detailed guide**
— quick start, `KeyboardAction` routing, the complete `KeyboardBehavior` reference, Liquid Glass
transparency, styling, autocomplete/emoji/slide-to-type, and App Group setup.

## Build
```bash
swift build
swift test
```
Visual/interactive behavior is exercised in `DemoApp/` (a standalone Xcode project with its
own keyboard extension) on the iPhone 17 Pro simulator.

## Docs
- [`INTEGRATION.md`](INTEGRATION.md) — **how to embed the kit in your app** (detailed guide)
- [`PLAN.md`](PLAN.md) — roadmap & decisions
- [`DESIGN.md`](DESIGN.md) — API design & the rules that keep productization additive
- [`TODO-ENGINE.md`](TODO-ENGINE.md) — task backlog (`ENG-N`)
- [`FEATURES.md`](FEATURES.md) · [`CHANGELOG.md`](CHANGELOG.md)

## Licence
PressboardKit is a commercial SDK. It is **free while it is in early access** — deliberately
temporary, not a permanent free tier — and it is licensed rather than given away: you register
and a licence is issued to you, at no charge for now. Terms: [`LICENSE`](LICENSE), which is
still a draft awaiting a lawyer.

The bundled word-frequency lists are MIT-licensed and come from Hermit Dave's FrequencyWords;
provenance and the full notice are in
[`Sources/PressboardKitAutocomplete/Resources/CREDITS.txt`](Sources/PressboardKitAutocomplete/Resources/CREDITS.txt).
