# iPhone Duo edge cases, verified

Compiled in October 2026 from four sweeps (lessons from a shipping app, app archetypes, lifecycle, testing, store), each item checked against its source by a second pass. Where the check corrected an item, the corrected wording is used here.

Each entry: **Case**, **Breaks**, **Detect**, **Fix**, **Verify**, **Source**. Tags:
- `[General]` applies to any app. `[General; seen in a shipping app (Mitamaton)]` is just as general, with the proof taken from the worked example in `mitamaton-case-study.md`. Nothing here is specific to that app; details from it are marked "(Mitamaton)".
- `(unverified)` marks a part nobody has measured on a Duo or that Apple has not stated. Test it before asserting it.
- The sweep id (D01, A05, L3, T07, store-04) is kept so a trial can report which item fired. `(review)` marks entries added from a review of missing areas; where Apple says nothing Duo-specific those entries say so, and their detect and verify steps are the point.

Source keys (HIG, PREP, TT461, LAB0917, ...) are listed in `apple-guidance.md`. "Case study" means `mitamaton-case-study.md`.

Areas: [Lifecycle and scenes](#lifecycle-and-scenes) · [Layout](#layout) · [Bars and chrome](#bars-and-chrome) · [Presentation and in-tree panes](#presentation-and-in-tree-panes) · [Input](#input) · [Accessibility](#accessibility) · [Archetypes](#archetypes) · [Testing](#testing) · [Store and release](#store-and-release)

---

## Lifecycle and scenes

### E-LIFE-1. No UIScene lifecycle: the rebuild will not launch `[General]` (A01)
- **Case:** no `UIApplicationSceneManifest`, or the AppDelegate owns `var window`. Typical of old storyboard apps, React Native/Expo templates, older Flutter, Unity's trampoline.
- **Breaks:** TN3187: from the next major release after iOS 26, building with the latest SDK without the scene lifecycle means "your app won't launch". The requirement starts with the iOS 27 SDK, so a 27.0 rebuild already hits it. The rebuild that turns on Duo layout kills the app on every device.
- **Detect:** skip if `@main struct ...: App` exists (SwiftUI apps already use scenes). Else `/usr/libexec/PlistBuddy -c 'Print :UIApplicationSceneManifest' Info.plist` errors; `rg -n -F 'configurationForConnecting'` = 0; `rg -n -e 'var window: UIWindow' -e 'self.window = \[\[UIWindow alloc' AppDelegate*`; `flutter --version` below 3.38; RN/Expo AppDelegate with `window` and no SceneDelegate.
- **Fix:** add a scene manifest and a `UIWindowSceneDelegate`; move window, root view controller and foreground/background handling there (push-token callbacks stay in the AppDelegate). Flutter: 3.38+ and its UIScene adoption guide. RN/Expo: migrate the native project or wait for upstream. Multiple scenes stay optional.
- **Verify:** build with Xcode 27.1, launch on a 27.0 iPhone and the Duo: first screen, no UIKit assert; background, foreground, deep links and push still work.
- **Source:** TN3187 (fetched 2026-10-23, quoted verbatim). Flutter breaking-change page. Expo SDK 56 launch-assert report and the exact Flutter version are secondary (unverified).

### E-LIFE-2. `UIScreen.main`, all of it `[General; seen in a shipping app (Mitamaton)]` (A02, L11, D31)
- **Case:** `UIScreen.main` has been deprecated since iOS 26.0 (reference page; TT461 4:03 still says "will be deprecated"). Beyond `bounds`: `scale`, `brightness`, `isCaptured`/`capturedDidChangeNotification`, `displayLink(withTarget:)`, `maximumFramesPerSecond`, `mirrored`, and treating `UIScreen.screens.count > 1` as "external display".
- **Breaks:** geometry from the cover on the open screen; warnings-as-errors builds fail. A guard that scans only `.bounds`/`.nativeBounds` lets `UIScreen.main.scale` through (Mitamaton had one in an attachment view). Which screen brightness and capture state act on, and what `UIScreen.screens` returns on a Duo, are (unverified).
- **Detect:** `rg -n -e 'UIScreen\.main' -e '\[UIScreen mainScreen\]' -e 'UIScreen.screens' -e 'didConnectNotification' -e 'isCaptured' -e 'displayLink\(withTarget' -e '\.brightness'` across app, packages, extensions, every language. Build log: `'main' was deprecated in iOS 26.0`.
- **Fix:** geometry from the view or window; scale from `displayScale` / `traitCollection.displayScale`; brightness on `window.windowScene.screen`; capture state from `traitCollection.sceneCaptureState` (iOS 17); frame timing with `UIUpdateLink(view:)` (iOS 18, "adjusts to the timing of the new display automatically"); external displays by scene session role, not screen count. Widen the guard to every `UIScreen.main` read, comments excluded, with a minimum file count.
- **Verify:** guard red on old code, green after. On the open Duo log `UIScreen.screens.count` closed and open; a display-link animation runs at full rate. Connect an external display or AirPlay target closed and open and log which scene and role receive it (E-LIFE-13).
- **Source:** `uiscreen/main.json` deprecatedAt 26.0 (fetched); TT461 4:16, 9:25; sceneCaptureState, UIUpdateLink docs (fetched); case study section 16.

### E-LIFE-3. Global window lookup with two instances `[General]` (A16, L5)
- **Case:** `connectedScenes.first`, `first { activationState == .foregroundActive }`, `keyWindow`, `windows.first`, a singleton holding the root view controller. An early draft of this skill's T1 snippet did this.
- **Breaks:** only when multiple scenes are supported (`UIApplicationSupportsMultipleScenes`, scene manifest). Two instances in Split View are both foreground active; instance A measures, presents or toasts in B's window. `@AppStorage` is shared, so "last tab" written by one changes the other live. (Mitamaton declares no multiple-scene key, so its 13 `connectedScenes` uses are latent.)
- **Detect:** `rg -n -e 'connectedScenes' -e '\.keyWindow' -e 'windows\.first' -e 'UIApplication.shared.windows' -e '@AppStorage'`; check the plist key.
- **Fix:** `view.window?.windowScene`, a zero-size representable (WindowBox), the scene passed to the delegate. Per-window UI state in `@SceneStorage` or per-scene models; `@AppStorage` for true preferences only.
- **Verify:** two instances side by side on the open Duo (or iPad): sheet, tab change and one-off height act only on their own window.
- **Source:** TT464 3:38; LAB0916 57:37-57:44 (shared app storage); LAB0917 33:53 (shared audio route).

### E-LIFE-4. New windows requested on the cover `[General]` (A17, L7)
- **Case:** `openWindow`, `requestSceneSessionActivation`, "Open in New Window", drag to a new window, called unconditionally.
- **Breaks:** the outer display cannot create new windows; the request fails or does nothing behind a visible control; code waiting for the scene hangs.
- **Detect:** `rg -n -e 'openWindow' -e 'requestSceneSessionActivation' -e 'UIWindowScene.ActivationAction' -e 'DocumentGroup' -e 'supportsMultipleWindows'`.
- **Fix:** `UIWindowScene.ActivationAction` for menu items (hides itself when new windows are unavailable); always handle the error callback and fall back to opening in place. Do not rely on `@Environment(\.supportsMultipleWindows)`: per its doc it is static per platform and configuration, nothing says it tracks the cover (unverified on the Duo).
- **Verify:** on the closed cover every "new window" path is hidden or opens in place; open, each creates a window.
- **Source:** TT464 3:48-4:15; supportsMultipleWindows doc (fetched).

### E-LIFE-5. System sheets anchored to the wrong scene `[General; seen in a shipping app (Mitamaton)]` (L6, D30)
- **Case:** `ASWebAuthenticationSession.presentationAnchor(for:)`, `ASAuthorizationController`, `Product.purchase(confirmIn:)`, `AppStore.requestReview(in:)`, `showManageSubscriptions(in:)`, `presentOfferCodeRedeemSheet(in:)` fed `connectedScenes.first` or an empty `ASPresentationAnchor()`.
- **Breaks:** with two instances the sheet lands over the other half or not at all, and the flow waits forever. An empty anchor can fail to present. A fold is a resize, so a held window stays valid; behaviour mid-fold with a system sheet up is (unverified). Mitamaton: `scene?.keyWindow ?? ASPresentationAnchor()` in four OAuth sign-in sheets.
- **Detect:** `rg -n -e 'presentationAnchor\(for' -e 'ASPresentationAnchor\(\)' -e 'purchase\(confirmIn' -e 'requestReview' -e 'showManageSubscriptions' -e 'ASAuthorizationController'`.
- **Fix:** capture the presenting view's window at tap time; in SwiftUI use `@Environment(\.purchase)`, `\.requestReview`, `\.webAuthenticationSession`. Fall back to the foreground-active scene, never an empty window.
- **Verify:** start each auth and purchase flow on the cover, unfold with the sheet up, finish: one completion. With two instances, trigger from the left one.
- **Source:** StoreKit AppStore and Product doc indexes (fetched); TT464 3:38; LAB0917 3:04.

### E-LIFE-6. Scene phase side effects and no restoration `[General; seen in a shipping app (Mitamaton)]` (D28, L8)
- **Case:** handlers on `.inactive`/`.background` that lock, log out, stop recording, end sessions, cancel uploads; no `@SceneStorage` or `stateRestorationActivity`; refresh only in `onAppear`.
- **Breaks:** Apple says closing can change scene phase, and closing in Split View backgrounds the app not touched last. A fold ends user work; a reclaimed backgrounded app loses its place; `onAppear` does not fire again on return. (That every fold passes through `.inactive` is not established.)
- **Detect:** `rg -n -e 'scenePhase' -e 'sceneWillResignActive' -e 'sceneDidEnterBackground' -e 'willResignActiveNotification' -e 'didEnterBackgroundNotification'` and read each handler; `rg -c -e '@SceneStorage' -e 'stateRestorationActivity'` (0 = none). Mitamaton: 35 scenePhase hits in app sources, 0 `@SceneStorage`.
- **Fix:** end long-running work only on `.background`; save drafts on `.inactive` without ending the session; privacy shields only on `.inactive`; refresh when phase becomes `.active`; adopt `@SceneStorage` for navigation, selection, drafts; background modes only where really needed.
- **Verify:** start a recording, fold, unfold: it continues. Split View, touch the other app, close, reopen soon and later: unlocked, work intact. Kill while backgrounded, relaunch: navigation and draft restored.
- **Source:** LAB0917 3:52; LAB1013 43:38-46:01; case study section 16; SceneStorage doc.

### E-LIFE-7. Orientation locks, forced rotation, `UIRequiresFullScreen` `[General]` (A28, L13)
- **Case:** portrait lock or `UIRequiresFullScreen` expected to behave as on iPhones; screens forcing orientation (`requestGeometryUpdate`, `setValue(_:forKey: "orientation")`, toggled `supportedInterfaceOrientations`), "rotate your phone" overlays, transforms faking rotation.
- **Breaks:** TT461 4:37-4:52: the key is honoured but the app still resizes on fold and "will scale on the inner display, including in split view multitasking". The inner display ignores supported orientations, so flows that WAIT for a rotation never proceed. (Whether "scale" means stretched rendering or a resize is unstated; how `requestGeometryUpdate` behaves on the inner display is unverified.) The HIG lets games lock orientation provided they fill the screen in every pose.
- **Detect:** Info.plist `UIRequiresFullScreen`, portrait-only `UISupportedInterfaceOrientations`; `rg -n -e 'requestGeometryUpdate' -e 'setNeedsUpdateOfSupportedInterfaceOrientations' -e 'forKey: *"orientation"' -e 'rotationAngle' -e 'rotationEffect'`.
- **Fix:** treat the key as no protection; size-driven layout; nothing gated on an orientation callback; treat `requestGeometryUpdate` as a refusable hint and handle its error. Consider cover landscape (Apple suggests it for tent use). Cameras: rotation coordinator.
- **Verify:** with the key set, open held wide, turned tall, and Split View both sides; record whether it resizes or scales (file the observation in `apple-guidance.md`). Each forced-orientation flow: no stuck overlay, content upright.
- **Source:** TT461 3:30, 3:36, 4:37-4:52; LAB1015 38:32-40:29; HIG games line.

### E-LIFE-8. Extensions and other targets left out of the audit `[General; seen in a shipping app (Mitamaton)]` (L20, D30)
- **Case:** share and action extensions, custom keyboards, notification content extensions, iMessage apps, App Intents snippets (App Clips, controls, interactive snippets and Safari web extensions also have E-LIFE-14), with fixed sizes (`preferredContentSize` 420 x 540 in Mitamaton's share extension) or hand-placed buttons.
- **Breaks:** the audit and six-pose walk never touch them; buttons never reach the column; content sizes from the screen. Specific breakages (stretched keys) are plausible but (unverified).
- **Detect:** `rg -l NSExtensionPointIdentifier -g '*.plist'`; app-extension targets in project.yml or pbxproj; in those folders `rg -n -e 'preferredContentSize' -e 'UIScreen' -e 'UIInputViewController' -e 'frame\(width: *[0-9]'`.
- **Fix:** run the same audit and pose walk per target; size from the extension's view; set `preferredContentSize` from fitting size.
- **Verify:** the audit report lists every target with a verdict; capture each extension point closed and open.
- **Source:** PREP ("based on your scene or containing view's bounds"); case study section 16.

### E-LIFE-9. Widgets and Live Activities `[General]` (L17, D30)
- **Case:** on the Duo a Live Activity expands from the BOTTOM of the Dynamic Island and is narrower; the outer camera occlusion grows into the Island while one shows; side-column items share space with Live Activities and overflow. Extra-large widgets fill half the open display in StandBy.
- **Breaks:** expanded regions truncate; app bar items vanish into overflow only while an activity runs; custom chrome near the Island collides.
- **Detect:** `rg -n -e 'ActivityConfiguration' -e 'DynamicIslandExpandedRegion' -e 'ActivityKit' -e 'supportedFamilies'`.
- **Fix:** flexible activity regions, reading the environment value for horizontally constrained presentation (Apple did not name it: unverified symbol); `visibilityPriority` on key bar items; keep chrome off `.occlusion` regions, re-read on change. Record "N/A" where the feature does not exist. Extra-large widget support is optional.
- **Verify:** compact, minimal and expanded states on the cover and open; app open with the activity running.
- **Source:** LAB1015 20:26-20:41; TT466 5:37-5:51; HIG reserved regions; LAB1013 15:34, 31:41.

### E-LIFE-10. Findings on unreachable screens `[General; seen in a shipping app (Mitamaton)]` (D30)
- **Case:** the audit reports and fixes screens nothing constructs.
- **Breaks:** wasted work (Mitamaton planned fixes for a tour chapter and an Email prime nothing reached).
- **Detect:** for each finding, find the view's initializer call and the flag defaults on a shipping path.
- **Fix:** drop or park dead-path findings with a note.
- **Verify:** every finding in the report names its entry point.
- **Source:** case study section 16.

### E-LIFE-11. Hinge state arrives after first layout `[General; seen in a shipping app (Mitamaton)]` (D09, T22)
- **Case:** an `if` (or a modifier inside one) gated on a hinge-derived Bool, or on the first `reservedRegions` report; the value starts false and flips asynchronously.
- **Breaks:** Duo only. The subtree gets new identity mid-launch; the old view's `onDisappear` runs after the new `onAppear`, so paired state ends wrong (Mitamaton: Email's Search vanished). A sheet opened in that first hop can switch from sheet path to pane path.
- **Detect:** `rg -n -e 'UIHingeInteraction' -e 'onHingeChange' -e 'reservedRegions'`, follow each into an `if`/`switch`/`.id(`; look for `onAppear`/`onDisappear` set/clear pairs nearby.
- **Fix:** gate identity on something fixed per host (its surface, a value injected at construction). Treat "unknown" and "no fold" the same for structure; change only padding and frames when the report lands.
- **Verify:** guard test flips the flag under a hosted view and asserts identity is kept (Mitamaton pins this with a mutation-tested guard); Duo UI test switches to the tab immediately and asserts its controls.
- **Source:** case study section 17 (the hinge flag was set via `DispatchQueue.main.async`); SKILL T15.

### E-LIFE-12. Relayout storms on hinge streams `[General]` (T23)
- **Case:** layout reading the hinge angle or region frames writes state on every update while someone folds slowly or scrubs Device Hub's hinge.
- **Breaks:** hitches, 99% CPU, feedback loops; tests assuming a fixed callback count flake. Apple: "rate and granularity of hinge angle updates are system policy". The simulator renders both displays at 60 Hz, so its fps says nothing.
- **Detect:** `rg -n -e 'onHingeChange' -e 'UIHingeInteraction' -e 'onGeometryChange'` with state writes and no tolerance; body-eval counters spiking during a scrub.
- **Fix:** do not lay out from the hinge angle at all (Apple: hinge for interactions and effects, arrangement and region APIs for layout); quantise, ignore sub-half-point deltas, keep structure fixed (E-LIFE-11). Measure with body counts or signposts across Closed, Partially Open, Open.
- **Verify:** body-eval count per fold under a stated budget; `sample <pid>` during a scrub shows no spinning loop; confirm on a device.
- **Source:** forum thread 847644; Duo capabilities.plist refreshRate 60; SKILL T10.

### E-LIFE-13. External display and AirPlay while folded `[General]` (review)
- **Case:** an app that shows content on a TV or monitor (cable or AirPlay), or that assumes a connected display mirrors "the" screen. On the Duo there are two built-in displays, and the device can open or close while connected.
- **Breaks:** Apple: "Beginning in iOS 27, your app receives a scene with the `windowExternalDisplayNonInteractive` role only after it registers a scene accessory. In earlier releases, the system connected this scene automatically." So a 27.x rebuild that relied on the automatic scene (or on `UIScreen.didConnectNotification`) shows nothing extra on the TV. Without that scene the system mirrors "your app's primary display". Which built-in display is mirrored when the Duo is open versus closed, and what a fold does to an active mirror, are (unverified, test it). Scene accessories are "enabled by default, but can be toggled at any time" (TT464 4:43-4:58), so availability can change mid-session.
- **Detect:** `rg -n -e 'windowExternalDisplayNonInteractive' -e 'windowExternalDisplay' -e 'didConnectNotification' -e 'UIScreen.screens' -e 'sceneAccessor' -e 'AVRoutePickerView' -e 'AVPlayerViewController'`; check the scene manifest for an external-display configuration.
- **Fix:** register a scene accessory for supplementary TV content, as the connected-display article describes; observe its availability and keep the app fully usable without it (Apple: "design your app to remain fully functional without the external display"); decide by scene role, never by screen count (E-LIFE-2). Video players: `AVPlayerViewController` handles AirPlay itself.
- **Verify:** the Duo profile lists a TVOut display (id 2, E-TEST-17); attach an external display in the simulator if the host offers it, else on a physical Duo with AirPlay to an Apple TV. Connect closed, then open; open, then close; toggle the accessory off and on. Log the scene role and window size each time and capture the TV. No crash, no stuck black TV, the phone UI unaffected.
- **Source:** "Presenting content on a connected display" and `windowExternalDisplayNonInteractive` reference (both fetched 2026-10-23, quoted); TT464 4:21-4:58 (scene accessories); Duo profile display list (E-TEST-17). Duo-specific mirroring behaviour (unverified).

### E-LIFE-14. App Clips, controls, interactive snippets, Safari web extensions `[General]` (review)
- **Case:** targets and system surfaces outside the main app: an App Clip target, `ControlWidget` controls (Control Center, Lock Screen, Action button), App Intents that return static or interactive snippets (`SnippetIntent`, `ShowsSnippetView`), and Safari web extensions shipped inside the iOS app.
- **Breaks:** each is missed by an audit that only walks the main app (E-LIFE-8). An App Clip is its own app target with its own Info.plist, so the scene-lifecycle rule (E-LIFE-1) and the `UIScreen.main` sweep (E-LIFE-2) apply to it separately (inferred from it being a separate target). Controls are template-based, which Apple says ensures they "work at all sizes and in all system spaces", so they are low risk. Snippet views are your SwiftUI, drawn by the system at a width you do not choose; how wide they are on the cover and the inner display is (unverified, test it). Apple also says that with Siri AI the system "might not display `IntentDialog` or `ShowsSnippetView`", so a snippet cannot be the only place a result appears. Safari web extension popups are web pages: the E-ARCH-7 rules apply, and their size on the Duo is (unverified).
- **Detect:** App Clip: a target of product type `com.apple.product-type.application.on-demand-install-capable` in pbxproj or project.yml. Controls: `rg -n -e 'ControlWidget' -e 'ControlWidgetButton' -e 'ControlWidgetToggle'`. Snippets: `rg -n -e 'SnippetIntent' -e 'ShowsSnippetView' -e 'ShowsSnippetIntent'`. Safari: `rg -l 'com.apple.Safari.web-extension' -g '*.plist'` (inferred extension point id; confirm in the target's Info.plist).
- **Fix:** run the same audit per target (E-LIFE-8); size snippet views flexibly (no fixed widths, `ViewThatFits`), and make sure the app or a dialog also reports the result; web-extension popups follow E-ARCH-7.
- **Verify:** launch the App Clip on the cover and open (Xcode can run an App Clip target directly); add each control to Control Center and the Lock Screen, use it closed and open; run each snippet intent from Spotlight or Shortcuts closed and open, capture; open the Safari extension popup closed and open. Record "N/A" for any surface the app does not ship.
- **Source:** App Clips, `ControlWidget`, "Displaying static and interactive snippets" and "Safari web extensions" pages (all fetched 2026-10-23). Nothing Duo-specific in any of them.

### E-LIFE-15. Calls, alarms and SharePlay across a fold `[General]` (review)
- **Case:** a VoIP app using CallKit (with PushKit), an alarm or timer app, or a SharePlay (Group Activities) session, where the system interrupts or the session runs while the device opens or closes.
- **Breaks:** CallKit "displays the same interfaces as the Phone app", so the incoming-call screen itself is the system's. Your in-call screen after answering is your UI and must survive a fold with the call, mute state and video views intact (E-LIFE-6, E-ARCH-5). AlarmKit: an Apple engineer said the inner-display glow in tent pose "is provided by AlarmKit" and that a second simultaneous display is otherwise only for the camera accessory, so a custom alarm screen cannot reproduce it (LAB0917 13:46-14:30, 31:21-32:07; tent behaves like closed landscape for other apps). SharePlay: playback sync is per device; a player owned by a view that a fold rebuilds drops out of the session (inferred from E-ARCH-5, unverified on the Duo).
- **Detect:** `rg -n -e 'CXProvider' -e 'CXCallController' -e 'PKPushRegistry' -e 'AlarmManager' -e 'import AlarmKit' -e 'GroupActivity' -e 'GroupSession' -e 'AVPlayerPlaybackCoordinator'`.
- **Fix:** keep call, alarm and group-session objects in models outside any view that a size change can swap; let AlarmKit present alarms (its templated and widget presentations, E-LIFE-9 for the Live Activity part); end nothing on `.inactive` (E-LIFE-6).
- **Verify:** physical Duo for real calls and SharePlay (FaceTime is needed). Answer on the cover, open, close, end the call: one call, state kept. Fire an AlarmKit alarm in tent pose and closed. Start SharePlay playback, fold and unfold: still in sync. These are (unverified, test it) until run.
- **Source:** CallKit, AlarmKit and Group Activities overviews (fetched 2026-10-23); LAB0917 13:46-14:30, 31:21-32:07 (AlarmKit glow, tent pose), 32:45-33:01 (FaceTime uses scene accessories on both displays).

### E-LIFE-16. "Designed for iPhone" on Apple silicon Macs and visionOS after the `#available` work `[General]` (review)
- **Case:** the iPhone build also runs unmodified on Apple silicon Macs ("iOS Apps on Mac") and on Apple Vision Pro as a compatible app, unless opted out. Duo code is gated with `#available(iOS 27.1, *)`.
- **Breaks:** an OS-version gate is not a device gate. Where the host reports a new enough iOS version, the Duo branch runs with no hinge, no fold region and no side column (inferred; what iOS version each host reports is (unverified, test it)). Code that treats "27.1 available" as "is a Duo", or force-unwraps a hinge, a region or a vertical edge, breaks there. Apple's own docs give the safe values: `UIHingeInteraction`'s hinge is nil when the hierarchy provides no hinge updates, and `toolbarVerticalEdge` is nil where there is no vertical bar. Window resizing on the Mac also exercises every resize path (inferred).
- **Detect:** `rg -n -e '#available\(iOS 27' -e 'isiOSAppOnMac' -e 'isiOSAppOnVision' -e 'hinge!' -e 'verticalEdge!'`; read each `#available` branch for Duo assumptions. App Store Connect > Pricing and Availability for the Mac and Vision Pro settings.
- **Fix:** branch on capability (hinge non-nil, an active division, a non-nil vertical edge), never on OS version or idiom; handle every nil. If the app makes no sense on a platform, opt out in App Store Connect (Apple documents this for both).
- **Verify:** run the "My Mac (Designed for iPhone)" destination and, if the app is available on Vision Pro, the visionOS simulator's Designed-for-iPhone destination; open every changed screen; resize the Mac window; no crash, no empty side column, no fold gap.
- **Source:** "Running your iOS apps in macOS", "Determining whether to bring your app to visionOS", `ProcessInfo.isiOSAppOnMac` (iOS 14) and `isiOSAppOnVision` (iOS 26.1) references (all fetched 2026-10-23); UIHingeInteraction and toolbarVerticalEdge docs (fetched 2026-10-23).

---

## Layout

### E-LAYOUT-1. Regular width is not iPad `[General]` (A03, L10)
- **Case:** `NavigationView` without `.navigationViewStyle(.stack)`; `if horizontalSizeClass == .regular { iPadLayout }`; `userInterfaceIdiom == .pad` checks; storyboard wR hR variations; asset-catalog Regular width or height variants; `~ipad` resources; pointer or keyboard features inferred from size class.
- **Breaks:** the inner display is regular x regular with the phone idiom, so idiom code and size-class code disagree on one screen; iPad-only constraints, assets and copy load on a phone. A `NavigationView` is expected to become two columns with an empty detail (from regular-width behaviour on other phones; confirm on the Duo sim).
- **Detect:** `rg -n -F 'NavigationView'`; `rg -n -e 'userInterfaceIdiom' -e 'UIUserInterfaceIdiomPad' -e 'isPad' -e 'horizontalSizeClass *== *\.regular'`; `rg -c 'widthClass="regular"' -g '*.storyboard' -g '*.xib'`; `rg -l '"width-class" *: *"regular"' -g '*.xcassets/**/Contents.json'`.
- **Fix:** `NavigationStack` (or `NavigationSplitView` where two columns are meant); decide by size class and measured width, never idiom; review each regular variant at 951 x 669 and 669 x 951.
- **Verify:** every such screen on the open Duo both orientations and on the cover; layout functions unit-tested with (regular, regular, phone idiom).
- **Source:** PREP ("Don't use userInterfaceIdiom or UIInterfaceOrientation for layout decisions", fetched); TT461 2:30, 2:46; LAB0917 52:55.

### E-LAYOUT-2. Storyboard and Auto Layout apps `[General]` (A19)
- **Case:** constraints to superview edges, symmetric constants, `multiplier 0.5` splits, frames set once in `viewDidLoad`, deprecated `traitCollectionDidChange` (iOS 17) and `statusBarOrientation`.
- **Breaks:** content under the 84 pt column, a seam on the fold, stale frames after a fold, wrong orientation inputs.
- **Detect:** in `.storyboard`/`.xib`: leading/trailing constraints whose second item is the root view, `multiplier="1:2"` or `"0.5"`; `rg -n -e 'traitCollectionDidChange' -e 'statusBarOrientation' -e 'view.bounds.width'` inside `viewDidLoad`.
- **Fix:** pin to `safeAreaLayoutGuide` / `layoutMarginsGuide`; width = `bounds.inset(by: safeAreaInsets).width`; frame code in `viewWillLayoutSubviews`/`layoutSubviews`; `registerForTraitChanges`; replace half splits with `UISplitViewController`, `UIArrangementViewController` or region-driven constraints.
- **Verify:** every scene on the cover, open held wide (and Split View left), open tall, folding on each.
- **Source:** PREP (fetched); TT461 7:30 talk code.

### E-LAYOUT-3. Screen buckets and device-model tables `[General]` (A18, T27)
- **Case:** `height < 700` small-phone paths, aspect checks, `bounds.height == 844`, `utsname`/DeviceKit model tables, `isSE` flags, including window-based buckets the `UIScreen` grep misses.
- **Breaks:** the cover (466 x 678, aspect about 0.69) picks a cramped SE layout or an "iPad" path; `iPhone19,4` falls into "unknown" with hard-coded insets.
- **Detect:** `rg -n -e 'height *[<>]=? *[5-9][0-9]{2}' -e 'width *[<>]=? *[34][0-9]{2}' -e 'utsname' -e 'DeviceKit' -e 'iPhone1[0-9],' -e 'isSmallScreen' -e 'isSE'`.
- **Fix:** size classes and continuous layout (`ViewThatFits`, flexible stacks). Where a threshold must stay, test it at 466 x 678, 678 x 466, 669 x 951, 951 x 669 and each Split View width. Where a model id is needed (tests, analytics), add `iPhone19,4` and read `SIMULATOR_MODEL_IDENTIFIER` in the sim.
- **Verify:** unit-test bucket functions with the four sizes; log the resolved device and tier on the Duo sim, no "unknown".
- **Source:** Prepare page ("`bounds.height == 844` ... won't match iPhone Duo"); LAB1015 39:58-40:16; HIG; Duo profile.plist.

### E-LAYOUT-4. Collection views and carousels sized to the device `[General]` (A20)
- **Case:** item size = `(width - padding)/3` computed once; compositional groups with fixed `count:`; carousel cards "device width minus padding".
- **Breaks:** 900 pt cards on the inner display; an odd column on the fold; stale item sizes after a fold.
- **Detect:** `rg -n -e 'itemSize *= *CGSize' -e 'count:' -e 'GridItem' -e 'invalidateLayout' -e 'sizeForItemAt'`.
- **Fix:** column count from available width and the fold, even when open; section provider with `layoutEnvironment.container.effectiveContentSize`, or `invalidateLayout()` on size change; cap card width or aspect so wider screens show more cards.
- **Verify:** scroll mid-list, fold and unfold: visible item kept, even columns open, no card past its cap.
- **Source:** LAB1013 29:00-29:33; HIG even columns; LAB1015 34:01-37:26.

### E-LAYOUT-5. Paged scroll views and onboarding carousels `[General]` (A21)
- **Case:** `isPagingEnabled` with page width = screen width, `UIPageViewController`, `TabView(.page)`, `.scrollTargetBehavior(.paging)`, art for 19.5:9.
- **Breaks:** after a resize the offset matches the old page width, so a half page shows (general resizable-app behaviour from iPad multitasking; not reproduced on a Duo). Full-bleed art crops at 951 x 669; a centred hero sits on the fold.
- **Detect:** `rg -n -e 'isPagingEnabled' -e 'UIPageViewController' -e 'tabViewStyle\(\.page' -e 'scrollTargetBehavior\(\.paging' -e 'UIPageControl'`.
- **Fix:** store the page index; restore `contentOffset = index * newWidth` after layout; SwiftUI `scrollPosition(id:)`. Open: subject on one side of the fold, copy and buttons on the other, capped (Mitamaton onboarding, 440 pt).
- **Verify:** on page 3, fold, unfold, rotate: still page 3, aligned. Partly open: nothing on the band.
- **Source:** case study section 12; HIG fold rules.

### E-LAYOUT-6. Window shapes the six poses never produce `[General]` (L4, A09)
- **Case:** (a) open and held tall with a video pinned at the top, the app below it in a short wide window; (b) 50/50 Split View on the inner display (about 475 x 669 each); (c) the side column's height changing with PiP, outer rotation or Split View.
- **Breaks:** minimum-height assumptions clip CTAs or stop scrolling; unseen size-class combinations; unexpected bar overflow. That a pinned PiP shrinks the host's window is inferred from LAB1013 1:01:21 ("at the bottom of the screen") (measure it).
- **Detect:** `rg -n -e 'minHeight: *[0-9]{3}' -e 'frame\(height: *[0-9]{3}' -e 'verticalSizeClass *== *\.compact'`; count toolbar items per screen and check `visibilityPriority`.
- **Fix:** every full-screen flow scrolls when it outgrows its window; layouts from measured size, not pose names; critical CTAs in a `safeAreaBar`/bottom inset or moved with regions; `visibilityPriority` and the system overflow.
- **Verify:** start PiP from Safari or TV, pin it in open portrait, open the app below and capture key screens; capture 50/50 Split View left and right.
- **Source:** TT464 3:17; TT466 2:55, 3:22; LAB1013 1:00:47-1:01:34; LAB0917 50:59; TT462 11:41-11:57; HIG Split View sizes.

### E-LAYOUT-7. Full-width text and centred screens with nothing selected `[General; seen in a shipping app (Mitamaton)]` (D17)
- **Case:** headers, brief or summary lines, email lists and centred single-column screens (sign-in, permission primes, chat) run the full 951 pt.
- **Breaks:** in book pose a sentence, row or primary button sits on the crease; App Store frames too (Mitamaton's submitted 02 chat and 03 board frames crossed the hinge).
- **Detect:** files with `.frame(maxWidth: .infinity)` plus text or button stacks and no `reservedRegions` read (`rg -L reservedRegions` over them); Partly Open capture of each screen, look at the 40 pt band.
- **Fix:** hold text to one lane with the same fold reader as the content; lists leading, trailing pane reserved with `ContentUnavailableView`; centred screens on one side of the fold in one `AnyLayout` tree, capped (Mitamaton 440 pt).
- **Verify:** Partly Open capture of each screen with nothing selected: no glyphs or controls in the band.
- **Source:** case study sections 15 and 16.

### E-LAYOUT-8. Readers and editors in book pose `[General]` (A11)
- **Case:** a centred readable column on the open screen held wide, scrolling vertically.
- **Breaks:** the column sits on a vertical fold; lines run through the band when partly open.
- **Detect:** `rg -n -e 'readableContentGuide' -e 'readableContent' -e 'frame\(maxWidth: *[0-9]' -e 'UITextView' -e 'TextEditor' -e 'WKWebView'`.
- **Fix:** Apple's reconciliation: do not displace continuous content; content scrolling parallel to the fold shifts off it or changes axis (Preview pages horizontally, two pages, in book pose). Pick one per reader. Keep the text anchor across the change (`UITextView` keeps it; custom renderers save a semantic anchor).
- **Verify:** partly open with a long document: no line crosses the band; position survives fold and unfold; readable at AX sizes.
- **Source:** LAB1015 47:27-48:30, 34:30-37:33; TT463 3:43; `apple-guidance.md` section 5.

### E-LAYOUT-9. Paywall on the stacked axis `[General; seen in a shipping app (Mitamaton)]` (D18)
- **Case:** a paywall that splits well held wide is held tall: the fold runs across, and a pinned bottom purchase stack leaves little room above.
- **Breaks:** plans and the auto-renew disclosure sit behind the buttons at rest (review risk, hides the best plan). Mitamaton estimates, unmeasured: decision half about 465 pt, stack about 200 pt. Measured open turned tall: at rest 5 of 9 rows sat above the main button.
- **Detect:** `rg -n -e 'SubscriptionStoreView' -e 'purchase' -e 'Restore'`; a `safeAreaInset(edge: .bottom)` holding buttons; the stacked case in the fold resolver.
- **Fix:** measure in Open turned and Partly Open turned; give the stacked axis its own arrangement in one `AnyLayout` tree (plans and disclosure on the half with room, or a capped pinned stack).
- **Verify:** a UI test per pose asserting each plan row and the disclosure are visible without scrolling, and on which side; the iPhone layout asserted separately.
- **Source:** case study sections 15 and 16; LAB0917 35:59.

### E-LAYOUT-10. Hard-coded display-hugging corners `[General]` (L21)
- **Case:** full-bleed cards, previews or sheets with a fixed large radius, or the private `_displayCornerRadius` key.
- **Breaks:** the outer display's corners away from the hinge are rounder, with corners in new places; one radius clips or leaves uneven gaps. (What the private key returns on a Duo is unverified.)
- **Detect:** `rg -n -e '_displayCornerRadius' -e 'cornerRadius: *[3-6][0-9]' -e 'layer.cornerRadius *= *[3-6][0-9]' -e 'static let .*[Rr]adius.*= *[3-6][0-9]'`, then follow radius tokens to the full-bleed views that use them.
- **Fix:** `ConcentricRectangle` / `UICornerConfiguration` (iOS 26); corner-adapted safe area near corners (T5).
- **Verify:** `simctl io ... screenshot --mask=alpha` or Device Hub window captures on the cover both orientations and open: equal inset per corner.
- **Source:** TT461 4:19-4:34; LAB0917 41:54-42:12; Duo capabilities.plist corner radii.

### E-LAYOUT-11. Launch screen to splash handoff `[General]` (A27)
- **Case:** launch storyboard logo centred in the safe area, then a custom splash re-centres on the screen; `scaleToFill` launch images.
- **Breaks:** the asymmetric 84 pt inset puts a safe-area-centred logo 42 pt off screen centre, so the handoff jumps; full-screen images distort at 0.69 and 1.42. The launch screen always shows the vertical status bar (no Info.plist key).
- **Detect:** LaunchScreen.storyboard image views with `scaleToFill` and safe-area constraints; `rg -n -e 'Splash' -e 'LaunchAnimation' -e 'lottie'` near app start.
- **Fix:** one anchor for both (screen centre or safe-area centre); aspect-fit or colour designs; splash geometry from its own view.
- **Verify:** cold launch on the cover and open held wide, recorded at 60 fps: no jump, no distortion.
- **Source:** LAB1015 21:24-21:44; TT461 7:30.

### E-LAYOUT-12. Display Zoom on the cover `[General]` (review)
- **Case:** iPhones offer Display Zoom (Settings > Display & Brightness), which enlarges the UI by giving the app fewer points. Layouts tuned at exactly 466 x 678 on the cover.
- **Breaks:** if the Duo offers Display Zoom, the cover would be narrower than 466 pt in points, so labels, side-by-side buttons and width thresholds tuned at 466 truncate or pick the wrong branch. Whether the Duo offers Display Zoom on the cover, the inner display or both, and what point sizes result, is (unverified, test it). Apple's Duo material does not mention it, and the Duo simulator profile has no key that names it (checked 2026-10-23).
- **Detect:** width thresholds near the cover width (E-LAYOUT-3 regex, plus `rg -n -e 'width *[<>]=? *4[0-9]{2}' -e 'nativeScale' -e 'nativeBounds'`); fixed-width rows on cover screens.
- **Fix:** continuous layouts (`ViewThatFits`, wrapping stacks) instead of point thresholds; where a threshold must stay, test it below 466 pt as well as at it.
- **Verify:** on a physical Duo, look for Display Zoom in Settings; if present, set the larger option and log the window size and `displayScale` on the cover and the inner display, then repeat the cover captures. In the simulator, check whether Settings offers it (unverified). As a proxy, run cover screens in the resizable simulator at a width below 466 pt (E-TEST-26).
- **Source:** no Apple Duo source found; Duo profile.plist and capabilities.plist (checked 2026-10-23: no Display Zoom key, only an unexplained `reverse-zoom` false).

### E-LAYOUT-13. Long strings at 466 pt: double-length and real long languages `[General]` (review)
- **Case:** copy sized in English for the cover: buttons, segmented controls, tab labels, single-line titles, onboarding headlines. Only RTL was on the test list.
- **Breaks:** German, Finnish or Russian strings, or a double-length pseudolanguage, overflow at 466 pt (and in a 50/50 Split View half, about 475 pt): truncation ("Continue wit..."), clipped buttons, two-line titles that push content down. The side column holds icons and reveals labels only while touched (LAB0917 6:53), so long tab titles mostly hit the cover's content area and any custom chrome. Nothing here is Duo-specific; the Duo adds a new narrow width.
- **Detect:** `rg -n -e 'lineLimit\(1\)' -e 'minimumScaleFactor' -e 'fixedSize\(\)' -e 'adjustsFontSizeToFitWidth' -e 'NSStringVariableWidthRuleType' -e 'NSStringDeviceSpecificRuleType'` on cover screens; any width or device string variants (they pick by width or device, so check which variant the cover and the inner display get).
- **Fix:** let text wrap and buttons grow; `ViewThatFits` for button rows; avoid width variants (Apple: "In Xcode 15 and later, use Dynamic Type instead of width variants").
- **Verify:** run with Xcode's double-length pseudolanguage (scheme > Run > Options > App Language; Apple documents pseudolanguages for Interface Builder previews, the scheme menu entry name is from Xcode's UI) and with a real long language, `xcrun simctl launch <device> <bundle> -AppleLanguages '(de)' -AppleLocale de_DE` (quote the parentheses in zsh). Capture every changed screen on the cover, in Split View left, and open wide; combine with AX5 on the cover (E-TEST-21) and `performAccessibilityAudit(for: [.textClipped])` (E-TEST-20).
- **Source:** "Testing localizations when running your app", "Previewing localizations" and "Creating width and device variants of strings" (fetched 2026-10-23); LAB0917 6:53; HIG Split View sizes.

### E-LAYOUT-14. QuickLook, PDFKit and document viewers in book pose `[General]` (review)
- **Case:** a PDF or document viewer: `QLPreviewController`, a `PDFView`, or a custom page renderer, scrolling vertically.
- **Breaks:** in book pose a vertically scrolling page runs parallel to the fold, so a line or figure sits on the band. Apple's own Preview and Quick Look switch to "horizontal scrolling with a two-page layout" when folded like a book (LAB1015 48:02-48:30; LAB1013 49:18-49:53). Whether a third-party `QLPreviewController` gets that behaviour automatically is (unverified, test it); a `PDFView` you configure does not change mode by itself (inferred: its display mode is a property you set).
- **Detect:** `rg -n -e 'QLPreviewController' -e 'QuickLook' -e 'PDFView' -e 'displayMode' -e 'displayDirection' -e 'displaysAsBook' -e 'usePageViewController'`.
- **Fix:** prefer `QLPreviewController` for plain viewing. For `PDFView`, on an active vertical division switch to `.twoUp` with `displayDirection = .horizontal` (and `displaysAsBook` if a cover page should stand alone), back to single-page vertical when flat; keep the current page across the switch (E-LAYOUT-8).
- **Verify:** open a multi-page PDF partly open held wide: no text on the band, pages turn horizontally; fold and unfold: same page; closed: normal vertical reading.
- **Source:** LAB1015 47:47-48:30; LAB1013 49:18-49:53; QLPreviewController, PDFView, `displaysAsBook` and `displayDirection` references (fetched 2026-10-23; none mention the Duo).

---

## Bars and chrome

### E-BARS-1. Empty `safeAreaBar` gives no edge effect `[General; seen in a shipping app (Mitamaton)]` (D06)
- **Case:** custom top chrome placed in `safeAreaBar(edge: .top)` for the cover's scroll edge, with content `Color.clear`, `EmptyView` or `Spacer` because the real chrome is drawn elsewhere.
- **Breaks:** no edge effect; `.scrollEdgeEffectStyle(.hard)` changes nothing (Mitamaton's first try differed by 1,666 px).
- **Detect:** `rg -n -A6 'safeAreaBar\(edge'` and check the content; a scrolled cover capture compared with the build before.
- **Fix:** the bar must draw: `Rectangle().fill(Color.primary.opacity(0.001)).frame(height: h).accessibilityHidden(true)` (a 0.001 fill, content at opacity 0, or visible content all work). Reduce content top padding by `h`; height 0 where the bar would wash content at rest.
- **Verify:** probe build with one DEBUG launch switch per variant, captured scrolled on the cover; then before/after with an unchanged rest frame.
- **Source:** case study section 10. Observed SwiftUI behaviour, not documented.

### E-BARS-2. Taps on the band reach the content under it `[General; seen in a shipping app (Mitamaton)]` (D07)
- **Case:** the bar fill uses `allowsHitTesting(false)`; the system bar host takes no touches.
- **Breaks:** a tap on the header band opens a card scrolled under it. Gating the rule on the hinge or pose makes it differ where the bar has height (open Duo at larger text).
- **Detect:** scroll content under the band and tap its empty side; `rg -n -B2 -A8 'safeAreaBar' | rg 'allowsHitTesting\(false\)'`.
- **Fix:** let the bar take touches wherever it has height while still forwarding taps to real chrome; one rule, not gated on `UIHingeInteraction` or pose (it is 0 pt on ordinary iPhones).
- **Verify:** UI test on the cover AND the open Duo held wide at a larger text size: band tap over a scrolled card opens nothing. Mutation: restore `allowsHitTesting(false)` and the test must fail.
- **Source:** case study section 10.

### E-BARS-3. Placing custom items in the bar region `[General; seen in a shipping app (Mitamaton)]` (D25)
- **Case:** custom chrome (a Back circle, a panel's collapse target) placed by guess or from a zero-size probe.
- **Breaks:** items sit at the old position after a fold or a vertical-edge change, because a zero-size probe "only measured when some other layout reached it".
- **Detect:** `rg -n -e 'LayoutRegion.bar' -e 'verticalBarEdge' -e 'toolbarVerticalEdge'`; zero-frame `UIViewRepresentable` probes measuring only in `layoutSubviews`.
- **Fix:** `UIView.LayoutRegion.bar(onEdge: edge == .leading ? .left : .right, extent:)` (iOS 27.1; Mitamaton measured x 903 / top 120 open, top 170 on the cover, matching the system glass at 902.8 / 119.3). Probe fills its surface and measures in `didMoveToWindow`, `layoutSubviews` and on `verticalBarEdge` changes; one shared probe.
- **Verify:** log the region open and closed against a capture of the system items; re-check after Rotate Right (column on the left).
- **Source:** case study sections 8 and 16; SKILL T4, T6.

### E-BARS-4. Side drawers and hamburger menus `[General]` (A24)
- **Case:** a drawer sized as a fraction of the screen (`bounds.width * 0.8`) with a custom Back or menu button.
- **Breaks:** about 760 pt wide or across the fold open; moving it to the other side for the Duo breaks continuity; a custom Back is not in the side column where users expect it.
- **Detect:** `rg -n -e 'SideMenu' -e 'Drawer' -e 'hamburger' -e 'line.3.horizontal' -e 'width \* 0\.[0-9]'`.
- **Fix:** prefer a sidebar (`NavigationSplitView`, `.defaultTabBarPlacement(.sidebar)`). If keeping the drawer: width capped in points, same side on both displays, off the fold; Back and menu as container toolbar items (`.cancellationAction`). If it acts as an overflow menu, move its items into `ToolbarOverflowMenu` / `additionalOverflowItems`.
- **Verify:** drawer on the cover, open held wide, partly open: sensible width, same side, nothing on the band, Back on top of the column.
- **Source:** LAB1015 22:55-25:57; TT461 5:44.

### E-BARS-5. Bottom rows under the side column `[General; seen in a shipping app (Mitamaton)]` (D21)
- **Case:** a non-scrolling bottom row (formatting, attachments, a reader's actions, a custom `safeAreaInset(edge: .bottom)` bar) laid out full width.
- **Breaks:** trailing buttons sit under the clock, tab icons or Close on the cover and cannot be tapped.
- **Detect:** `rg -n -e 'safeAreaInset\(edge: *\.bottom' -e 'ignoresSafeArea\(.*horizontal'` in editors and readers; cover capture with the keyboard up and down.
- **Fix:** respect the horizontal safe area; fold excess controls into one menu (Mitamaton: one Attach menu for 6 controls). This changes iPhones too; name it.
- **Verify:** cover both orientations plus Split View left: every control's frame clear of the bar region.
- **Source:** case study section 16.

### E-BARS-6. Bottom space reserved for a tab bar that moved `[General]` (D29)
- **Case:** a bottom reservation sized for a tab bar (constant or old custom bar).
- **Breaks:** on the cover and open held wide the tab bar is in the column: about 85 pt of empty foot, floating controls (Undo) sit high.
- **Detect:** `rg -n -e 'tabBarHeight' -e 'padding\(\.bottom, *[0-9]{2,}' -e 'safeAreaInset\(edge: \.bottom'`.
- **Fix:** derive from the measured bottom safe area, or from `toolbarVerticalEdge` (non-nil means the tab bar is in the column).
- **Verify:** cover capture: content runs to the foot; iPhone unchanged.
- **Source:** case study section 16; TT466.

### E-BARS-7. A Duo fix that changes every iPhone `[General; seen in a shipping app (Mitamaton)]` (D19)
- **Case:** a standalone pinned `UINavigationBar`, or moving a screen into `NavigationStack` so Close goes to `.cancellationAction`.
- **Breaks:** plain iPhones get a pinned 44 pt header (the paywall lost viewport) and Close moves from top right to top left; "iPhone unchanged" claims are false and a Duo-only decision is silently widened.
- **Detect:** diff against the pre-Duo base and capture the screen on a plain iPhone; `rg -n -e 'UINavigationBar\(' -e 'safeAreaInset\(edge: \.top'` in changed files.
- **Fix:** gate the pinned bar to layouts with a side column or split (`toolbarVerticalEdge != nil`, or a fold region). Where a change must reach every iPhone, name it as a broadening and get the product owner's sign-off.
- **Verify:** before/after on a plain and the smallest iPhone: identical pixels for gated cases.
- **Source:** case study sections 13 and 16.

### E-BARS-8. Mini-player above "the tab bar" `[General]` (A22)
- **Case:** a now-playing bar at a tab-bar offset, `safeAreaInset(edge: .bottom)` on the TabView, `tabViewBottomAccessory`, or an overlay at `bottom - tabBarHeight`.
- **Breaks:** with the tab bar in the column, the mini-player floats 49+ pt above nothing or overlaps content; full width crosses the fold in book pose. Where `tabViewBottomAccessory` lands with a vertical tab bar is (unverified).
- **Detect:** `rg -n -e 'tabViewBottomAccessory' -e 'MiniPlayer' -e 'NowPlaying' -e 'tabBarHeight'`.
- **Fix:** prefer the system accessory and capture where it lands; custom bars from the measured bottom safe area, off the fold; now-playing screen with `ArrangementView` (Podcasts: split wide, inline tall).
- **Verify:** play audio, check on cover tall and wide, open wide and tall, partly open, Split View.
- **Source:** TT463 9:36-10:26; LAB0917 9:33.

### E-BARS-9. Price and cart items in the bar; sticky CTAs `[General]` (A23)
- **Case:** a toolbar item showing a total or "Buy $9.99"; a sticky full-width bottom CTA.
- **Breaks:** forcing `.verticalPreferred` drops the price; a full-width CTA is cut by the band when partly open.
- **Detect:** `rg -n -e 'ToolbarItem.*Text\(' -e 'Text\(.*price' -e 'safeAreaInset\(edge: *\.bottom'` near purchase views.
- **Fix:** keep text-bearing items horizontal (Apple's advice); symbol-only actions go vertical; move revenue buttons off the fold with reserved regions, capped.
- **Verify:** cart and checkout on the cover and open wide: price visible; partly open: no CTA on the band.
- **Source:** TT462 9:37-9:58; LAB0917 35:59.

### E-BARS-10. Custom toasts, banners and snackbars at the top edge `[General]` (review)
- **Case:** an in-app toast, banner or snackbar (hand-made or from a library) pinned to the top of the window at `safeAreaInsets.top + n`, full width or centred.
- **Breaks:** on the cover the status bar, Dynamic Island and bars sit in the side column (HIG), so content runs to the top edge with little top inset; a top toast spans under the column, the outer camera corner or the Island, and the Island grows while a Live Activity shows (E-LIFE-9). Turned (Rotate Right) the column is on the other side. Libraries that add the toast to the key window also land in the other instance's window (E-LIFE-3). The exact top inset on the cover and the overlap are (unverified, test it); E-BARS covers headers, not transient overlays.
- **Detect:** `rg -n -i -e 'toast' -e 'snackbar' -e 'banner' -e 'SwiftMessages' -e 'NotificationBanner' -e 'AlertToast' -e 'SPIndicator' -e 'overlay\(alignment: *\.top' -e 'keyWindow\?*\.addSubview'`; package manifests for toast libraries.
- **Fix:** inset the toast horizontally by both side safe-area insets, keep it off `.occlusion` regions (read `reservedRegions`, kind occlusion), and on the cover prefer a bottom or inline placement; attach it to the presenting view's window.
- **Verify:** trigger every toast on the cover tall and turned, with a Live Activity running, open wide, partly open and in Split View left: capture, no overlap with the column, the camera corner or the Island, nothing on the band.
- **Source:** HIG (controls on the side include "the Dynamic Island, the status bar, the toolbar ... and the tab bar"; full width is fine "as long as nothing conflicts with the Dynamic Island or the status bar"); RR (occlusion kind, fetched); LAB1015 20:26-20:41.

### E-BARS-11. Status bar legibility over content behind the vertical column `[General]` (review)
- **Case:** `preferredStatusBarStyle` (or SwiftUI `.toolbarColorScheme`, `.preferredColorScheme`) chosen for what sits at the TOP of the screen, for example light text over a dark hero image.
- **Breaks:** on the cover and open held wide the status bar is in the side column (HIG), so its time and battery glyphs sit over whatever runs down that side, which changes as the user scrolls. A style picked for the top hero can leave glyphs dark on dark further down the column. How much the column's glass separates the glyphs from the content is (unverified, test it); Reduce Transparency puts an opaque plate behind the column (LAB0916 30:27). The launch screen always shows the vertical status bar (LAB1015 21:24-21:44).
- **Detect:** `rg -n -e 'preferredStatusBarStyle' -e 'UIStatusBarStyle' -e 'statusBarStyle' -e 'toolbarColorScheme' -e 'preferredColorScheme' -e 'UIViewControllerBasedStatusBarAppearance' -e 'prefersStatusBarHidden'`.
- **Fix:** pick the style for what lies under the column on that screen, not the top band; call `setNeedsStatusBarAppearanceUpdate()` when it should change (as the reference page says); keep content behind the column tonally consistent or give it a scroll-edge treatment (E-BARS-1); where `toolbarVerticalEdge` is nil the old rule still holds.
- **Verify:** cover tall and turned, open wide, light and dark appearance (`xcrun simctl ui <device> appearance dark`, restore after): scroll a light and a dark region under the column and capture; the glyphs stay readable in every frame.
- **Source:** `preferredStatusBarStyle` reference (fetched 2026-10-23, nothing Duo-specific); HIG (status bar in the side column); LAB0916 30:27; LAB1015 21:24-21:44.

---

## Presentation and in-tree panes

### E-PRES-1. Detail beside list, and the cover `[General; seen in a shipping app (Mitamaton)]` (D01)
- **Case:** `ArrangementView(.split)` or `NavigationSplitView` chosen for "detail beside its source".
- **Breaks:** `.split` is one tree only while both views are in it. Apple's doc: on a closed iPhone Duo held in landscape the split shows ONLY the primary. The talk, not the doc: when a style restricted with `.axes()` "cannot split along an axis and it's the primary axis, the arrangement view chooses to only show a single view" (TT463 12:41-12:57; the talk's example keeps the player). On the cover held tall it stacks (halving a 678 pt screen). The doc also says not to put a navigation split view or tab view INSIDE an arrangement view, and not to put an arrangement view inside a scroll view, navigation split view, grid or other layout container. A detail held as the secondary can leave the tree and lose `@State` on fold (note half-written, reader scrolled). `NavigationSplitView` keeps a detail column on the open screen even when nothing is selected.
- **Detect:** `rg -n -F 'ArrangementView'`, `-F 'arrangementViewStyle(.split'`, `-F 'NavigationSplitView'`; for each, does the secondary hold user state that must survive the cover? Does either side contain a `NavigationSplitView` or `TabView` (move it outside)? On the sim: open the detail open-held-wide, press Closed, check it is still there with its state.
- **Fix:** when the detail must survive, a one-tree host: one overlay, frame changes only (beside / fullScreen with the side column / closed). Duo detection from `UIHingeInteraction` (`update.hinge` was non-nil in every pose, closed included, in Mitamaton; Apple's doc only says nil when no hinge updates are provided); it reports async, see E-LIFE-11. Other iPhones keep `.sheet`; the `.sheet` modifier stays attached, bound to `.constant(nil)` while in-tree.
- **Verify:** fold-proof UI test: open, type, fold to the cover, text still there, unfold, again. Mutation: back to `ArrangementView(.split)` and the cover half fails.
- **Source:** ArrangementView doc (fetched 2026-10-23: closed-landscape line, both container rules); TT463 12:41-12:57 (single-view fallback; the doc does not state it); case study sections 7 and 16.

### E-PRES-2. `dismiss()` in an in-tree pane does nothing `[General; seen in a shipping app (Mitamaton)]` (D02)
- **Case:** a view shown both as a sheet and in-tree calls `@Environment(\.dismiss)`.
- **Breaks:** "The dismiss action has no effect on a view that isn't currently presented." Close or Delete does nothing on the Duo; iPhone tests stay green.
- **Detect:** `rg -n -e '@Environment\(\\\.dismiss\)' -e 'dismiss\(\)'`, crossed with views under a custom split host or `ArrangementView`.
- **Fix:** the presenter puts a close action in the environment (sheet: `dismiss()`; tree: clear the item); views call it and fall back to `dismiss()`. A view with its own close logic (consent, save, conflict) claims a close route so the column's Close and VoiceOver escape call it. Key the route by selection id (the next detail appears before the previous disappears).
- **Verify:** unit test: pane mode clears the item. Duo UI test: Delete in the pane closes it. Mutation: revert to `dismiss()`.
- **Source:** DismissAction doc (fetched, verbatim); case study section 16.

### E-PRES-3. `onDismiss` parity `[General; seen in a shipping app (Mitamaton)]` (D03)
- **Case:** a sheet runs `onDismiss` on ANY dismissal, including the model clearing the item (refresh, a repair tap setting selection nil). An in-tree presenter running clean-up only in its own `close()` skips those.
- **Breaks:** a deferred "after dismiss" action never runs, then fires unasked at the next close.
- **Detect:** `rg -n 'onDismiss:'`; `rg -n -e 'Selection = nil' -e 'item = nil'` for direct writes.
- **Fix:** `.onChange(of: item?.id) { old, new in if inTree && old != nil && old != new { onDismiss?() } }`; `close()` only clears the item. Sheet path untouched.
- **Verify:** pure test of the rule; presenter test: a direct clear and a Close each give exactly one clean-up. Mutation: remove the watch.
- **Source:** case study section 16.

### E-PRES-4. Re-taps and details that refuse to close `[General; seen in a shipping app (Mitamaton)]` (D04)
- **Case:** in-tree panes do not block taps the way sheets do. A re-tap of the same card, or a second source, while a detail is open; a detail that cannot close now (resolving a conflict).
- **Breaks:** a re-tap replaces the open detail and bypasses its consent or unsaved-changes close; two details open on one surface; a hidden tapped card appears later unasked.
- **Detect:** find the open function (`rg -n -e 'func openDetail' -e 'Selection = .*\('`) and its "already open" guard; whether the close route returns a Bool.
- **Fix:** `openDetail` does nothing when that source is already open; a second detail on the same surface closes the first through its route; the route returns whether it closed; on refusal post a signal so the newcomer clears its own item.
- **Verify:** unit test with two presenters and a refusing detail: the newcomer yields (mutation-tested in Mitamaton).
- **Source:** case study section 16.

### E-PRES-5. "Pane open" scoped to the process, not the surface `[General; seen in a shipping app (Mitamaton)]` (D05)
- **Case:** one process-wide flag drives the column's Close; open/close notifications carry no surface; `TabView` keeps visited tabs alive.
- **Breaks:** open a Home detail, switch to Email: Email's column shows Close with nothing open, and tapping it closes the hidden Home detail; opening an Email reader closes Home's. A host that unmounts while open (sign-out) leaves its entry stuck.
- **Detect:** `rg -n -e 'NotificationCenter.default.post\(name: \.[a-zA-Z]*(Close|Pane|Detail)' -e 'static let shared'`; a singleton Bool read by toolbar content in several tabs.
- **Fix:** a set of open surfaces; each host passes its surface through the environment; every post carries it and every observer checks it; clear on host disappear as well as on preference change.
- **Verify:** unit tests: a close on `.email` leaves `.home`; UI test: open on Home, switch tab, no Close.
- **Source:** case study section 16.

### E-PRES-6. Two Close buttons `[General; seen in a shipping app (Mitamaton)]` (D20)
- **Case:** the detail moves in-tree, the column gains Close, and the view's corner X is still drawn.
- **Breaks:** two X buttons for one action (both read "Close"); hiding the X breaks UI tests that tap the X identifier on the Duo.
- **Detect:** `rg -n -e 'system(Name|Image): *"xmark' -e 'Label\("Close"' -e 'Button\("(Close|Cancel|Done)"'` in views that can draw in-tree (SwiftUI's `Button(_:systemImage:)` spells it `systemImage:`); your X identifier in the test targets. Count Close controls per presented view, not per file.
- **Fix:** hide the view's X when in-tree and the column owns Close (an environment flag from the presenter); keep it on page sheets; UI tests resolve the X to the column's Close when absent.
- **Verify:** one X on cover and open Duo; iPhone sheet keeps its X; smoke tests pass on both.
- **Source:** case study section 16.

### E-PRES-7. Custom floating panels across a fold `[General; seen in a shipping app (Mitamaton)]` (D16)
- **Case:** a chat card or popover-like panel anchored to a bar button that hides while the panel is open.
- **Breaks:** closing on resize loses conversation or camera state on every fold (Apple repositions its own menus and popovers, TT463 5:23). Kept open naively, it collapses into the column's first slot (another item), then the real button returns lower.
- **Detect:** `rg -n -e 'onChange\(of: *geo.size' -e 'resizeCloses'` near panels; panels placed from a frame captured on tap.
- **Fix:** keep it open; re-place from the column measured on the CURRENT screen (E-BARS-3), drop the tapped frame on resize; record the button's slot in the column order at open, clear it on every open, compute only from a frame inside this pose's column.
- **Verify:** fold proof: open (and its camera cover), fold, still open and beside the column or fold, close, collapses into its own slot. Pure geometry tests for cover, open, book, leading column.
- **Source:** case study section 8; TT463 5:23.

### E-PRES-8. Camera picker as a card `[General]` (D27)
- **Case:** `UIImagePickerController` (`.camera`) or `VNDocumentCameraViewController` presented with `.sheet`.
- **Breaks:** may show as a card on the regular-by-regular inner display (Mitamaton's audit judged it likely; not captured). Apple's picker doc recommends the camera interface only full screen (written for iPad's regular size class).
- **Detect:** `rg -n -e 'UIImagePickerController' -e 'VNDocumentCameraViewController' -e 'sourceType = \.camera'`, then the presenter.
- **Fix:** `.fullScreenCover { CameraPicker(...).ignoresSafeArea() }`; keep it open through a fold; stub the camera with the photo library behind a DEBUG launch argument for sim fold tests.
- **Verify:** open Duo capture full screen; fold test with the camera cover open.
- **Source:** UIImagePickerController doc (fetched); case study section 16.

### E-PRES-9. Popovers and action sheets at regular width `[General]` (A04, L9)
- **Case:** `UIAlertController(.actionSheet)`, `UIActivityViewController`, `.popover`, `modalPresentationStyle = .popover`, with `sourceView` set only when `idiom == .pad`, or relying on adaptation to a sheet.
- **Breaks:** popover adaptation follows horizontal size class, so on the inner display these may present as real popovers on a phone for the first time (unverified on the Duo: a hypothesis from documented UIKit adaptivity; capture it before relying on it). Unanchored, they may hit the iPad "sourceView" exception or anchor at (0, 0) (unverified: that crash historically follows the iPad idiom). Folding mid-presentation adapts popover to sheet and back.
- **Detect:** `rg -n -e 'preferredStyle: *\.actionSheet' -e 'UIActivityViewController' -e '\.popover' -e 'popoverPresentationController' -e 'presentationCompactAdaptation' -e 'userInterfaceIdiom *== *\.pad'`.
- **Fix:** always set `popoverPresentationController?.sourceItem` (iOS 16) or `sourceView` plus `sourceRect`, whatever the idiom; prefer `ShareLink` and anchored `.confirmationDialog`; size popover content intrinsically; `presentationCompactAdaptation(_:)` (iOS 16.4) where a sheet is meant.
- **Verify:** trigger every share, action sheet and popover on the open Duo (wide and tall) and in Split View: no crash, anchored, off the fold; fold with each open.
- **Source:** TT461 2:46; LAB0917 52:55; sourceItem and presentationCompactAdaptation docs (fetched); HIG.

### E-PRES-10. `.inspector` across the fold `[General]` (review)
- **Case:** `.inspector(isPresented:content:)` (iOS 17), first seen on iPad and now reachable on a phone.
- **Breaks:** Apple: an inspector "can present as a trailing column in a horizontally regular size class, but adapt to a sheet in a horizontally compact size class". So on the Duo it is a trailing column open and a sheet on the cover, and a fold switches between the two with the inspector open. Inspectors keep horizontal bars (PREP). Whether the column's leading edge lands on the band in book pose, and whether content `@State` survives the column-to-sheet switch, are (unverified, test it). "Trailing column inspectors have their presentation state restored by the framework" covers whether it is shown, not your content state.
- **Detect:** `rg -n -e '\.inspector\(' -e 'inspectorColumnWidth' -e 'InspectorCommands'`.
- **Fix:** keep inspector content state in a model; set `inspectorColumnWidth` so the column clears the fold when partly open (read `.includeInactive` regions) or accept a full trailing half; make sure the sheet form on the cover shows the same controls.
- **Verify:** open the inspector open held wide, partly open, then fold to the cover (sheet) and back, with an edit in progress: the edit survives, nothing sits on the band, horizontal bars in the inspector.
- **Source:** `inspector(isPresented:content:)` reference (fetched 2026-10-23, quoted); PREP (inspectors keep horizontal bars, `apple-guidance.md` section 3).

### E-PRES-11. TipKit popovers anchored to side-column items `[General]` (review)
- **Case:** `.popoverTip(_:arrowEdge:action:)` (iOS 17) on a toolbar or tab item, often with a fixed `arrowEdge: .top` chosen for a horizontal bar.
- **Breaks:** on the cover and open held wide that item sits in the vertical column, possibly in overflow (E-LIFE-9, E-INPUT-2). A tip with a hard-coded arrow edge can point the wrong way or cover the column; a tip anchored to an item in overflow may have nowhere to show; a fold while the tip is up moves the anchor. All of this is (unverified, test it); Apple's popoverTip page says nothing Duo-specific, and system popovers are repositioned across a fold (TT463 5:23).
- **Detect:** `rg -n -e 'popoverTip' -e 'TipView' -e 'Tips.configure' -e 'TipUIPopoverViewController'`.
- **Fix:** let the system choose the arrow edge, or derive it from `toolbarVerticalEdge` (leading column: arrow `.leading`; trailing: `.trailing`); give the anchored item `visibilityPriority(.high)` so it stays out of overflow; for Duo layouts consider an inline `TipView` in content instead.
- **Verify:** force tips with `Tips.showAllTipsForTesting()` (and `Tips.resetDatastore()` before `Tips.configure`), then capture each tip on the cover tall and turned, open wide and partly open; fold with a tip showing.
- **Source:** `popoverTip(_:arrowEdge:action:)`, `Tips.showAllTipsForTesting()` and `Tips.resetDatastore()` references (fetched 2026-10-23); TT463 5:23.

---

## Input

### E-INPUT-1. Keyboard frames in screen space `[General; seen in a shipping app (Mitamaton)]` (L1, A10)
- **Case:** a composer placed from `keyboardFrameEndUserInfoKey`, compared with a geometry assumed to be the full screen, or a keyboard top cached across a resize.
- **Breaks:** the frame is "in the screen's coordinate space" and Apple says it "might be different when your app isn't full screen in Split View". On the Duo that also covers the app below a pinned PiP video. A cover-based keyboard top is applied after unfolding. Mitamaton's chat sheet assumed `.ignoresSafeArea()` made its geometry "the true full screen". Converting via `UIScreen.main.coordinateSpace` misplacing the composer is inferred.
- **Detect:** `rg -n -e 'keyboardFrameEndUserInfoKey' -e 'UIKeyboardFrameEnd' -e 'keyboardWillChangeFrame' -e 'keyboardWillShow' -e 'keyboardTopY' -e 'IQKeyboardManager'` in every language; log window frame in screen space against the keyboard end frame in pinned-PiP and right-half Split View.
- **Fix:** `view.keyboardLayoutGuide` (iOS 15) or SwiftUI's keyboard safe area / `.safeAreaInset(edge: .bottom)`. If the notification stays, `view.convert(endFrame, from: window.screen.coordinateSpace)` on every change; clear any cached value on window resize.
- **Verify:** focus each input on the cover, unfold with the keyboard up, fold back; repeat in right-half Split View and below a pinned video: input bottom on the keyboard top in raw `--display=3` captures and logs.
- **Source:** keyboardFrameEndUserInfoKey and keyboardLayoutGuide docs (fetched); TT464 3:17; TT466 3:22; LAB1013 1:01:13.

### E-INPUT-2. Accessory bars, overflow and search with the keyboard up `[General]` (L2)
- **Case:** Apple did give keyboard guidance: accessory bars stay attached to the keyboard rather than moving to the vertical bar; the keyboard (and PiP in open portrait) makes the bar overflow; a focused search field over the keyboard adapts width and position as the device folds.
- **Breaks:** typing controls built as `.bottomBar` toolbar items end up in the side column; a key action vanishes into overflow only while typing; a hand-placed search field crosses the band.
- **Detect:** `rg -n -e 'inputAccessoryView' -e 'placement: *\.keyboard' -e 'ToolbarItemGroup\(placement: *\.bottomBar' -e '\.searchable' -e 'UISearchController'`; focus every field on the cover and open wide and see which items survive.
- **Fix:** typing controls in `inputAccessoryView` or `ToolbarItemGroup(placement: .keyboard)`; `visibilityPriority(.high)` on actions needed while typing; system search instead of a fixed-position field.
- **Verify:** each form with the keyboard up in all six poses: accessories on the keyboard; partly open, search on one side of the band; high-priority items visible.
- **Source:** TT462 10:01, 11:50-11:55; TT463 4:55-5:09.

### E-INPUT-3. Focus, drafts and IME through a fold `[General]` (L3, T24)
- **Case:** a fold while a field is first responder; focus and draft live in a view that a branch swap or `.id(size)` rebuilds. Long-lived sims have a hardware keyboard attached, so the software keyboard never shows in tests.
- **Breaks:** in a swapped tree the keyboard drops, `@FocusState` resets, the draft goes (T7). Whether a stable tree keeps first responder and IME composition through the resize is (unverified). The keyboard moving between displays on fold is an expectation, not measured. Taps computed before the keyboard appeared miss.
- **Detect:** `rg -n -e '@FocusState' -e 'becomeFirstResponder' -e 'focused\(' -e '\.id\(.*(size|width|isSplit|sizeClass)'`; fold tests that open a form but never assert `app.keyboards.count > 0`; the sim's Connect Hardware Keyboard setting.
- **Fix:** one tree (T7); drafts in a model or `@SceneStorage`; restore focus from the model if lost. Run form fold proofs on an erased Duo or with the hardware keyboard disconnected; diagnose missed taps from the xcresult Synthesized Event attachment.
- **Verify:** type "abc" (and a Japanese IME composition), fold, unfold: keyboard up (or back) on the same field, text intact, field above the keyboard on the cover.
- **Source:** LAB0917 19:11; LAB1013 39:30-42:13; observed in Mitamaton's simulator runs (hardware keyboard hides the software keyboard).

### E-INPUT-4. Drag and drop and pasteboard between two instances or apps in Split View `[General]` (review)
- **Case:** the inner display runs two apps side by side, or two instances of yours if it supports multiple scenes. People drag text, images and links across the middle, and copy in one half to paste in the other.
- **Breaks:** Apple's Duo material does not cover this (`apple-guidance.md` section 12), so the general rules apply. `UIDragSession.localContext` is "visible only to the app in which the drag activity begins", so a drop handler that needs it rejects drags from the other app; a drag that offers only a custom in-process type gives the other app nothing it can accept. Since iOS 14 the system notifies the user when an app reads general-pasteboard content from another app without user intent; an app that reads the pasteboard whenever it becomes active may show that notice each time focus moves between halves (unverified, test it). Drop targets placed under the column or on the band are hard to hit. A drag cannot start on one display and end on the other (touch on both displays at once is not supported, LAB0917 5:23); what a fold does to a drag in progress is (unverified).
- **Detect:** `rg -n -e 'UIDragInteraction' -e 'UIDropInteraction' -e 'onDrop\(' -e 'dropDestination' -e 'draggable\(' -e 'localContext' -e 'NSItemProvider' -e 'UIPasteboard' -e 'PasteButton' -e 'didBecomeActive.*[Pp]asteboard'`.
- **Fix:** offer public representations (plain text, URL, image) beside any custom type (`Transferable` or `NSItemProvider`); treat `localContext` as an optimisation only; read the pasteboard on user intent (`PasteButton`, the paste command), and test for content with the `has...` properties instead of reading it; keep drop targets clear of the column and the band.
- **Verify:** inner display, Split View with Notes, Files or Safari on the other side (drag the app by its home indicator in Device Hub): drag text, an image and a link each way, then copy and paste each way; repeat with your app on the left and on the right; with two instances of your app if supported. Watch for a paste notice when switching halves.
- **Source:** UIKit "Drag and drop" (inter-app drag since iOS 15 on iPhone; "The system handles all security aspects of inter-app drag and drop"), `UIDragSession.localContext`, `UIPasteboard`, `PasteButton`, `draggable(_:)` references (all fetched 2026-10-23); LAB0917 5:23; TT461 7:44 (Split View).

### E-INPUT-5. Hardware keyboard shortcuts, pointer and Full Keyboard Access `[General]` (review)
- **Case:** `UIKeyCommand`s, SwiftUI `.keyboardShortcut`, `pressesBegan` handlers, pointer effects (`UIPointerInteraction`, `.hoverEffect`), and Full Keyboard Access users moving focus with Tab and arrows.
- **Breaks:** Apple's Duo material does not cover hardware keyboards (`apple-guidance.md` section 12). Per the docs, `.keyboardShortcut` targets are found by a leading-to-trailing, depth-first search of "the active scene, and then the command groups", and "If multiple controls are associated with the same shortcut, the first one found is used"; `UIKeyCommand`s come from the responder chain and fire "only for devices with an attached hardware keyboard". On the Duo, a shortcut attached to a view that leaves the tree on the cover (E-PRES-1) stops working there, and with two instances the shortcut acts on the active scene only. Whether a shortcut on a toolbar item still fires while that item is in the column's overflow is (unverified, test it). Whether the Duo supports a hardware pointer outside AssistiveTouch is (unverified). Custom chrome in the bar region (E-BARS-3) may be skipped by Full Keyboard Access if it is not focusable (unverified).
- **Detect:** `rg -n -e 'UIKeyCommand' -e 'keyCommands' -e 'keyboardShortcut' -e 'pressesBegan' -e 'UIPointerInteraction' -e 'hoverEffect' -e 'onHover' -e 'focusable\(' -e 'focusSection'`.
- **Fix:** put shortcuts on commands (SwiftUI `.commands`, UIKit main-menu builder or a scene-level responder) rather than on views that move or leave; one owner per shortcut per scene; make custom column items real buttons with labels so focus and Full Keyboard Access reach them (`accessibilityInputLabels`: "Voice Control and Full Keyboard Access use the input labels").
- **Verify:** in the simulator the Mac keyboard acts as a hardware keyboard (I/O > Keyboard > Connect Hardware Keyboard); run every shortcut on the cover, open wide, with its item in overflow, and with two instances. Turn on Full Keyboard Access in the simulator's Settings > Accessibility > Keyboards and Tab through the side column and content on the cover and open. Then disconnect the hardware keyboard again, or later software-keyboard tests go blind (E-INPUT-3).
- **Source:** `UIKeyCommand`, `keyboardShortcut(_:modifiers:)`, `UIPointerInteraction`, `focusable(_:)`, `accessibilityInputLabels(_:)` references (all fetched 2026-10-23, none Duo-specific); `apple-guidance.md` section 12.

---

## Accessibility

### E-A11Y-1. VoiceOver order and focus across the fold `[General]` (T18)
- **Case:** list then detail on two sides of the fold; panes appearing, disappearing or moving to the other display. VoiceOver runs on both displays.
- **Breaks:** focus stays on the board while a detail opens unannounced, or jumps to the top after a fold (Mitamaton's audit, case study section 16).
- **Detect:** custom split hosts with none of `AccessibilityFocusState`, `accessibilityElement(children:`, `accessibilitySortPriority`, `UIAccessibility.post`.
- **Fix:** `.accessibilityElement(children: .contain)` on panes; sort priority where layout crosses the fold; post `.layoutChanged` on pane changes; `.accessibilityAction(.escape)`. Automate with `XCUIDevice.shared.voiceOverService` (iOS 27.0: `enable()`, `moveForward()`, `currentSpeech()`, which return an `XCUIVoiceOverOutput` object, not a string). Whether it works across a fold on the sim is unmeasured.
- **Verify:** UI test reads the utterance sequence on the open Duo, folds via the handshake, asserts current speech still names the detail.
- **Source:** XCUIVoiceOverService.h (Xcode 27.1 SDK); forum 847644; LAB0916 29:42.

### E-A11Y-2. In-tree panes get no sheet accessibility; Reduce Motion cut-in `[General; seen in a shipping app (Mitamaton)]` (D26)
- **Case:** a detail beside its source or full screen on the cover is not a presentation.
- **Breaks:** no focus move, no announcement, no escape, no focus return; the surface under a full-screen detail stays in the tree. An `.opacity` transition riding the host's animation never plays when that animation is nil under Reduce Motion, so the pane cuts in.
- **Detect:** near the host `rg -n -e 'accessibilityAction\(\.escape' -e 'AccessibilityFocusState' -e 'accessibilityHidden'`; `rg -n -e 'transition\(.*opacity' -e 'animation\(reduceMotion \? nil'`.
- **Fix:** pane `.accessibilityElement(children: .contain)`; `.accessibilityAction(.escape)` calls the close route; focus into the pane on open, back to the source about 350 ms after close (only in-tree, E-A11Y-3); hide the covered surface; under Reduce Motion give the transition its own `.opacity.animation(.easeOut(duration: 0.2))` while the reflow has none. Escape and focus return are good practice, not HIG text.
- **Verify:** a human VoiceOver walk on the Duo (the sim cannot say where focus lands); UI test that covered cards leave the tree; Reduce Motion capture sequence.
- **Source:** case study section 16.

### E-A11Y-3. `accessibilityFocused` blanks `ImageRenderer` `[General; seen in a shipping app (Mitamaton)]` (D08)
- **Case:** `.accessibilityFocused($focus)` on content also rendered with `ImageRenderer` (share images, widgets, render or snapshot tests).
- **Breaks:** observed in Mitamaton under Xcode 27.1, not documented: the body never ran and the render was blank; `UIHostingController` renders were fine. The live app looks fine.
- **Detect:** `rg -n -F '.accessibilityFocused('`; `rg -n -F 'ImageRenderer'` (tests included); views reachable from both.
- **Fix:** attach the modifier only inside the live host, gated on something fixed per host (an environment value it sets), never runtime state (E-LIFE-11).
- **Verify:** guard test renders the content with `ImageRenderer` outside a host and fails when the body never runs (Mitamaton keeps one such guard test).
- **Source:** case study section 17. Apple's ImageRenderer page does not mention accessibility focus (fetched 2026-10-23).

### E-A11Y-4. Dynamic Type caps on wide reading surfaces `[General; seen in a shipping app (Mitamaton)]` (D32)
- **Case:** `dynamicTypeSize(...xxLarge)`, `Font.custom(_, size:)` without `relativeTo:`, or a two-step ladder on detail, reader and editor text.
- **Breaks:** AX sizes stay small across 951 pt where Apple says large sizes look best (opened); a Larger Text claim on the Accessibility Nutrition Label becomes false. (Mitamaton: capped at xxLarge.)
- **Detect:** `rg -n -e 'dynamicTypeSize\(\.\.\.' -e 'Font\.custom\([^)]*size:' -e '\.system\(size:' -e 'min\([^)]*[Ss]caled' -e '@ScaledMetric'` and look for missing `relativeTo:` and capped scaled metrics; classify each hit as reading text or as titles and icons.
- **Fix:** let detail, reader and editor text scale (`.custom(face, size:, relativeTo: .body)`); cap line length with the readable content guide; keep fixed ladders only on dense cards.
- **Verify:** AX5 captures on the open Duo and a 390 pt iPhone, with a truncation oracle where available.
- **Source:** LAB0916 29:14-29:42; case study section 16.

### E-A11Y-5. Voice Control "show numbers" and Switch Control on side-column items `[General]` (review)
- **Case:** icon-only items in the side column (the column shows icons and reveals labels only while touched, LAB0917 6:53), custom chrome placed in the bar region (E-BARS-3), and panes on both sides of the fold.
- **Breaks:** Voice Control's "show names" and spoken commands use the accessibility label or input labels; an icon-only custom button with no label can only be reached by number or grid. Switch Control scans in accessibility order, so a custom column or a pane on the far side of the fold can come last or be skipped. An Apple engineer said Voice Control and the other assistive technologies are on the device and VoiceOver works on two displays at once (LAB0916 29:44-30:01); how Voice Control numbers and Switch Control scanning treat the vertical column is (unverified, test it). XCUI sees side tab items as plain buttons (E-TEST-12), so assistive tech may too (unverified).
- **Detect:** `rg -n -e 'Image\(systemName:' -e 'UIBarButtonItem\(image:'` in toolbar and chrome files without a nearby `accessibilityLabel`, `Label(` or title; `rg -c 'accessibilityInputLabels'`; custom column views without `accessibilityElement(children: .contain)` or sort priority.
- **Fix:** every bar item a `Label("Title", systemImage:)` or titled item (the title drives names even where only the icon shows); `accessibilityInputLabels` for short spoken names; group panes and order the column with `accessibilitySortPriority` (E-A11Y-1).
- **Verify:** Accessibility Inspector (Xcode > Open Developer Tool) pointed at the Duo simulator: every column item has a label, on the cover and open. On a physical Duo: Voice Control "Show numbers" and "Show names" on the cover and open wide, and Switch Control item scanning through the column and both panes. Whether Voice Control runs in the simulator is (unverified).
- **Source:** `accessibilityInputLabels(_:)` reference ("Voice Control and Full Keyboard Access use the input labels", fetched 2026-10-23); LAB0916 29:44-30:01; LAB0917 6:53.

### E-A11Y-6. Bold Text, Button Shapes, Increase Contrast and Dark Mode, cover and inner `[General]` (review)
- **Case:** the condition lists named Dynamic Type, Reduce Motion, Reduce Transparency and RTL, not these four display settings.
- **Breaks:** Bold Text widens every label, so text that just fits at 466 pt truncates on the cover. With Button Shapes on, "interactive custom controls such as buttons should be drawn in such a way that their edges and borders are clearly visible", which custom column chrome may not do. Increase Contrast and Dark Mode change system materials and colours; hard-coded colours in custom chrome, scroll-edge fills (E-BARS-1) and toasts do not follow, and contrast against content behind the column can drop. How the vertical column's own material responds to Increase Contrast is (unverified, test it); for Reduce Transparency it gets an opaque plate (LAB0916 30:27).
- **Detect:** `rg -n -e 'legibilityWeight' -e 'isBoldTextEnabled' -e 'colorSchemeContrast' -e 'accessibilityShowButtonShapes' -e 'buttonShapesEnabled' -e 'Color\(red:' -e 'UIColor\(red:' -e 'Color\(\.sRGB'` in chrome, toast and paywall files. (UIKit's `buttonShapesEnabled` is marked deprecated in iOS 26.1 on its reference page; check the SDK for the current symbol.)
- **Fix:** semantic and asset-catalog colours with high-contrast and dark variants; text that wraps or scales instead of fitting by a hair; custom buttons that show an edge when Button Shapes is on (SwiftUI `accessibilityShowButtonShapes`, which its reference page marks for a future deprecation); read `legibilityWeight` only if a custom font needs a heavier face.
- **Verify:** `xcrun simctl ui <device> appearance dark` and `xcrun simctl ui <device> increase_contrast enabled` (read each value first and restore it in a trap); Bold Text and Button Shapes in the simulator's Settings > Accessibility > Display & Text Size. Capture changed screens on the cover tall, open wide and partly open with each on; run `performAccessibilityAudit(for: [.contrast, .textClipped])` (E-TEST-20). Note: an app with its own stored appearance setting can override `simctl ui appearance` (seen in Mitamaton).
- **Source:** `isBoldTextEnabled`, `legibilityWeight` ("reflects the value of the Bold Text display setting"), `accessibilityShowButtonShapes` (quoted), `buttonShapesEnabled`, `colorSchemeContrast` references (all fetched 2026-10-23, none Duo-specific); `simctl ui` help (Xcode 27.1); LAB0916 30:27.

---

## Archetypes

### E-ARCH-1. Games: aspect, letterbox, controls, state `[General]` (A05, L15)
- **Case:** fixed design resolution with `scaleMode = .aspectFill`/`.fill`, letterboxing, HUD at scene corners, scene rebuilt on size change.
- **Breaks:** HIG: playable in every pose, fill the screen, prefer changing aspect ratio over letterboxing, add artwork to unavoidable bars. Aspects 0.69, 1.42, 0.70 crop HUDs or stretch art; a rebuilt scene resets the game.
- **Detect:** `rg -n -e 'scaleMode *= *\.' -e 'SKScene\(size' -e 'didChangeSize' -e 'drawableSizeWillChange' -e 'SCNView' -e 'GCVirtualController'`; Unity `ProjectSettings.asset` orientation keys.
- **Fix:** resize in `didChangeSize`/`viewWillLayoutSubviews` (`.resizeFill`); HUD from the safe area on every change; update projection on resize; game model outside the view; text and control sizes constant.
- **Verify:** play in all six poses and through live folds and Split View: no stretch, no cut HUD, no bare bars, state unchanged.
- **Source:** HIG Best practices (fetched); LAB1013 38:10, 58:05-59:03.

### E-ARCH-2. Metal and engine drawables sized once `[General]` (A06)
- **Case:** a custom `CAMetalLayer`/`GLKView`/engine view sets `drawableSize` or `contentsScale` from `UIScreen.main` or the launch size; `MTKView` with `autoResizeDrawable = false` (it defaults to true).
- **Breaks:** after a fold the drawable keeps the other screen's size: blur, stretch, crop, wasted fill rate. Engines reading the main screen render for the cover (unverified for Unity specifically).
- **Detect:** `rg -n -e 'drawableSize *=' -e 'contentsScale *=' -e 'autoResizeDrawable *= *false' -e 'nativeScale' -e 'mainScreen\]'` across `.swift`, `.m`, `.mm`, `.cpp` and Unity's `Classes/` trampoline.
- **Fix:** `drawableSize = bounds.size * traitCollection.displayScale` in `layoutSubviews` or the size callback; recreate size-dependent targets.
- **Verify:** log drawable and bounds after each pose: drawable = bounds x 3; compare sharpness on both displays.
- **Source:** TT461 4:16, 9:25 talk code; UIScreen.main deprecation.

### E-ARCH-3. Immersive full screen and corner controls `[General]` (A07)
- **Case:** status bar and bars hidden, edge-to-edge drawing, touch controls in the corners.
- **Breaks:** the cover's outer camera is always in a corner; corners away from the hinge are rounder and some are new; full-inset placement wastes the 84 pt; in laptop pose controls cover gameplay.
- **Detect:** `rg -n -e 'prefersStatusBarHidden' -e 'persistentSystemOverlays' -e 'prefersHomeIndicatorAutoHidden' -e 'toolbarVerticalBehavior' -e 'ignoresSafeArea\(\)'`; joystick nodes placed from scene corners.
- **Fix:** corner-adapted safe area (`edgeInsets(for: .safeArea(cornerAdaptation:))`); keep off `.occlusion`; `ArrangementView(.overlay)` for handheld console layouts, or `.division` regions to move controls to the stable bottom half in laptop pose.
- **Verify:** cover tall and turned with the corner mask (`--mask=alpha`); partly open in book and laptop: controls off gameplay and band.
- **Source:** LAB0917 41:21-42:12; LAB1013 36:39-37:31, 58:41; LAB0916 56:04; LAB1015 22:36.

### E-ARCH-4. Video players that force rotation `[General]` (A08)
- **Case:** full screen forced landscape via `setValue(_:forKey: "orientation")`, `requestGeometryUpdate`, toggled supported orientations, controls laid out assuming the rotation happened.
- **Breaks:** the inner display ignores supported orientations, so the layout can stay portrait-shaped or go sideways on the cover (outcome of forcing on the inner display unverified). Controls overlay video in laptop pose.
- **Detect:** `rg -n -e 'forKey: *"orientation"' -e 'requestGeometryUpdate' -e 'AVPlayerLayer' -e 'videoGravity' -e 'AVPlayerViewController'`.
- **Fix:** `AVPlayerViewController`/`VideoPlayer` (AVKit adapts across poses); custom players size by window shape, `ArrangementView(.overlay)` for video top / controls bottom when partly open, `toolbarVerticalBehavior(.disabled)` as a stable per-screen choice; remove forcing hacks or gate them to the cover.
- **Verify:** enter full screen in all six poses and fold during playback.
- **Source:** TT461 3:30; LAB1015 38:34, 41:53; LAB0916 40:24; LAB1013 37:27.

### E-ARCH-5. Playback through a fold; PiP `[General]` (L16, A09)
- **Case:** an `AVPlayer` owned by a swapped view, pause in `onDisappear`/`viewWillDisappear`, `AVPlayerLayer` frame set once, no audio background mode, PiP unsupported.
- **Breaks:** video restarts, stops or plays invisibly; the layer stays cover-sized; audio cuts when a Split View close backgrounds the app; two instances share one audio route.
- **Detect:** `rg -n -e 'AVPlayer\(' -e 'AVPlayerLayer' -e 'VideoPlayer\(' -e 'onDisappear.*pause' -e 'viewWillDisappear' -e 'AVPictureInPictureController'`; Info.plist `UIBackgroundModes`.
- **Fix:** players in a model outside swapped views; layer fills bounds in `layoutSubviews`; pause on intent, not disappearance; audio mode only for real background playback; support PiP (people pin it in open portrait).
- **Verify:** play, fold, unfold, turn: no restart, fills its frame; Split View close: audio continues if it should, stops cleanly if not.
- **Source:** LAB0917 3:04, 3:52, 33:53; LAB1013 43:47, 45:19, 1:00:56.

### E-ARCH-6. Camera and scanner preview geometry `[General]` (A26)
- **Case:** `rectOfInterest` set once from `metadataOutputRectConverted`, overlay frames laid out at setup.
- **Breaks:** after a fold, rotation or display switch, preview size and gravity change, so the scan region and overlay no longer match (general AVFoundation behaviour on any preview resize; needs a device).
- **Detect:** `rg -n -e 'rectOfInterest' -e 'metadataOutputRectConverted' -e 'AVCaptureVideoPreviewLayer' -e 'previewLayer.frame *='`.
- **Fix:** recompute in `viewDidLayoutSubviews` after the layer's frame and rotation update (rotation coordinator); choose letterbox (photos) or fill (AR, scanning) deliberately.
- **Verify:** physical Duo: fold, unfold, rotate, scan a code at the overlay's edge each time; log rects.
- **Source:** CAM; LAB1013 19:00-20:01; TT465.

### E-ARCH-7. Web views and hybrid apps `[General]` (A14, L19)
- **Case:** `width=device-width` breakpoints, `viewport-fit=cover` with symmetric `env(safe-area-inset-*)`, JS `screen.width`, hinge-aware CSS, `contentInsetAdjustmentBehavior` wrong for the side inset, web-drawn app chrome.
- **Breaks:** 951 pt crosses tablet breakpoints; web UI under the column or a gap on the wrong side; web chrome never moves to the column. Viewport Segments / Device Posture support in Duo WebKit is unverified (an Apple engineer said WebKit supports hinge position, LAB1013 36:19; MDN's "unsupported in Safari" was not fetched).
- **Detect:** `rg -n -e 'WKWebView' -e 'viewport-fit' -e 'safe-area-inset' -e 'screen\.width' -e 'viewport-segment' -e 'Capacitor' -e 'Cordova'` over Swift and bundled `.html/.css/.js/.ts`.
- **Fix:** each `env(safe-area-inset-*)` side independently, or keep the web view inside the native safe area; test breakpoints at 466, 669, 951; wrap web screens in native `NavigationStack`; split natively for the fold until WebKit support is confirmed. `SFSafariViewController` needs nothing.
- **Verify:** representative pages in all six poses and Split View left; log `innerWidth`, `screen.width` and the inset values; test `@media (horizontal-viewport-segments: 2)` partly open.
- **Source:** LAB1013 36:06-36:27; LAB0916 38:48-39:26; HIG asymmetry. Capacitor-specific detection is in E-ARCH-12.

### E-ARCH-8. Cross-platform hosts draw their own bars `[General]` (A15)
- **Case:** react-navigation JS bottom tabs, Flutter `BottomNavigationBar`/`CupertinoTabBar`, MAUI renderers; layout from `Dimensions.get('screen')`, `display.size`, Unity `Screen.width`.
- **Breaks:** framework-drawn bars are custom bars: they never go into the column, so the app shows horizontal bars beside an empty 84 pt column; screen dimensions may report the cover; the JS/Dart side cannot read the vertical edge or regions without a bridge. (Framework specifics inferred.)
- **Detect:** package.json `@react-navigation/bottom-tabs` vs native tabs; `rg -n -e "Dimensions.get\('screen'\)" -e 'useWindowDimensions' -e 'MediaQuery.of\(context\).size' -e 'displayFeatures' -e 'Screen.safeArea'`.
- **Fix:** native containers where possible (RN native-stack plus native tabs over `UITabBarController`); otherwise a native module exposing `toolbarVerticalEdge`, per-side insets and `.division` regions, placing the bar in the bar region; window, not screen, dimensions; the framework's iOS 27.1 release.
- **Verify:** cover and open wide: bar placement matches the HIG or is a stated exception; bridged values update live on fold, rotation, Split View.
- **Source:** PREP; TT462 2:39; LAB1013 1:01:41-1:02:51; LAB0916 15:11. Kotlin/Compose Multiplatform, Skip, Capacitor and .NET MAUI detection: E-ARCH-12.

### E-ARCH-9. Maps `[General]` (A25)
- **Case:** selected pin or user location centred on the map; full-width bottom sheet; custom map buttons in fixed corners.
- **Breaks:** held wide and partly open, the map centre is the band (fold mid 475.5 pt; inferred from geometry); a default sheet covers the open map; buttons under the column or camera occlusion.
- **Detect:** `rg -n -e 'MKMapView' -e '\bMap\(' -e 'import MapKit' -e 'setCenter' -e 'setRegion' -e 'MapCameraPosition' -e 'safeAreaPadding'` (a bare `Map\(` also matches `flatMap(` and `compactMap(`).
- **Fix:** offset the focus point into a half (`MKMapView.layoutMargins`, SwiftUI `.safeAreaPadding`); `presentationPlacement(.leading)` sheets (Maps, Find My); system map controls.
- **Verify:** select a pin partly open held wide: lands in a half; open Duo and closed landscape: leading sheet, map visible.
- **Source:** LAB0917 47:29-48:14; LAB0916 33:30; RR.

### E-ARCH-10. Drawing and canvas apps `[General]` (A13)
- **Case:** strokes stored in view-relative or normalised coordinates; canvas sized to the screen; stroke commit only on `touchesEnded`; fixed-corner palettes; `.pencilOnly`.
- **Breaks:** strokes stretch on resize; a mid-stroke fold may cancel touches (unverified); palette under the column or fold; Pencil is not supported at launch, so `.pencilOnly` paths are unreachable (PKCanvasView's default already allows fingers on iPhone).
- **Detect:** `rg -n -e 'PKCanvasView' -e 'PKToolPicker' -e 'touchesCancelled' -e 'drawingPolicy' -e '\.pencilOnly' -e 'UIPencilInteraction'`.
- **Fix:** strokes in canvas coordinates; handle `touchesCancelled`; palettes by corner-adapted insets, off `.division`; allow finger input.
- **Verify:** draw, fold mid-stroke, check stroke and undo; position before and after resize; palette in six poses.
- **Source:** LAB1015 37:33; LAB1013 38:26; LAB0917 5:23.

### E-ARCH-11. Motion-sensor features `[General]` (L14)
- **Case:** a level, compass, tilt parallax or AR overlay.
- **Breaks:** "level" and "north" have no single meaning in tent, laptop and book poses (Apple's Measure team found this hard). Do not claim sensor frames map to the wrong display: an Apple engineer said the system handles the mapping ("it's magic you don't need to worry about"), unverified on a device.
- **Detect:** `rg -n -e 'CMMotionManager' -e 'deviceMotion' -e 'CMAttitude' -e 'headingOrientation' -e 'ARSession'`.
- **Fix:** decide what the reading means per pose (use hinge status where needed); recompute on resize.
- **Verify:** physical device only: compare readouts with a reference phone in each pose.
- **Source:** LAB1015 59:17-59:55; LAB0916 58:04-58:23; LAB1013 11:46.

### E-ARCH-12. Kotlin/Compose Multiplatform, Skip, Capacitor, .NET MAUI `[General]` (review)
- **Case:** cross-platform stacks beyond React Native and Flutter (E-ARCH-8) and plain web views (E-ARCH-7). Apple's Duo material names none of these; every framework claim below is inferred from how each stack draws its UI and is (unverified) on a Duo.
- **Breaks:** Compose Multiplatform draws its own UI inside a `UIViewController`, so Compose bottom bars and top bars are custom bars that never move to the column (as E-ARCH-8), and Kotlin/Native code in `iosMain` can read `UIScreen.mainScreen` directly. Skip transpiles SwiftUI for Android while the iOS app is native SwiftUI, so the iOS side follows the SwiftUI rules, but `#if SKIP` / `#if os(Android)` branches and Android-shaped layout choices can leak into shared views. Capacitor is a `WKWebView` host: E-ARCH-7 applies, plus its native shell may own the window in the AppDelegate (E-LIFE-1). .NET MAUI and older Xamarin apps read the main screen from C# (`UIScreen.MainScreen`, `DeviceDisplay.MainDisplayInfo`), and whether a MAUI Shell tab bar is a native `UITabBarController` that moves to the column is (unverified, test it).
- **Detect:** Compose: `rg -n -e 'ComposeUIViewController' -e 'org.jetbrains.compose' -e 'kotlin\("multiplatform"\)' -g '*.kts' -g '*.kt'`, and `rg -n 'UIScreen.mainScreen' -g '*.kt'`. Skip: `rg -n -e 'import SkipUI' -e 'import SkipFuse' -e '#if SKIP' -e '#if os\(Android\)'` and a `Skip/skip.yml`. Capacitor: `capacitor.config.json` or `.ts`, `@capacitor/core` in package.json, `ios/App/App/AppDelegate.swift`. MAUI: `<UseMaui>true</UseMaui>` in a `.csproj`, `Platforms/iOS/`, `rg -n -e 'UIScreen.MainScreen' -e 'DeviceDisplay.MainDisplayInfo' -e 'Window.Bounds' -g '*.cs'`.
- **Fix:** the same as E-ARCH-8: native containers where the framework allows them, otherwise a native bridge that exposes `toolbarVerticalEdge`, per-side insets and `.division` regions; window, not screen, dimensions; each framework's release built with the iOS 27.1 SDK and the scene lifecycle. For Skip, audit shared views for Android-only branches that change iOS layout.
- **Verify:** as E-ARCH-8: cover and open wide, bar placement matches the HIG or is a stated exception; bridged values update live on fold, rotation and Split View; the main-screen greps above return 0 in first-party code.
- **Source:** inferred; Apple sources for the shared rules are PREP and TT462 2:39 (custom bars), TN3187 (scene lifecycle), UIScreen.main deprecation (fetched). No framework page was fetched for this entry.

---

## Testing

### E-TEST-1. The 27.1 runtime has only the Duo `[General]` (D10, T01)
- **Case:** `xcrun simctl list runtimes` shows iOS 27.1 (24A94401) supporting only "iPhone Duo"; a non-Duo 27.1 sim cannot be created.
- **Breaks:** a red Duo test cannot be split into "Duo host" vs "iOS 27.1"; "run the iPhone half on 27.1" fails at creation; 27.1 branches on non-Duo iPhones (for example `.verticalPreferred` staying horizontal) go unverified.
- **Detect:** `xcrun simctl list runtimes -j | jq '.runtimes[] | select(.version|startswith("27.1")) | .supportedDeviceTypes[].name'`.
- **Fix:** control pin on iOS 27.0 (fallbacks only); differential gate: same sha on the Duo and the 27.0 iPhone, each failure filed under one; a Duo-only red is a Duo-host artifact (record it, do not skip, do not "fix" app code). Unit-test 27.1 branch logic as pure functions. Release checklist line: "non-Duo iPhone on 27.1 unverified" until a device run. Recheck on later runtimes.
- **Verify:** the runtime listing beside the gate report.
- **Source:** measured 2026-10-23; Duo profile.plist minRuntimeVersion 27.1; case study section 14 (the differential run sorted 8 failures).

### E-TEST-2. Device detection by window width is wrong `[General; seen in a shipping app (Mitamaton)]` (D13, T02)
- **Case:** an early draft of this skill said to decide split vs single from `app.windows.firstMatch.frame.width`. A Pro Max held sideways is about 956 pt, wider than the open Duo's 951.
- **Breaks:** a landscape iPhone takes the Duo branch (fold tests skip, iPhone assertions never run). On a physical Duo `SIMULATOR_MODEL_IDENTIFIER` is unset, so a model check reading only it goes falsely red.
- **Detect:** `rg -n -e 'frame.width *> *[0-9]{3}' -e 'windows.firstMatch.frame' Tests/`.
- **Fix:** device by model: `ProcessInfo.processInfo.environment["SIMULATOR_MODEL_IDENTIFIER"] == "iPhone19,4"`, else `uname()` machine on a device (on the sim uname names the Mac); width only for the POSE, paired with the model; one shared helper. Skip messages name the device and window size.
- **Verify:** suite on a landscape iPhone and the Duo: each takes its own branch.
- **Source:** case study section 14.

### E-TEST-3. Fold proofs that pass by accident `[General; seen in a shipping app (Mitamaton)]` (D14, T03)
- **Case:** the sim boots closed and XCUITest cannot fold it, so pose-branching tests run only the cover branch and report green. Fold tests wait for an external driver; the driver lives outside the repo; handshakes time out silently; skips look like passes; early exits and unmatched stage filters print PASS.
- **Breaks:** "fold proofs green" cannot be reproduced; reverting the fix still yields skip plus success (Mitamaton: the first run folded nothing; a failed Open pose printed PASS with 5 of 7 stages never run; the split UI tests ran only their sheet branch).
- **Detect:** `rg -n 'XCTSkip' Tests/` near pose code; `git ls-files | rg -i 'fold|pose'` for the driver; the gate's skip-marker check; branch logging per test.
- **Fix:** commit the driver. Each test records which branch ran and checks the window really changed after the handshake. A fold skip throws `XCTSkip("FOLD_SKIPPED: ...")`; the gate greps the log body and FAILS unless an explicit opt-out is set. Device-level skips (not a Duo) go before anything opens, unmarked. Handshake timeout is `XCTFail`. Verdict printed from an EXIT trap; every early exit records why; zero stages fails; FAIL exits non-zero.
- **Verify:** stubbed driver: no stage gives FAIL rc 1; a failed pose FAIL rc 1; all stages PASS (n) rc 0; gate without the driver prints FOLD_SKIPPED and fails.
- **Source:** case study section 14.

### E-TEST-4. Device Hub window missing, or the wrong Device Hub `[General]` (D15, T05)
- **Case:** Device Hub's per-device window vanishes while the sim stays Booted; with several Xcodes installed, `open -a DeviceHub` or Spotlight can launch another one. (That an older Device Hub lacks posture controls is a community report, forum 847556, unconfirmed.)
- **Breaks:** no pose button can be pressed, so fold runs skip; plain `open -a` steals the user's Space.
- **Detect:** Quartz `CGWindowListCopyWindowInfo`: a window whose `kCGWindowName` matches the device AND `kCGWindowIsOnscreen`; `ps -o command= -p <pid>` for the running bundle path.
- **Fix:** `open -g -a "$(xcode-select -p)/../Applications/DeviceHub.app"` (background, safe while running; window back in about 5 s); check the window before every fold run and print FOLD_SKIPPED if missing. Never quit Device Hub. (The claim that Device Hub re-boots a sim after a by-UDID shutdown is unverified and left out.) If Device Hub "stops answering" (System Events or an Accessibility pose driver sees zero windows while Quartz still lists them with `kCGWindowIsOnscreen` false), check first whether its windows sit on another desktop (Space): activating Device Hub brought them back and they answered within about 1 s (measured 2026-10-23). Activating switches the visible desktop, so on a shared Mac ask the person at the keyboard first.
- **Verify:** window present before stage 1; a press changes the app's window size.
- **Source:** observed in Mitamaton's fold runs (case study section 14).

### E-TEST-5. Accessibility pose-press failure modes `[General]` (T06)
- **Case:** when a script presses Device Hub's pose controls through macOS Accessibility, AXPress returns success when the press is sent, not when the pose finishes; window titles are "<name> – iOS <runtime>" (an en dash), so two same-named Duos are indistinguishable; the caller needs Accessibility (and Screen Recording for captures); a first-match press can fold a Duo another test run is using.
- **Breaks:** a "successful" press with the device unchanged or mid-animation; the wrong device folds; a new terminal cannot press at all.
- **Detect:** `xcrun simctl list devices | grep -c 'iPhone Duo ('`; list Device Hub's windows by title; AX errors in output.
- **Fix:** unique names (`xcrun simctl rename <UDID> "iPhone Duo (<run-name>)"`); always target the exact window from your device's name and runtime; wait for the window size to change (E-TEST-6); preflight AX and Screen Recording grants and fail with a named reason.
- **Verify:** with two Duos booted, a press with no target window refuses; with one, only the target changes (`--display=3` goes from black to content).
- **Source:** observed while scripting the Device Hub pose controls (case study section 14). Apple offers no pose API (LAB1013 33:52).

### E-TEST-6. Settling after a pose `[General; seen in a shipping app (Mitamaton)]` (T07)
- **Case:** fixed `sleep 4-6` after a press, or a handshake that times out and carries on (Mitamaton's 90 s loop falls through).
- **Breaks:** assertions read pre-fold frames; taps use old hit points; a timed-out handshake acts as if folded. The Duo sim is heavy (forum 847137).
- **Detect:** `rg -n -e 'sleep [0-9]' -e 'Thread.sleep'` around pose presses; handshake helpers without a failure.
- **Fix:** `XCTNSPredicateExpectation` on the window frame differing from pre-fold, then on a DEBUG accessibility value the app publishes once layout settles (window size plus fold region); re-resolve elements; timeout = `XCTFail`; scripts poll the target display's capture until non-black.
- **Verify:** fold proof on a loaded machine does not flake; a missing ack fails with a timeout message.
- **Source:** case study section 14; forum 847137.

### E-TEST-7. Parallel testing clones `[General]` (T08)
- **Case:** Xcode's parallel testing runs on cloned simulators that boot closed. Whether Device Hub can pose a clone is unmeasured.
- **Breaks:** the handshake folds the original while the test runs in a clone (expected).
- **Detect:** test plan `parallelizable = YES`; xcodebuild without `-parallel-testing-enabled NO`; "Clone" in the xcresult device name.
- **Fix:** pose and fold stages with `-parallel-testing-enabled NO` (and `-disable-concurrent-destination-testing`) against the UDID you drive; parallel only for pure suites.
- **Verify:** `xcrun xcresulttool get test-results summary` names your UDID.
- **Source:** precaution taken in Mitamaton's fold driver; not measured.

### E-TEST-8. xcodebuild success markers and pruned results `[General; seen in a shipping app (Mitamaton)]` (T09)
- **Case:** `test-without-building` ends with `** TEST EXECUTE SUCCEEDED **`; Xcode prunes derived-data result bundles; the post-run `simctl diagnose` can hold xcodebuild up to 10 minutes; `test-without-building` shuts down a sim it booted.
- **Breaks:** a runner grepping `** TEST SUCCEEDED **` fails every passing stage; failure hierarchies vanish; the next press collides with a parked diagnose.
- **Detect:** scripts matching `TEST SUCCEEDED` without `EXECUTE`; no per-stage `-resultBundlePath`; no `-collect-test-diagnostics never`.
- **Fix:** match `\*\* TEST (EXECUTE )?SUCCEEDED \*\*` and the exit status captured before any pipe; per-stage `-resultBundlePath`; `-collect-test-diagnostics never` between poses; boot the Duo before the run.
- **Verify:** a passing stage reports PASS; a failing one keeps its xcresult with the UI hierarchy; xcodebuild exits promptly.
- **Source:** case study section 14; the diagnose wait was observed in Mitamaton's test runs.

### E-TEST-9. Geometry tests fed window-space LTR inputs `[General; seen in a shipping app (Mitamaton)]` (T17)
- **Case:** synthetic regions given as window coordinates for a full-width LTR view with the column trailing.
- **Breaks:** frames are in the view's own space, can be negative or past its size, mirrored in RTL, and an inactive division can report width 0; a sub-view offset by a column or pane misplaces the split; RTL lands it on the wrong half. Mitamaton had no RTL or leading-column test (case study section 16).
- **Detect:** review test inputs for non-zero origins, overflow, RTL, leading 84, width-0 inactive, horizontal active fold, no regions.
- **Fix:** add those cases; map `ReservedRegion` to plain `(frame, isActive)` at the boundary so tests need no SDK type (whether it has a public init is unverified).
- **Verify:** each case fails against a deliberately naive implementation.
- **Source:** RR; TT463 7:27; case study section 16.

### E-TEST-10. Render tests inherit the cover's insets `[General; seen in a shipping app (Mitamaton)]` (D11, T15)
- **Case:** render or snapshot tests mount a `UIWindow` plus hosting controller in the test host's first scene; on the Duo the host sits on the cover (right 84, bottom 34).
- **Breaks:** content centred in the shrunken region and shifted (a 200 pt card at x = -42), clipped mid-glyph; 8 Duo-only failures that read as regressions.
- **Detect:** `rg -n -e 'UIWindow\(windowScene' -e 'connectedScenes.first' -e 'UIHostingController\(rootView' Tests/`; Duo-only failures with edge-cut text.
- **Fix:** `hostingController.safeAreaRegions = []` (or a mount with no scene insets), fixed window frame; inject insets for both column edges when rendering Duo layouts. In Mitamaton the fix is queued, not yet proven: confirm the Duo-only failures pass while the 27.0 pin stays green.
- **Verify:** same output on the Duo and the 27.0 pin; removing the line turns the Duo red.
- **Source:** case study section 14.

### E-TEST-11. `ImageRenderer` blank cells `[General; seen in a shipping app (Mitamaton)]` (T16)
- **Case:** open layouts covered by views rendered at measured sizes. Two separate cases, both measured in Mitamaton and neither documented: (1) `ImageRenderer` did not draw a `ScrollView`'s contents. Layout, preferences and state all ran, only the rows were missing, and the cell came out flat with no placeholder (seen twice). Apple's ImageRenderer page says views "composited by Core Animation layers, such as more complex controls and containers" are left out and replaced by "a placeholder image"; it does not name `ScrollView`, so treat "blank scroll view" as Mitamaton's observation, not Apple's rule. (2) Content under `.accessibilityFocused` on presented content rendered nothing because its body never ran (E-A11Y-3); the fix was attaching the focus target in the live tree only.
- **Breaks:** baselines recorded blank compare equal forever, on exactly the scroll-heavy surfaces folds affect.
- **Detect:** baseline file sizes (Mitamaton: 6.5 KB flat vs 11-56 KB); `rg -n accessibilityFocused` on rendered content.
- **Fix:** render the content inside the scroll view (or render through a hosting controller in a window); focus modifiers only in the live tree; a guard failing on flat cells (single colour or under a size floor); read first baselines by eye.
- **Verify:** remove a label: the snapshot goes red; the guard fails on a blank scroll surface.
- **Source:** observed in Mitamaton's render tests; case study section 17; ImageRenderer doc (fetched 2026-10-23).

### E-TEST-12. Element lookup on the Duo `[General; seen in a shipping app (Mitamaton)]` (D12, D24, T25)
- **Case:** side-column tab items are plain buttons in generic containers, not a tab bar; a detail has two Closes; an identifier on a plain stack is handed to every child; a `.contain` nested in another `.contain` is flattened with its identifier; an identifier after a `safeAreaBar` modifier appeared to attach to the bar (inferred; hierarchy not captured); drags from an element centre under a band start in the bar.
- **Breaks:** `app.tabBars` is empty on the Duo; wrong Close or "multiple matches"; `firstMatch` returns a 24 pt icon; child ids vanish; a drag "on the board" opened another screen.
- **Detect:** `rg -n -F 'app.tabBars'` over the UI-test targets (take the folders from `xcodebuild -list`; they are not always `Tests/`); label-only queries ("Close", "Done"); identifiers on stacks; drags near the top edge. Also tests that use the tab bar as a geometry line, such as `app.tabBars.firstMatch.frame.minY` as the edge content must stay above: with the tabs in the column the tab bar does not exist, its frame is zero, and every scroll-until-above loop fails (trial, 2026-10-24).
- **Fix:** helper trying `app.tabBars.buttons[name]` then `app.buttons[name]`; a second helper for the bottom edge that falls back to `app.windows.firstMatch.frame.maxY` when there is no tab bar; unique identifiers on leaf controls and toolbar item content; `.accessibilityElement(children: .contain)` before a container's identifier, not nested; scroll-view identifiers before bar modifiers; drags from an explicit coordinate below the band; assert match counts.
- **Verify:** dump `app.debugDescription` at the state; the same tests pass on the cover, open Duo and a 27.0 iPhone.
- **Source:** case study section 14.

### E-TEST-13. Hand-picked test subsets after shared changes `[General; seen in a shipping app (Mitamaton)]` (D22)
- **Case:** a pre-commit subset chosen by hand after touching shared Duo plumbing (presenters, chrome, clips).
- **Breaks:** Mitamaton missed a render test and two source-scan tests twice, while its affected-test selector had picked them. Separately, that selector did not follow source-to-source references.
- **Detect:** compare the run's tests with the selector's picks; `rg -n 'contentsOf.*\.swift' Tests/` for source scanners.
- **Fix:** the selector is the floor; when a shared presenter changes also run every render or snapshot suite that draws it.
- **Verify:** break the presenter; the pick for that diff includes a failing test.
- **Source:** case study sections 9 and 14.

### E-TEST-14. Audit scope: non-Swift and binary code `[General]` (L12)
- **Case:** Objective-C, C++/Metal, React Native, Flutter, Unity, HTML/JS, closed xcframeworks (ads, analytics) reading the main screen.
- **Breaks:** every `--type swift` grep returns zero, so the audit "passes" while a vendor banner or the whole UI sizes to the cover.
- **Detect:** greps without `--type`; plus `mainScreen`, `Dimensions.get`, `MediaQuery`, `physicalSize`, `Screen.width`; `nm -u <binary> | grep -c UIScreen`; list Pods and SPM binary targets.
- **Fix:** first-party code as T1; vendors: a 27.1-ready release, pin, or contain (size its container yourself); cross-platform: window-relative APIs and the framework's 27.1 release.
- **Verify:** seed one Objective-C hit and one in a vendored binary; the audit must find both. The search itself must reach vendored folders: see E-TEST-27.
- **Source:** TT461 4:16, 9:19, 9:25; case study section 1.

### E-TEST-15. Orientation reset by pixels does not transfer `[General; seen in a shipping app (Mitamaton)]` (T10)
- **Case:** Mitamaton's fold driver script inferred rotation from window shape and green status-ring pixels in a corner. A fresh app has no such script, and the heuristic is tuned to one app. Poses themselves come from the Device Hub pose controls (E-TEST-5).
- **Breaks:** apps that hide the status bar, use green near corners, run dark with another canvas colour, or toggle the sidebar read wrong or UNKNOWN; "default pose" captures are turned.
- **Detect:** run the check on the new app at all four rotations.
- **Fix:** ask the app: a DEBUG build logs the window size (and `UIDevice.orientation`, unverified on the Duo); aspect ratio on the cover suffices for apps with landscape; pixel heuristic as fallback; fail on UNKNOWN.
- **Verify:** Rotate Right 0 to 3 times from a known state, light and dark, sidebar shown and hidden: reported orientation matches.
- **Source:** case study section 14.

### E-TEST-16. Corner masks in raw captures `[General]` (T11)
- **Case:** an early draft of this skill said only Device Hub window captures show the corner mask. `simctl io screenshot --mask=alpha|black` masks a raw capture. Cover corners: 8 at UL/LL (the hinge side, inferred), 59 at UR/LR; inner 55 all round.
- **Breaks:** corner checks done on sidebar-dependent crops; an inner-screen "corner clear" judgement does not transfer to the cover's rounder corners, which change side on rotation.
- **Detect:** corner checks via `screencapture -l` crops; `screenshot` without `--mask`.
- **Fix:** `xcrun simctl io <UDID> screenshot --display=<id> --mask=alpha out.png`; check both cover rotations. For video, alpha is unsupported (use `black` or `ignored`).
- **Verify:** `--mask=ignored` vs `--mask=alpha` of the same state: transparent corners at the profile radii.
- **Source:** `simctl io` help (Xcode 27.1); Duo capabilities.plist; LAB0917 41:21.

### E-TEST-17. Display ids and `XCUIScreen` identity `[General]` (T12)
- **Case:** profile displays: 1 (cover), 3 (inner, native orientation 270), plus 2 TVOut, 4 CarPlay, 5 Resizable. `XCUIScreen.screens` exposes no id; the first entry is the main screen (the cover).
- **Breaks:** hard-coded ids unverified; picking `XCUIScreen.screens[1]` by index; comparing 2853 x 2007 with a 2007 x 2853 baseline.
- **Detect:** `rg -n -e '--display=' -e 'XCUIScreen.screens\[' -e 'XCUIScreen\.main'`, plus any `simctl io ... screenshot` without `--display` (it captures the cover). `XCUIScreen.main.screenshot()` is the cover and is black while open; `XCUIApplication().screenshot()` captured the app's own display in one trial.
- **Fix:** `xcrun simctl io <UDID> enumerate` once per session (confirm names such as LCD and LCD-1 there); identify XCUIScreen entries by pixel size (1398 x 2034 cover, 2006 x 2852 inner); normalise rotation before diffing.
- **Verify:** enumerate output logged beside captures.
- **Source:** `simctl io` help; Duo capabilities.plist; XCUIScreen.h.

### E-TEST-18. Recording transitions on one display `[General]` (T13)
- **Case:** Apple says test transitions, not only poses. A single-display recording goes black when the app moves to the other display (inferred from captures, not measured on video).
- **Breaks:** jumps and flashes during fold or unfold are never recorded.
- **Detect:** no video in the Duo steps, or `recordVideo` with one `--display`.
- **Fix:** record the transition once per display (`--display=1`, then repeat with `--display=3`): simctl records one display at a time, and a second `recordVideo` while one runs exits 16, "Host recording is already in progress" (measured 2026-10-23, Xcode 27.1; whether it also blocks a recording on a different simulator is untested). Wait for "Recording started" on stderr before pressing; stop by the captured PID with SIGINT. For both displays in one take, record the Device Hub window. The inner display does not record reliably: `recordVideo --display=3` saved a single frame (2006 x 2852, 0.07 s) in two takes on 2026-10-23, started closed and started open, while `--display=1` recorded normally (423 frames in 18.7 s). `screencapture -v -l<window>` of the Device Hub window gives black frames unless the window is on screen. So inner-display video needs the Device Hub window visible and an attended screen recording (QuickTime, or `screencapture -v` of the on-screen window); there is no known unattended path on Xcode 27.1. Count frames with `ffprobe -count_frames` on every take.
- **Verify:** both files show the hand-off with no mispositioned frames.
- **Source:** `simctl io` help; LAB0917 43:40; LAB0916 42:38; the one-at-a-time limit measured with two concurrent `recordVideo` calls on one Duo simulator (2026-10-23).

### E-TEST-19. CI cannot pose the Duo `[General]` (T04)
- **Case:** hosted runners have no Device Hub GUI, no AX or Screen Recording grants, maybe no 27.1 runtime. (No Apple statement; inferred from there being no pose API. Runner images unchecked.)
- **Breaks:** no Duo destination (xcodebuild picks another or errors), or every fold test skips green.
- **Detect:** CI destinations, the runner's Xcode and runtime list, per-run skip counts.
- **Fix:** CI runs pure geometry units and cover-pose UI tests on a Duo named by UDID (fail if absent, never fall back); fold proofs on a local GUI Mac; per-phase count lines; release checklist requires a local fold-proof PASS on the release sha.
- **Verify:** remove the runtime from a runner: CI fails with a named reason; all-skipped never prints PASS.
- **Source:** LAB1013 33:52; Prepare page.

### E-TEST-20. Automated accessibility audit per pose `[General]` (T19)
- **Case:** cover truncation and squeezed hit regions judged by eye only.
- **Breaks:** "Continue wit..." ships.
- **Detect:** `rg -n 'performAccessibilityAudit' Tests/` = 0.
- **Fix:** `try app.performAccessibilityAudit(for: [.textClipped, .dynamicType, .hitRegion, .contrast])` in each pose a test reaches, at default size and an AX size. It checks hit-region size, not occlusion by the column (unverified that it flags that).
- **Verify:** a fixed-width label on the cover fails the cover audit with textClipped and not the open one.
- **Source:** XCUIApplication.h:143-149; XCUIAccessibilityAuditTypes.h.

### E-TEST-21. Dynamic Type per pose, and leftover settings `[General]` (T20)
- **Case:** AX sizes checked only open; `simctl ui <UDID> content_size` changes the whole sim and persists.
- **Breaks:** the cover truncates at AX3-AX5; a leftover AX5 makes later suites or store captures wrong.
- **Detect:** verification steps naming only "open"; no reset in scripts.
- **Fix:** per test the launch argument `-UIPreferredContentSizeCategoryName UICTContentSizeCategoryAccessibilityXXXL` (community technique, not documented); if simctl, read and restore in a trap. Matrix: cover default and AX5, open wide AX5, open turned AX5.
- **Verify:** after the run `simctl ui <UDID> content_size` prints the original.
- **Source:** `simctl ui` help; forum 847644.

### E-TEST-22. Reduce Motion and Reduce Transparency have no simctl toggle `[General]` (T21)
- **Case:** `simctl ui` covers appearance, increase_contrast, content_size only.
- **Breaks:** nobody tests them (Mitamaton: a 400 pt pane sprang in under Reduce Motion, case study section 16).
- **Detect:** `rg -n -e 'accessibilityReduceMotion' -e 'isReduceMotionEnabled' -e 'accessibilityReduceTransparency'` near pane and fold transitions.
- **Fix:** toggle in the sim's Settings app (scripted with a short XCUITest against `com.apple.Preferences`, then reset), or a DEBUG launch override feeding the app's own flag; capture the fold transition with each on.
- **Verify:** dual-display recording shows a fade, not a slide; the column shows the opaque plate.
- **Source:** `simctl ui` help; LAB0916 30:27; case study section 16.

### E-TEST-23. Touch ID, not Face ID, in the simulator `[General]` (T26)
- **Case:** the Duo sim profile declares Touch ID (`touch-id` true, `pearl-id` false). Real hardware biometrics unchecked.
- **Breaks:** copy hard-coded "Face ID", branches on `.faceID`, Face ID enrol/match UI tests.
- **Detect:** `rg -n -F -e 'Face ID' -e '.faceID' -e 'BiometricKit_Sim'`.
- **Fix:** copy and icons from `LAContext().biometryType`; `NSFaceIDUsageDescription` only where used; tests drive the fingerprint path on the Duo.
- **Verify:** lock or sign-in screen says Touch ID on the Duo sim and the biometric test passes.
- **Source:** Duo capabilities.plist and profile.plist.

### E-TEST-24. Physical-device protocol `[General]` (T28)
- **Case:** no Device Hub on hardware; a person folds; `SIMULATOR_MODEL_IDENTIFIER` absent; no touch on both displays at once; different hinge rates; ergonomics only show on the device.
- **Breaks:** file-based handshakes cannot run; sim-only model guards skip everything; awkward column targets ship.
- **Detect:** handshakes that only work on the Mac; model checks without a uname fallback.
- **Fix:** the test pauses on a long `waitForExistence` for a window-size change and names the requested pose; uname fallback; a short device checklist (column reach, band in book and laptop, fold with forms and sheets open, camera direction).
- **Verify:** the same fold test passes on a physical Duo with a person folding at each prompt; skipped steps print FOLD_SKIPPED and fail.
- **Source:** forum 847644 ("validate button targets, vertical bar positioning, and navigation flows on physical iPhone Duo devices"); LAB0917 5:23; case study section 14.

### E-TEST-25. A pose priority when screens are many `[General]` (T29)
- **Case:** six poses plus eight conditions per screen is expensive and gets skimmed.
- **Breaks:** high-yield poses get the same attention as low-yield ones; Mitamaton's blank character showed only in partly open turned.
- **Detect:** does the log hold at least one tall pose with a horizontal fold, one bent band, one left-column case and one transition with state open?
- **Fix:** a suggested order, not a replacement for Done. Done still needs all six poses on every changed screen (SKILL.md "Done"); the tiers only say which to run first, and no tier lets you skip a pose. Tier 1 every changed screen: cover; open wide; partly open wide; partly open turned; one cover-open-cover transition with its sheet or form open. Tier 2 once per release per flow: Split View left, turned cover, RTL, AX5 on the cover, Reduce Transparency, Reduce Motion. Tier 3: open turned and closed turned, run on every changed screen before Done; run them first where tiers 1-2 differed. Then a 27.0 iPhone.
- **Verify:** seed a `UIScreen.main.bounds` cull, a hard-coded trailing 84 and an if/else container swap; tiers 1-2 catch all three.
- **Source:** case study section 1; LAB0917 43:40; TT461 7:44; LAB0916 41:06.

### E-TEST-26. Resizable simulator as a proxy `[General]` (T30)
- **Case:** Apple's Prepare page lists three tools: the iOS resizable simulator in Device Hub, iPhone Mirroring and the Duo simulator (not iPad windowing, which forum 847644 mentions only as an aid). Every recent iPhone device type has a Resizable display (screenID 5).
- **Breaks:** without it, resize bugs (stale frames, branch swaps, `UIScreen.main`) are found only by hand on the Duo.
- **Detect:** `plutil -p '<Device>.simdevicetype/Contents/Resources/profile.plist' | grep resizableScene`; `simctl io <UDID> enumerate` on a booted non-Duo iPhone.
- **Fix:** use Device Hub's resizable iPhone early. Driving it from `simctl io screenConfig geometry` is unverified, and `screenConfig` power toggles once left displays black: try only on a sim you own, then shut it down by UDID. If it works, add a CI resize sweep with state open.
- **Verify:** a scripted geometry change resizes the app window (DEBUG log) and catches a seeded state-loss bug.
- **Source:** Prepare page (fetched); forum 847644; `simctl io` help.

### E-TEST-27. The audit grep never reaches vendored code `[General]` (review)
- **Case:** the quick-audit greps run as `rg ... .` from the repo root. ripgrep skips files matched by `.gitignore` (and `.ignore`, `.rgignore`) and hidden directories by default, and Swift packages resolved by Xcode are not in the repo at all.
- **Breaks:** the audit reports zero `UIScreen.main` hits while vendored code still reads it: `Pods/` (often gitignored), `node_modules/` (React Native, Capacitor), `Carthage/Checkouts`, SwiftPM CLI checkouts under `.build/checkouts` (hidden), Flutter plugin links under `ios/.symlinks` (hidden), and Xcode's package checkouts under DerivedData. Measured 2026-10-23 with ripgrep 14.1.1 in a git repo with `Pods/` gitignored and one hit in each place: plain `rg -l` found 1 file, `--no-ignore` found 2, `--no-ignore --hidden` found 3.
- **Detect:** the audit log shows no flags and no vendor paths; `git check-ignore -v Pods node_modules Carthage .build` names ignored vendor folders; `xcodebuild -showBuildSettings | grep -m1 BUILD_DIR` locates the DerivedData folder in use.
- **Fix:** run a second pass with `rg -n --no-ignore --hidden <patterns> .` and report first-party and vendor hits separately (never hide vendor hits). Also search Xcode's package checkouts: `<DerivedData>/<Project>-<hash>/SourcePackages/checkouts/` by default (for example `~/Library/Developer/Xcode/DerivedData/*/SourcePackages/checkouts`), `<path>/SourcePackages/checkouts` when built with `-derivedDataPath <path>`, or the folder passed as `-clonedSourcePackagesDirPath`. Resolve packages first (`xcodebuild -resolvePackageDependencies`) so the checkouts exist. Closed binaries need `nm -u` (E-TEST-14). Treat vendor hits as E-TEST-14 says: update, pin or contain.
- **Verify:** seed one hit in a gitignored vendor folder and one in a SourcePackages checkout; the audit must report both. The audit report lists every searched root.
- **Source:** `rg --help` (14.1.1: `--no-ignore`, `--hidden`); `xcodebuild -help` (Xcode 27.1: `-derivedDataPath`, `-clonedSourcePackagesDirPath`); the measurement above; a SourcePackages/checkouts folder observed in a local DerivedData folder.

---

## Store and release

### E-STORE-1. The archive did not link the 27.1 SDK `[General; seen in a shipping app (Mitamaton)]` (store-01)
- **Case:** CI pinned to 27.0, `DEVELOPER_DIR` or `xcode-select` on an older Xcode, a release-script fallback toolchain, `#if compiler(>=...)`/`#if swift(>=...)` hiding Duo code.
- **Breaks:** a legal upload that does not reach the edges or runs letterboxed on a Duo, while screenshots and the badge advertise the Duo layout. Mitamaton's release script listed Xcode 27.0 GA as a release toolchain and 26.6 as a "legal upload" fallback, and read no DTSDKName.
- **Detect:** `plutil -p <X>.xcarchive/Products/Applications/<App>.app/Info.plist | grep -E 'DTSDKName|DTPlatformVersion|DTXcode|DTXcodeBuild'` (expect iphoneos27.1+, DTXcode 2710+, 27A9275+), for each `PlugIns/*.appex` too; ASC `GET /v1/builds/{id}/buildBundles` (`sdkBuild`, `platformBuild`); `rg -n -e 'xcode-version' -e 'DEVELOPER_DIR' -e 'xcodes select' -e '#if compiler' -e '#if swift'`.
- **Fix:** post-archive gate refusing export below 27.1 for the app and every appex; pin CI by build number; gate Duo APIs only with `#available`.
- **Verify:** the gate refuses a 27.0 archive and passes a 27.1 one; uploaded `sdkBuild` matches.
- **Source:** Prepare page; BuildBundle doc (fetched).

### E-STORE-2. Two wordings for the April 2027 requirement `[General]` (store-02)
- **Case:** the Screenshot specifications page: required "for any app using the iOS 27.1 SDK or later". The 2026-10-05 news post and App Store "What's new": "any apps or games submitted will need to include screenshots for iPhone Duo". The Duo slot has no scaled fallback (the general display-scaling paragraph does not say it covers the Duo).
- **Breaks:** under the broad reading a 26-SDK maintenance release after March 2027 is blocked; 6.9-inch frames cannot stand in.
- **Detect:** target release date vs April 2027; Media Manager > View All Sizes for a Duo set per localization.
- **Fix:** plan for the stricter reading; keep a Duo set for every app shipping after March 2027.
- **Verify:** re-fetch both pages (date-stamp) before relying on it; Duo slot populated before Submit.
- **Source:** screenshot-specifications; news `kkphp5qo`; app-store/whats-new (all fetched 2026-10-23).

### E-STORE-3. Alpha channels `[General; seen in a shipping app (Mitamaton)]` (store-03)
- **Case:** raw `simctl` PNGs and Figma/browser composites carry alpha even when opaque; bezel PNGs are transparent around the device.
- **Breaks:** "Images can't include alpha channels or transparencies". Mitamaton's captures had alpha (all 255) and were flattened as a precaution.
- **Detect:** `for f in *.png; do sips -g hasAlpha "$f"; done | grep -c 'hasAlpha: yes'` (count, never head).
- **Fix:** PIL `Image.open(p).convert('RGB').save(p)` or JPEG; fill outside a bezel first.
- **Verify:** the count is a literal 0; dimensions still one of the four Duo sizes.
- **Source:** screenshot-specifications; case study section 15.

### E-STORE-4. The Duo display type in tools and APIs `[General; seen in a shipping app (Mitamaton)]` (store-04, D23)
- **Case:** `APP_IPHONE_DUO` is accepted by the live API (confirmed 2026-10-22) but missing from the published OpenAPI spec (4.5.1) and the ScreenshotDisplayType page (`grep -c DUO` = 0 on 2026-10-23).
- **Breaks:** any uploader checking the published enum or inferring the type from pixel size skips or rejects 2853 x 2007 and 1398 x 2034; the version ships without the set or badge.
- **Detect:** POST `/appScreenshotSets` with `screenshotDisplayType` `NOT_A_TYPE`: the error lists allowed values, nothing is created; after upload list `appScreenshotSets` per localization.
- **Fix:** pass the type explicitly; or upload in the web Media Manager.
- **Verify:** each localization lists a Duo set with N frames in order.
- **Source:** screenshotdisplaytype.json (fetched); case study section 15 (live API probe and upload read-back).

### E-STORE-5. App previews for the Duo `[General]` (store-05)
- **Case:** the only accepted Duo preview resolutions are 886 x 1920 and 1920 x 886 (19.5:9); native Duo screens are about 0.69 and 1.42. The spec's own device-resolutions line labels 2034 x 1398 and 2853 x 2007 "19.5:9" (looks like a doc error).
- **Breaks:** a Duo-size recording is rejected; a letterboxed 1920 x 886 one no longer matches the screenshots, so it goes to "A Closer Look" instead of leading.
- **Detect:** `ffprobe -v error -select_streams v:0 -show_entries stream=width,height,r_frame_rate -of csv=p=0 preview.mov` (886,1920 or 1920,886, at most 30 fps); duration 15-30 s.
- **Fix:** skip a Duo preview, or build a framed 1920 x 886 composite of real capture (2.3.4), stereo AAC, 30 fps, accepting "A Closer Look". `simctl recordVideo --display=3` for the inner screen saved a single frame in two takes (2026-10-23), so capture a preview by screen-recording the on-screen Device Hub window. Recheck the spec.
- **Verify:** ffprobe checks; ASC preview tool on a Duo.
- **Source:** app-preview-specifications (fetched).

### E-STORE-6. Badge without the binary `[General]` (store-06)
- **Case:** the badge comes from submitting Duo screenshots; Asset Library allows standalone submissions "reviewed against your app's latest version".
- **Breaks:** Duo frames uploaded while the live binary is pre-27.1: users see a badge and a split product page, then a letterboxed app; risks rejection under 2.3.3.
- **Detect:** the attached (or latest live) build's `sdkBuild`; every frame's capture sha.
- **Fix:** Duo screenshots only on a version whose build passed the SDK gate, captured from it; no standalone Duo frames while an older binary is live.
- **Verify:** version > Build > buildBundle `sdkBuild` = 27.1, matching the capture log.
- **Source:** Prepare page; upload-app-previews-and-screenshots; guideline 2.3.3 (fetched).

### E-STORE-7. Inherited and stale frames `[General; seen in a shipping app (Mitamaton)]` (store-07, D23)
- **Case:** a new version starts with the previous version's sets (observed in Mitamaton, not documented in the pages fetched). Removing a scaled frame removes it for every size using it; delete only while Prepared for Submission.
- **Breaks:** Mitamaton found 7 inherited frames from the previous version in the new set, and a Duo frame still showing an old header after a retake (a frame captured before a code change).
- **Detect:** list each version's sets and compare against a capture manifest (frame, slot, sha, build, pose).
- **Fix:** replace sets in full (delete every frame in the set, then upload the new ones, per version and localization); on a retake check EVERY frame.
- **Verify:** set count and order equal the manifest; every frame's sha is the candidate's.
- **Source:** case study section 15; Apple upload page (fetched).

### E-STORE-8. Exact sizes and one shape per set `[General]` (store-08)
- **Case:** captures arrive in whatever orientation the sim is posed; XCUIScreen's inner entry is 2006 x 2852; cover and inner shapes mixed.
- **Breaks:** off-by-one files rejected; an uneven gallery; sideways frames.
- **Detect:** `sips -g pixelWidth -g pixelHeight *.png` against the four pairs exactly.
- **Fix:** a pre-upload dimension gate; keep one shape per set unless deliberate; record pose and rotation per capture; check the gallery in ASC's preview tool on iPhone Duo.
- **Verify:** gate passes; preview tool shows every frame upright and ordered.
- **Source:** screenshot-specifications; Prepare page.

### E-STORE-9. Bezels and composites `[General; seen in a shipping app (Mitamaton)]` (store-09, D23)
- **Case:** framing with Apple Design Resources' Duo bezel (opening exactly 2853 x 2007 at (120, 120)).
- **Breaks:** composites at bezel size are rejected; scaled into the slot, UI text shrinks (Mitamaton renders the framed image onto a 2853 x 2007 canvas); fewer than half in-use frames fails 2.3.3, judged per device set.
- **Detect:** dimensions and alpha; count in-use frames per set; bezel unmodified and not committed.
- **Fix:** composite under the unmodified bezel on an accepted canvas, flatten, majority in-use, Apple imagery only (2.3.10).
- **Verify:** 100% view: text legible; preview tool.
- **Source:** guideline 2.3.3 (fetched); case study section 15.

### E-STORE-10. Revenue and compliance screens nobody edited `[General; seen in a shipping app (Mitamaton)]` (store-10)
- **Case:** the rebuild changes every screen; paywalls, terms, Restore, Sign in with Apple, account deletion and permission prompts get no Duo pass.
- **Breaks:** price, terms or Restore unreachable on the cover or turned halves: revenue loss and review risk. Mitamaton's Pro paywall was broken on the Duo turned tall and the 1.1.0 submission waited for the fix. App Review IAP screenshots go stale.
- **Detect:** `rg -n -i -e 'paywall' -e 'SubscriptionStoreView' -e 'Product.purchase' -e 'restorePurchases' -e 'AppStore.sync' -e 'SignInWithAppleButton' -e 'deleteAccount'`; a UI test per pose asserting each row, Restore and terms are hittable.
- **Fix:** put these on the Duo done-list; two-pane open layout or a scroll view keeping purchase and legal links visible on the cover (E-LAYOUT-9); recapture IAP review screenshots on change.
- **Verify:** the paywall test passes per pose; cover and turned captures show price, terms and Restore.
- **Source:** Prepare page; case study section 15.

### E-STORE-11. Raising the deployment target instead of gating `[General; seen in a shipping app (Mitamaton)]` (store-11)
- **Case:** `IPHONEOS_DEPLOYMENT_TARGET` or `MinimumOSVersion` raised to 27.1 to skip `#available` fallbacks.
- **Breaks:** every user below 27.1 stops getting updates; nothing in build or tests flags it. Mitamaton targets iOS 18.0.
- **Detect:** `rg -n -e 'IPHONEOS_DEPLOYMENT_TARGET' -e 'deploymentTarget' -e 'MinimumOSVersion'` vs the last shipped value; archive Info.plist `MinimumOSVersion`; ASC `minimumOsVersion`.
- **Fix:** keep the target, gate every Duo symbol; a release-gate assertion that MinimumOSVersion is unchanged unless someone decides to raise it on purpose.
- **Verify:** archive value equals the previous release; app launches on the oldest supported runtime.
- **Source:** Mitamaton's project settings; BuildBundle doc.

### E-STORE-12. Featuring nomination `[General]` (store-12)
- **Case:** Apple asks for a nomination with Duo optimization and "support for all device poses" in Helpful Details.
- **Breaks:** a Duo update goes unseen; a wrong type (App Launch vs App Enhancements) cannot be changed after submitting. The device shipped 2026-10-23, so launch-week lead time has passed for anything filed now.
- **Detect:** ASC > Featuring > Nominations; submitter role (Account Holder, Admin, App Manager, Marketing).
- **Fix:** App Enhancements, at least 3 weeks before release, the right app linked, poses and arrangement behaviour named.
- **Verify:** submitted with correct type and app, at least 21 days before release.
- **Source:** Prepare page; nominate-your-app-for-featuring; news `kkphp5qo` (fetched).

### E-STORE-13. Product page header and creative assets `[General]` (store-13)
- **Case:** header 21:9 3840 x 1646 or 16:9 5244 x 2950; search result 3:2 or 16:9; videos 5-30 s, muted by default; no alpha. Apple does not document how the Duo crops them.
- **Breaks:** focal content clipped; alpha rejected; sound-dependent video plays muted.
- **Detect:** ASC preview tool set to iPhone Duo, both orientations, light and dark.
- **Fix:** focal content inside the template safe area; centred; works silent.
- **Verify:** preview-tool captures show nothing clipped.
- **Source:** creative-assets-specifications; asset-best-practices; app-store/whats-new (fetched).

### E-STORE-14. Promotional text and What's New `[General; seen in a shipping app (Mitamaton)]` (store-14)
- **Case:** Promotional Text (170 characters) changes without a new submission; What's New (4000 characters, localizable) belongs to the version.
- **Breaks:** a Duo claim stays on the version for good or ships in one language; promo text never reverted; unverifiable claims risk 2.3.7.
- **Detect:** list both per localization; check each Duo claim against six-pose captures.
- **Fix:** time-limited Duo copy in Promotional Text with a dated revert task (Mitamaton kept one); What's New factual and true in every pose; localize both (Promotional Text localizability not stated on the reference page; confirm in ASC).
- **Verify:** every localization has the copy; the revert task exists; each claim maps to a capture.
- **Source:** platform-version-information (fetched); case study section 15.

### E-STORE-15. TestFlight for the Duo `[General]` (store-15)
- **Case:** Apple reviews the first build added to an external group ("Subsequent builds may not require a full review"); builds expire after 90 days; TestFlight shows no store assets.
- **Breaks:** a launch-day Duo beta waits on review; testers report on the cover only; feedback lacks the pose.
- **Detect:** external status in ASC; "What to Test" text; the build's `sdkBuild`.
- **Fix:** submit the first 27.1 build for external review early; "What to Test" as a six-pose checklist with fold and unfold mid-task; ask testers to name the pose; check store assets in the ASC preview tool.
- **Verify:** an external group has the build, the checklist lists poses, feedback per pose exists.
- **Source:** TestFlight overview (fetched).

### E-STORE-16. Localized Duo sets `[General]` (store-16)
- **Case:** screenshots are localizable; text overlays need a Duo set per language; RTL: the HIG says the column stays on the hardware side, but `toolbarVerticalEdge` is "based on locale and device" (unmeasured, see SKILL.md section 0); split columns flip.
- **Breaks:** English overlays in other storefronts, or no Duo set there if ASC does not fall back (unverified); RTL frames from an LTR build misrepresent.
- **Detect:** per `appStoreVersionLocalizations`, list `appScreenshotSets` for a Duo set; untranslated overlay strings.
- **Fix:** generate per localization; capture RTL from an RTL run (`-AppleLanguages '(ar)'`), never mirrored images.
- **Verify:** every localization has a Duo set; preview tool per language.
- **Source:** platform-version-information; guideline 2.3.3 FAQ; asset-best-practices; HIG; LAB1015 29:13.
