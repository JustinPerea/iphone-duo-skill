---
name: iphone-duo-converter
description: Convert, audit or test an iPhone app (SwiftUI, UIKit, storyboard, React Native, Flutter, Unity, games, web views) for Apple's foldable iPhone Duo (a closed cover screen and a larger open inner screen; folding, partly folded, unfolding; held wide or turned tall). Use when someone asks to convert, adapt, audit or test an app for iPhone Duo or a foldable iPhone, try the Duo on a fresh app, debug a view that breaks, blanks, loses state or gets cut off on the cover or inner screen, handle fold and unfold mid-task, use the fold (reserved regions, division, occlusion), ArrangementView, the hinge, the side column or vertical toolbars, Duo cameras, run or script the Duo simulator in Device Hub, write Duo UI or fold tests, or prepare iPhone Duo App Store screenshots and release. Carries Apple's HIG, docs, tech talks and group labs with sources, plus an edge-case catalog (unmeasured items marked unverified) and a fresh-app test plan.
---

# iPhone Duo converter

Built from the Duo work on a shipping SwiftUI app (Mitamaton: version 1.1.0, then a 1.1.1 audit and review, October 2026), checked against Apple's own guidance, then widened with verified edge cases for other app types.

**Start at section 1 (Quick audit). Section 0 is reference: come back to it when a hit needs a fact.**

Files:
- `references/apple-guidance.md`: Apple's docs, HIG, six tech talks, four group labs and store pages, a source on every claim. Read before choosing an approach.
- `references/edge-cases.md`: the edge-case catalog by area, each with detect, fix, verify, source. IDs (E-LIFE-1, E-PRES-3, ...) are cited below.
- `references/verification.md`: simulator, Device Hub, capture, test and CI detail.
- `references/fresh-app-test-plan.md`: step-by-step trial on an unfamiliar app, with a results template.
- `references/mitamaton-case-study.md`: a worked example from a shipping SwiftUI app: what broke on the Duo, how it was found, the fix and the measured numbers.

Source keys in brackets ([HIG], [AV], [RR], [PDUO], [TT463 12:41], ...) are defined in the Sources table at the end of `references/apple-guidance.md`.

Code marked **(shipped)** compiled and ran in Mitamaton under Xcode 27.1. **(not compile-verified)** means build it before trusting it. **(unverified)** means nobody has measured it on a Duo yet.

## 0. Facts to hold

- Two displays. Cover: 1398 x 2034 px = 466 x 678 pt at 3x. Inner: 2007 x 2853 px = 669 x 951 pt; held wide (the default open pose) it measured 951 x 669 pt, safe area top 0, 84 on the side-column edge, bottom 34. The cover is wider (466) and shorter (678) than most iPhones, aspect about 0.69.
- The CLOSED cover also has a side column: status bar, toolbar buttons and tab bar sit in it and content runs to the cover's top edge. Turned tall (open or partly open turned), the column goes away: buttons return to a top bar, the tab bar to the foot.
- The column follows the hardware, not "trailing": after a cover rotation it can be on the LEFT, and in Split View the left app's column is on the left. The HIG says it stays "on the same side in right-to-left languages", but the `toolbarVerticalEdge` page says "the system determines the edge based on locale and device", and no RTL run has been measured: treat RTL as unmeasured. Always read `toolbarVerticalEdge` (SwiftUI) or `verticalBarEdge` (UIKit) and the safe area; never assume the 84 pt is on the right.
- Size classes: cover portrait compact x regular, cover landscape compact x compact, inner regular x regular with the PHONE idiom. Code that treats regular width as "iPad", or idiom as "compact", is wrong here (E-LAYOUT-1).
- The inner display ignores supported orientations; the cover honours them (a portrait-only app reads sideways on the turned cover). `UIRequiresFullScreen` is honoured, but the app still resizes on fold and "will scale on the inner display, including in Split View" [TT461 4:37]. Lay out by size, never orientation.
- SDK ladder [PDUO]: 26 SDK or earlier runs centred with empty space open, and left of the status bar and camera closed; 27 SDK resizes to fill most of the inner display "but still avoid[s] the status bar on the right edge"; **27.1 SDK** uses the full display, with toolbars and tab bars vertical below the status bar. The rebuild is the switch; once linked on the 27 SDK there is no opt-out of resizing [LAB0916 47:12]. Per-screen opt-out of the vertical bar: `toolbarVerticalBehavior(.disabled)` (UIKit `preferredVerticalBarBehavior`), a stable choice.
- Building with the iOS 27 SDK requires the UIScene lifecycle, or the app will not launch [TN3187]. SwiftUI `App` apps already have it; old UIKit, React Native/Expo, older Flutter and Unity hosts may not (E-LIFE-1).
- `UIScreen.main` is deprecated since iOS 26.0 (reference page; TT461 still says "will be"). On the open Duo it returns the cover.
- Duo APIs (`reservedRegions`, `axisBehavior`, `ArrangementView`, `UIHingeInteraction`) are iOS 27.1 SDK symbols: Xcode 27.0 fails to compile them (exit 65). Build with Xcode 27.1 final (27A9275, numbered in Apple's seed range) and gate each call with `#available(iOS 27.1, *)`. Never raise the deployment target to avoid the gates (E-STORE-11).
- Simulator: the iOS 27.1 runtime's only device type is iPhone Duo (model id `iPhone19,4`, Touch ID, not Face ID). A plain-iPhone control run must use iOS 27.0, which only runs your fallback branches.
- Tool ladder (Apple's): system containers and presentations, then `ArrangementView` / `UIArrangementViewController`, then `reservedRegions`, then the hinge (`onHingeChange`, `UIHingeInteraction`) only for angle-driven features. Never lay out from the hinge angle.
- In one trial's simulator, reserved regions never arrived. With Xcode 27.1 27A9275, the iOS 27.1 runtime 24A94425 and Device Hub, `reservedRegions` returned `[]` for `.division` and `.occlusion`, with and without `.includeInactive`, from `GeometryProxy` (a page and the window root) and from `UIView` (a view and its window), in every pose, while `UIHingeInteraction` reported the hinge partly open (2.23 rad, status 2). A layout driven only by reserved regions showed no change there. Log the regions and the hinge together before trusting either, and give fold-driven layouts a fallback (T2).
- Multiple instances: an app that supports multiple windows on iPad runs two instances side by side on the Duo; both are foreground active and share `@AppStorage`. The cover cannot create new windows.
- Apple's pages: HIG `.../human-interface-guidelines/designing-for-iphone-duo` (not `.../iphone-duo`, which 404s); article `.../technologyoverviews/preparing-your-app-for-iphone-duo`; Prepare page `developer.apple.com/iphone-duo/prepare/`. Tech talks: 111461 prepare, 111462 bars, 111463 poses and layout, 111464 displays, scenes and hinge, 111465 camera, 111466 design. Four group labs (Apple Developer "iPhone Duo Group Lab" videos, 9/16, 9/17, 10/13 and 10/15/26; links in `references/apple-guidance.md`).

## 1. Quick audit (run first on any unknown app, read every hit)

Step 0, inventory. List every target (`xcodebuild -list`; `rg -l NSExtensionPointIdentifier -g '*.plist'`), every language present (`.m`, `.mm`, `.cpp`, `.js/.ts`, `.dart`, `.cs`, `.html/.css`), every local package and vendored binary, and the lifecycle (`@main struct ...: App` or a scene manifest). Pick the archetypes in section 2. Mitamaton's first audit scanned only the app's `Sources` and missed the `UIScreen.main` in its own package, which blanked the main character.

Do NOT add `--type swift`: it returns zero on Objective-C, cross-platform and web code and looks like a pass (E-TEST-14). Fixed-string search for literal needles; count before reading; never pipe an absence check through `head`.

```bash
rg -n -e 'UIScreen\.main' -e 'mainScreen\]' -e 'UIScreen\.screens' -e '\.nativeBounds' .
rg -n -e 'userInterfaceIdiom' -e 'UIUserInterfaceIdiomPad' -e '\bisPad\b' -e 'NavigationView' -e 'horizontalSizeClass' .
rg -n -e 'supportedInterfaceOrientations' -e 'UIInterfaceOrientation' -e 'requestGeometryUpdate' -e 'interfaceOrientation' -e 'UIDevice\.current\.orientation' -e 'forKey: *"orientation"' -e 'statusBarOrientation' -e '\.isLandscape' -e '\.isPortrait' .
rg -n -e 'utsname' -e 'iPhone[0-9]+,[0-9]' -e 'height *[<>=]=? *[5-9][0-9]{2}' -e 'isSmallScreen' -e 'Face ID' -e '\.faceID' .
rg -n -e '\.frame\(width: *[0-9]' -e 'padding\(\.(leading|trailing|horizontal), *[0-9]' -e 'width \* 0\.[0-9]' .
rg -n -e 'tabBarHeight' -e 'barHeight' -e 'safeAreaInset\(edge: *\.(top|bottom)' -e 'safeAreaBar' -e '\.clipped\(\)' .
rg -n -e '\.sheet\(' -e 'fullScreenCover' -e 'popover' -e 'actionSheet' -e 'UIActivityViewController' -e 'confirmationDialog' .
rg -n -e 'Environment\(\\\.dismiss\)' -e 'ArrangementView' -e 'NavigationSplitView' -e 'onDismiss:' .
rg -n -e 'ToolbarItem' -e 'UIToolbar' -e 'UITabBar' -e 'UINavigationBar\(' -e 'TabView' -e 'inputAccessoryView' .
rg -n -e 'if .*(SizeClass|isSplit|isWide|isCompact)' -e '\.id\(.*(size|width)' .        # branch swaps lose @State
rg -n -e 'connectedScenes' -e 'keyWindow' -e 'windows\.first' -e 'openWindow' -e 'requestSceneSessionActivation' .
rg -n -e 'keyboardFrameEndUserInfoKey' -e 'keyboardWillShow' -e 'keyboardWillChangeFrame' .
rg -n -e 'scenePhase' -e 'willResignActive' -e 'didEnterBackground' -e '@SceneStorage' .
rg -n -e 'frame\(in: *\.global' -e 'convert\(.*to: *nil' -e 'onHingeChange' -e 'UIHingeInteraction' .
rg -n -e 'accessibilityFocused' -e 'ImageRenderer' -e 'dynamicTypeSize\(\.\.\.' -e 'Font\.custom\(' .
rg -n -e 'drawableSize' -e 'contentsScale' -e 'scaleMode' -e 'CAMetalLayer' -e 'SKScene\(size' .
rg -n -e 'WKWebView' -e 'viewport-fit' -e 'safe-area-inset' -e "Dimensions\.get\('screen'\)" -e 'MediaQuery' -e 'Screen\.width' .
rg -n -e 'AVPlayer' -e 'AVCaptureVideoPreviewLayer' -e 'rectOfInterest' -e 'UIImagePickerController' -e 'CMMotionManager' .
```

Vendored pass. `rg` honours `.gitignore`, so ignored `Pods/`, `node_modules/` and SPM checkouts are silently skipped by the greps above (E-TEST-14, E-TEST-27). Rerun the high-yield needles with `--no-ignore --hidden` (hidden dirs like `.build/` need both) over every vendor dir, and list binaries:
```bash
DD=<the -derivedDataPath you build with, or ~/Library/Developer/Xcode/DerivedData/<App>-<hash>>
for d in Pods node_modules Carthage vendor Vendor ThirdParty .build/checkouts ios/.symlinks "$DD/SourcePackages/checkouts"; do
  [ -d "$d" ] && { echo "== searched $d"; rg -n --no-ignore --hidden -c -e 'UIScreen\.main' -e 'mainScreen\]' -e '\.nativeBounds' -e 'userInterfaceIdiom' -e 'UIInterfaceOrientation' -e 'keyWindow' "$d"; }
done   # only "== searched" lines prove a dir was read; resolve SPM packages first or the checkouts dir does not exist
find . "$DD/SourcePackages" \( -name '*.xcframework' -o -name '*.framework' -o -name '*.a' \) -prune -print   # each one: nm -u
```

Also check binaries (`nm -u <binary> | grep -c UIScreen`), storyboards and asset catalogs for regular-width variants (`rg -c 'widthClass="regular"' -g '*.storyboard' -g '*.xib'`), and after archiving, the linked SDK in the app and every appex (`plutil -p .../Info.plist | grep -E 'DTSDKName|DTXcodeBuild|MinimumOSVersion'`).

What hits mean: `UIScreen.main` geometry is the cover (T1). Idiom and size-class branches take the iPad path on a phone. Fixed widths miss asymmetric insets. Hand-built bars (and every React Native/Flutter-drawn bar) never go vertical. Bar-height constants are wrong when the nav bar is 0 pt or the tab bar is in the column. Branch-swapped containers lose state on fold. Global window lookups grab the other instance. Keyboard notification frames are in screen space. `.global` frames captured once go stale on fold. Full lookup table per hit: `references/edge-cases.md`.

## 2. Archetype routing

| App has | Read these edge cases first |
|---|---|
| No scene lifecycle (old UIKit, RN/Expo, Flutter < 3.38, Unity) | E-LIFE-1 (launch blocker) |
| Storyboards / xibs, Auto Layout, iPad variants | E-LAYOUT-1, E-LAYOUT-2, E-LAYOUT-3 |
| List + detail, editors, forms, inspectors, tips | E-PRES-1 to E-PRES-7, E-PRES-10, E-PRES-11, E-INPUT-1 to E-INPUT-3 |
| Paywall, IAP, sign-in, account deletion | E-LAYOUT-9, E-STORE-10, E-LIFE-5 |
| Custom chrome, drawers, mini-player, sticky CTAs, toasts, status bar | E-BARS-1 to E-BARS-11 |
| Games, Metal, SpriteKit, Unity | E-ARCH-1 to E-ARCH-3 |
| Video, audio, PiP | E-ARCH-4, E-ARCH-5 |
| Camera, scanner | E-ARCH-6, E-PRES-8 |
| Web views, hybrid apps, cross-platform UI (incl. Compose MP, Skip, Capacitor, MAUI) | E-ARCH-7, E-ARCH-8, E-ARCH-12 |
| Maps, drawing, motion sensors | E-ARCH-9 to E-ARCH-11 |
| Extensions, widgets, Live Activities, App Clips, controls, snippets | E-LIFE-8, E-LIFE-9, E-LIFE-14 |
| Multiple windows / documents, drag and drop, pasteboard | E-LIFE-3, E-LIFE-4, E-INPUT-4 |
| PDF, QuickLook, document readers | E-LAYOUT-14 |
| Calls, alarms, SharePlay, external display / AirPlay | E-LIFE-13, E-LIFE-15 |
| Keyboard shortcuts, pointer, assistive input, display settings | E-INPUT-5, E-A11Y-5, E-A11Y-6 |
| Long or localized copy, Display Zoom | E-LAYOUT-12, E-LAYOUT-13 |
| Also runs as Designed for iPhone on Mac / visionOS | E-LIFE-16 |
| Vendored code (Pods, node_modules, SPM, binaries) | E-TEST-14, E-TEST-27 |

## 3. Conversion playbook (ordered by impact)

1. **Lifecycle and toolchain.** Adopt the scene lifecycle if missing (TN3187). Xcode 27.1 final; fix release guards that call it a beta or fall back to an older Xcode; add a post-archive gate that refuses a DTSDKName below iphoneos27.1 (E-STORE-1). Expect a visible change on every screen; record before/after captures, including paywall, sign-in, permission and account screens nobody edited.
2. **Kill `UIScreen.main`.** Measure the view's own window or `GeometryReader`/`onGeometryChange` size; scale from `traitCollection.displayScale`; drawables from bounds x displayScale. Guard test against EVERY `UIScreen.main` member (T1).
3. **Use system bars.** Only container-owned bars move to the side column. Give each item a symbol and a title. Items whose text carries information (a cart total, a price) stay horizontal on purpose: do not force `.verticalPreferred` on them. Order: Back/Close on top (`.cancellationAction`), prominent action (`.topBarPinnedTrailing`), then groups. Set `visibilityPriority` (the keyboard and PiP force overflow), use `ToolbarOverflowMenu`, pick `toolbarVerticalCompressionBehavior` per screen. Cross-platform apps: native containers, or a bridge exposing the vertical edge and bar region (E-ARCH-8).
4. **Measure chrome.** Bar heights from the measured safe area (T4); bottom reservations from the bottom safe area or `toolbarVerticalEdge`, not a tab-bar constant (E-BARS-6); corner-adapted inset at the top (T5); bottom rows respect the horizontal safe area (E-BARS-5). Toasts and banners inset by both side insets and stay off `.occlusion` regions (E-BARS-10); pick the status bar style for what lies under the column (E-BARS-11).
5. **Read the fold.** `reservedRegions(kind: .division, options: .includeInactive)` (T2). Even grid columns. Keep the layout steady as the hinge moves: "Avoid extreme layout changes as people fold the device. Move only what's necessary to keep elements visible and easy to tap." [HIG]. Below a minimum side width do not split (Mitamaton used 343 pt, its narrowest iPhone board; case study section 6). Lists and scroll views do not split at the fold by themselves [RR]. Content scrolling parallel to the fold moves off it or changes axis (Preview pages horizontally in book pose); content scrolling across it may run through; continuous reading content is not displaced. Give readers and editors an explicit book-pose policy. With nothing selected, hold headers, summary lines and lists to one lane and put centred single-column screens (sign-in, primes, chat) on one side of the fold (E-LAYOUT-7).
6. **Use the second half.** Details open beside their source on the open Duo, sheets elsewhere. Keep navigation outside the split. `NavigationSplitView` keeps a detail column even when empty and adapts its widths to the fold (about 35/65 flat, 50/50 folded [LAB1013 47:39]). `ArrangementView`: wrap it in the `NavigationStack`; never put it inside a scroll view, grid or `NavigationSplitView`, and never put a `NavigationSplitView` or `TabView` inside it [AV]. `.split` is one tree only while both views are in it: on a closed Duo held in landscape it shows only the primary [AV], it stacks on the cover held tall, and a style restricted to an axis it cannot use shows one view [TT463 12:41], so a secondary detail can leave the tree and lose `@State` on fold. When a detail must survive the cover, use a one-tree host: one overlay whose FRAME changes (beside / fullScreen / closed), `.sheet` kept attached and bound to `.constant(nil)` while in-tree (T12, E-PRES-1). Empty detail: `ContentUnavailableView`; do not auto-select things with side effects.
7. **In-tree panes need sheet parity.** Close through an environment action, not `dismiss()` (a no-op off-presentation); run `onDismiss` on any clear of the item; guard re-taps and let a detail refuse closing; scope "pane open" per surface; one Close, not two; VoiceOver container, escape, focus in and back; Reduce Motion fade with its own animation (T13, E-PRES-2 to E-PRES-6, E-A11Y-2).
8. **Presentations survive folds.** Prefer system sheets, alerts, menus, popovers (they reposition). `presentationPlacement(.leading/.trailing)` side-anchors a sheet (iOS 27.0). Always anchor action sheets and popovers (`sourceItem`) whatever the idiom: popover adaptation follows the size class, so regular width might present them as real popovers (unverified on the Duo; E-PRES-9). Camera pickers go `.fullScreenCover`. Check `.inspector` and TipKit popovers across the fold (E-PRES-10, E-PRES-11). Keep custom panels OPEN through a fold and re-place them from the current screen; closing is only a fallback (T6). One view tree across arrangements (T7).
9. **Scroll edges and the band.** Apple's soft edge (T8). On the cover, custom top chrome needs a `safeAreaBar(edge: .top)` whose content DRAWS something (a 0.001 fill), and the band must swallow taps (T14).
10. **Text and accessibility.** Check copy at 466 pt and AX sizes on the cover and in turned halves. Lift Dynamic Type caps on detail, reader and editor text; cap line length with the readable content guide (E-A11Y-4). Test Reduce Transparency (it puts an opaque plate behind the side column [LAB0916 30:27]), Reduce Motion, RTL, VoiceOver order across the fold; Voice Control and Switch Control on column items (E-A11Y-5); Bold Text, Button Shapes, Increase Contrast, Dark Mode (E-A11Y-6); long strings and Display Zoom on the cover (E-LAYOUT-13, E-LAYOUT-12).
11. **Keyboard, scenes, phase.** `keyboardLayoutGuide` or the keyboard safe area, never cached screen-space frames (E-INPUT-1). Accessory controls stay on the keyboard. Keyboard shortcuts, pointer and Full Keyboard Access (E-INPUT-5); drag and drop between instances (E-INPUT-4). Get windows from the view, not `connectedScenes.first` (E-LIFE-3). End work only on `.background`; treat `.inactive` as transient; adopt `@SceneStorage` (E-LIFE-6).
12. **Cameras.** Virtual front camera follows the display; direction coordinator for physical cameras; mirror from the direction map; rotation coordinator; recompute preview-dependent rects on resize. Outer-display content only with an active capture session (`apple-guidance.md` section 9).

## 4. Traps and fixes

**T1. `UIScreen.main` is the cover.** A view blanks or stops animating right of x 466 or below 678 on the open screen. Test against the view's own window. (shipped)
```swift
final class WindowBox { weak var window: UIWindow?
  func contains(_ f: CGRect) -> Bool {
    guard f.width > 0, f.height > 0 else { return false }
    guard let window else { return true }      // unknown window counts as visible
    return window.bounds.intersects(f) } }     // f from proxy.frame(in: .global)
// A zero-size UIViewRepresentable records `window` in didMoveToWindow() and updateUIView.
// Zero size so ImageRenderer captures have nothing to paint.
```
One-off height: read it from the CALLING view's window (`view.window?.bounds.height`). Use the foreground-active scene only when there is no view: with two instances both are foreground active (E-LIFE-3). Guard test: scan every source root (app, packages, extensions, every language) for any `UIScreen.main` read, ignore comments, fail on a hit and on a suspiciously small file count.

**T2. Reading the fold.** (shipped, iOS 27.1; vertical regions only, as the board splits side by side; Mitamaton's onboarding resolver also stacks on a horizontal region.)
```swift
content.onGeometryChange(for: Fold?.self) { proxy in
  proxy.reservedRegions(kind: .division, options: .includeInactive)
    .map { ($0.frame, $0.isActive) }
    .first { $0.0.height > $0.0.width && $0.0.midX > 0 && $0.0.midX < proxy.size.width }
    .map { Fold(midX: $0.0.midX, width: $0.0.width, isBent: $0.1) }
} action: { fold = $0 }
```
Vertical region = held wide (split); horizontal = turned tall (stack); none = cover or other iPhone. Bent: `isActive`, band measured 40 pt (margins included). Frames are in the view's own space, may be negative or past its size, and are mirrored in RTL (`.fixed` for raw). Test pure functions with offset, partly-outside, RTL, leading-column and inactive-width-0 inputs (E-TEST-9).
- A default query returns ACTIVE regions only, so a flat open Duo returns no division without `.includeInactive` [RR]. The SwiftUI method is `reservedRegions(kind:options:layoutDirectionBehavior:)`; UIKit drops the last parameter.
- Fallback when no region arrives (see section 0): for a regular-width window that is wider than tall (the inner display held wide), use the window's centre as a zero-width fold, which is where the hinge is; a reported region always wins. Measure the window at the root with the measurement inside the safe-area escape, `Color.clear.onGeometryChange(...).ignoresSafeArea()`: with the order reversed the size stopped at the safe area (867 x 635 held wide, not 951 x 669) and the stand-in fold landed 42 pt left of the real one. Recompute whenever the page geometry or the window size changes: a fold can update them in separate passes, and lanes resolved inside one `onGeometryChange` transform from a window size captured earlier stayed stale after unfolding with the app open (trial, 2026-10-24).
- `.occlusion` regions use the same API: the Dynamic Island, the outer camera always, the inner camera only while active (and iPad window controls) [RR]. Keep floating controls off the inner camera when it is on.

**T3. Buttons into the side column.** (shipped) `.toolbar { if #available(iOS 27.1, *) { items.axisBehavior(.verticalPreferred) } else { items } }`. Needed for custom views and when both bars exist; symbol-plus-title items go vertical without it; other iPhones stay horizontal, so no device check is needed (the `#available` check is). Also in Apple's docs, not used in Mitamaton: `.axisBehavior(.horizontalOnly)`, `.toolbarVerticalBehavior(.disabled)` (27.1), `.toolbarVerticalCompressionBehavior(.prefersToolbarItems)` (27.1), `.visibilityPriority(.high)` (27.0), `ToolbarOverflowMenu` (27.0), `toolbarVerticalEdge` (27.1); UIKit `UIBarButtonItem.axisBehavior`, `visibilityPriority`, `UINavigationItem.additionalOverflowItems`, `verticalBarCompressionBehavior`, `UITraitCollection.verticalBarEdge` (`references/apple-guidance.md` section 3).

**T4. The navigation bar takes no room.** With buttons in the side column a shown nav bar is 0 pt, so a constant 54 pt offset lands content too high. Measure it: a probe `UIView` in the content reports
```swift
let navBar = probe.convert(CGPoint.zero, to: nil).y + probe.safeAreaInsets.top - window.safeAreaInsets.top
```
The probe FILLS its surface and re-measures in `didMoveToWindow`, `layoutSubviews`, `safeAreaInsetsDidChange` and on `verticalBarEdge` changes (a zero-size probe measured only when other layout reached it). Never write observed state inside a layout pass; dispatch to main. (shipped)

**T5. Content touches the rounded corner.** Safe area top is 0 open; drop leading top content by the corner-adapted inset, not a Duo constant (16 pt there, 0 on iPhones). Anything anchored under it adds the same drop. (shipped, iOS 26+)
```swift
let adapted = window.edgeInsets(for: .safeArea(cornerAdaptation: .vertical))
let corner = max(0, (adapted.top - window.safeAreaInsets.top).rounded())
```

**T6. Custom panels across a fold.** A panel laid out from a frame measured at tap time used the other screen after a fold (858 pt on the 466 pt cover). Apple repositions its own menus and popovers [TT463 5:23]; do the same: keep the panel open, re-place it from the side column measured on the CURRENT screen (`UIView.LayoutRegion.bar(onEdge:extent:)`, shipped in Mitamaton), drop the tapped frame on resize, and record the trigger's slot in the column order on open (cleared on every open) so it collapses into its own button. (shipped) Branch on where the trigger actually is: a panel that hung from a top button was cut to a mid-screen strip once the button moved 174 pt down the side column (case study section 8). Fallback, only when nothing can re-measure: close on resize and use a frame-free layout. (shipped in 1.1.0)
```swift
.onChange(of: geo.size) { old, new in
  if presented && (abs(old.width - new.width) > 1 || abs(old.height - new.height) > 1) { dismiss() } }
```

**T7. State lost across a fold.** `if split { HStack } else { VStack }` rebuilds children and resets `@State` (Mitamaton lost an email sheet, a signing-in flag, an Apple nonce). One tree with `AnyLayout`; split-only children are optional EARLIER siblings. (shipped)
```swift
let layout = split?.axis == .sideBySide ? AnyLayout(HStackLayout()) : AnyLayout(VStackLayout())
layout { if let split { hero.frame(...) }; column }   // `column` keeps its identity
```
Same idea for details: keep both the pane path and the `.sheet` path attached (sheet bound to `.constant(nil)` while in-tree), so folding mid-detail hands it across (T12).

**T8. Scroll edges and clipping.** `.clipped()` after `.ignoresSafeArea(.container, edges: .bottom)` still cuts at the safe-area foot. An edit-mode mask can cut at the same line: open both. With the default style the blur covered only the status-bar strip. (shipped, iOS 26+)
```swift
scroll.ignoresSafeArea(.container, edges: .bottom)
  .clipShape(OpenEnds())                                   // a rect extended past top and foot
  .scrollEdgeEffectStyle(.soft, for: [.top, .bottom])
```
Caveats: verified on an iPhone 17 Pro simulator only (light and dark, at rest and scrolled), never captured on the open Duo. On the closed cover no bar takes room, so content passes sharp under custom top chrome; that is system behaviour, and the HIG allows full-width tops only for non-scrolling UIs, with scrollable content kept inset. The fix there is T14. Standard navigation bars also minimize on scroll by default on the Duo (UIKit `navigationBarMinimization`). Choose a scroll-edge style by measurement: one probe build with a DEBUG launch switch per variant, captured scrolled under both bars, then remove the probe.

**T9. Horizontal scrollers and bottom rows under the column.** Clip horizontal scroll views to their own frame. Bottom formatting, attachment and action rows ignore the horizontal inset and land under the column: respect it, or fold controls into one menu (E-BARS-5).

**T10. Layout feedback loops.** A lift measured from a frame that includes the lift went black at 99% CPU. Measure the unmoved position inside the geometry transform; ignore changes under 0.5 pt. Quantise hinge-driven values; never assume an update rate.

**T11. Standalone system bar.** When a flow cannot live in a `NavigationStack` (Mitamaton: anchor preferences do not cross its hosting boundary; a nested tab root), wrap `UINavigationBar` in a `UIViewRepresentable` with one `UINavigationItem`, a transparent appearance and a `UIBarButtonItem` Back. Use a `titleView` label with `adjustsFontSizeToFitWidth` and a capped scaled font (Mitamaton capped at 20 pt; case study section 13) so accessibility sizes fit. This gives Apple's look, not Apple's placement: hand-built bar content "will never go vertical" [TT462; LAB1013]. Prefer a real container; otherwise place the custom bar from `toolbarVerticalEdge` and the system bar region. If you must ship it, gate the pinned bar to layouts with a side column or split, or plain iPhones lose about 44 pt and Close moves; name any iPhone change as a broadening (E-BARS-7). (shipped on 1.1.1 onboarding)

**T12. One-tree detail host.** Each presenter hands its detail up as a preference; one overlay draws it with only its frame changing: beside (vertical fold), fullScreen (cover or turned tall, with the side column), closed. Decide "is a Duo" from `UIHingeInteraction` (`update.hinge` was non-nil in every pose, closed included, in Mitamaton) but it reports ASYNC after first layout: never gate view identity on it; gate on something fixed per host (T15). Other iPhones keep `.sheet`. (shipped in 1.1.1)

**T13. In-tree close.** `@Environment(\.dismiss)` "has no effect on a view that isn't currently presented" [DismissAction doc]. Put a close action in the environment (sheet mode: `dismiss()`; tree mode: clear the item), keyed by selection id, returning whether it closed. Run clean-up on `.onChange(of: item?.id)` when in-tree and the id changed, so direct clears also fire it once. (shipped)

**T14. Cover band.** `safeAreaBar(edge: .top) { Color.clear }` gives no edge effect, and `.hard` does not help. The bar must draw: `Rectangle().fill(Color.primary.opacity(0.001)).frame(height: h).accessibilityHidden(true)`, with content top padding reduced by `h`. Then let the band take touches wherever it has height (no `allowsHitTesting(false)`), one rule, not gated on the hinge or pose. (shipped; observed behaviour, not documented)

**T15. Hinge first report flips identity.** A Bool fed by the hinge starts false and flips true after the view appeared; an `if` on it gives the subtree new identity, and the old view's `onDisappear` runs after the new `onAppear` (Email's Search vanished). Duo only. Gate on fixed per-host values. Same rule for `accessibilityFocused`, which also blanks `ImageRenderer` output when attached to offscreen-rendered content (E-A11Y-3). (shipped)

## 5. Verification protocol

Details: `references/verification.md`. Trial on a new app: `references/fresh-app-test-plan.md`.

- Simulator: Xcode 27.1, iPhone Duo (boots CLOSED). Only Device Hub poses it (Closed, Partially Open, Open, Rotate Right; continuous hinge and 3D rotation). No simctl, devicectl or XCTest API. Launch Device Hub by full path in the background: `open -g -a "$(xcode-select -p)/../Applications/DeviceHub.app"`. Never quit it (it shuts down every simulator). Give each Duo a unique name; target its exact window.
- Control runs: an iPhone on iOS 27.0 (only fallbacks run), plus the smallest supported iPhone. A failure red on the Duo and green on the 27.0 pin is a Duo-host artifact: list it as such (E-TEST-1).
- Proxies first: Device Hub's resizable iOS simulator and iPhone Mirroring (Apple's listed tools; resizable Mirroring needs macOS 27 [PDUO]); iPad windowing also helps. Then Split View on the LEFT, cover turned, RTL, Reduce Transparency, Reduce Motion, AX5.
- Poses: six = three pose buttons x default and one Rotate Right. Priority order when time is short: cover; open held wide; partly open held wide (40 pt band); partly open turned (horizontal fold, below 678 pt); one fold-unfold transition with the screen's sheet, form or panel open. The order only sequences the work: open turned and closed turned still run for every changed screen, and Done still means all six.
- Captures: `xcrun simctl io <UDID> enumerate` once; `screenshot --display=1` cover, `--display=3` inner (comes back rotated held wide); off display is black; `--mask=alpha` adds the corner mask to raw captures (cover corners 8 at the hinge side, 59 outer; inner 55). Record transitions on BOTH displays.
- UI tests: decide the DEVICE by model (`SIMULATOR_MODEL_IDENTIFIER == "iPhone19,4"`, else `uname` on hardware) and use window width only for the pose: an iPhone held sideways is 956 pt wide. `XCUIScreen.main` is the cover. Side-column tab items are plain buttons, not `app.tabBars`. Fold tests wait at named states, check the window really changed, and throw `XCTSkip("FOLD_SKIPPED: ...")` that the gate FAILS on unless opted out. Run pose stages with `-parallel-testing-enabled NO`; accept `TEST EXECUTE SUCCEEDED`; keep each stage's `-resultBundlePath`.
- Render tests on the Duo host inherit the cover's insets (right 84, bottom 34): set `safeAreaRegions = []` on the test's hosting controller (queued in Mitamaton, unproven). `ImageRenderer` draws a `ScrollView` empty: guard against flat cells.
- Automated accessibility: `performAccessibilityAudit` per pose; `XCUIDevice.shared.voiceOverService` for order across the fold (iOS 27.0, unmeasured across a fold).
- Never hand-pick a test subset after touching shared Duo plumbing; run an affected-test selector's picks plus every render suite that draws the presenter.
- CI cannot pose the Duo: run geometry units and cover-pose UI tests there, fold proofs on a GUI Mac, and require a local fold-proof PASS on the release sha.
- One booted simulator per test run; iPhone half then Duo half; shut down by UDID only the simulators you booted.

Done means: audit hits fixed or justified; every target audited (or marked N/A); guard tests red on old code, green now; every changed screen seen in six poses plus a 27.0 iPhone; fold and unfold with every sheet, panel, form, keyboard and playback open loses nothing and places nothing from the other screen; fold proofs PASS with zero FOLD_SKIPPED; the archive links the 27.1 SDK with `#available` fallbacks and an unchanged MinimumOSVersion.

## 6. App Store and release

- Duo slot: cover 1398 x 2034 / 2034 x 1398, inner 2007 x 2853 / 2853 x 2007, no scaled fallback. April 2027: the spec page says required for apps on the 27.1 SDK; Apple's news post says every submission. Plan for the stricter reading (E-STORE-2). Submitting the set adds the "optimized for iPhone Duo" badge, so only upload it against a build that links 27.1 (E-STORE-6).
- ASC display type `APP_IPHONE_DUO` (live API; missing from the published spec): pass it explicitly. Flatten to RGB. Gate each file to the four exact sizes (not XCUIScreen's 2006 x 2852). Replace sets in full: new versions inherit old frames. Check EVERY frame against the change that triggered a retake. Read back each as COMPLETE.
- Bezel frames: Apple Design Resources' Duo bezel opening is 2853 x 2007 at (120, 120); more than half of each set must show the app in use (2.3.3).
- App previews for the Duo accept only 886 x 1920 / 1920 x 886, so a preview lands in "A Closer Look" (E-STORE-5).
- Paywall, terms, Restore, sign-in and account deletion get a pose pass even when unedited; recapture App Review IAP screenshots when they change.
- An XCUITest screenshot step is a late timing marker on the Duo: time grabs from a logged state and capture with `simctl io --display=3`.
- Featuring nomination: App Enhancements, 3 weeks ahead, Duo poses named in Helpful Details. Time-limited Duo copy goes in Promotional Text (changes without a new submission [PVI]) with a dated revert. What's New stays factual: Duo-week wording is fine there if it states what actually changed for the Duo. TestFlight "What to Test" as a six-pose checklist.
