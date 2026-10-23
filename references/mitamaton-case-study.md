# Worked example: a shipping SwiftUI app on iPhone Duo (Mitamaton)

Mitamaton is a SwiftUI iOS app (a daily board of cards, calendar, email, chat and a small animated character). Its Duo work shipped in version 1.1.0 and was audited and reviewed again for 1.1.1, in October 2026. This file records what broke, how it was found, how it was fixed and what was measured. Type names are the app's own; they show the pattern, not a library you can import.

The skill's traps (T1 to T15 in `SKILL.md`) and many edge cases cite the sections below.

---

## 1. `UIScreen.main.bounds` is the cover: the blank character

- **Symptom.** In a Duo walk-through, partly open and turned tall, the character face drew nothing on the open screen; it came back when folded.
- **How found.** A probe log showed the face placed correctly (68 pt at 138.7, 656); a red debug border showed its frame drawn with nothing inside. Layout was right and rendering had stopped.
- **Cause.** The face view and its playback view stopped their render loop for an off-screen face and judged "off screen" against `UIScreen.main.bounds`. On the open Duo that is the 466 pt cover, so any face right of x 466, or below the cover's height when turned tall, never ticked.
- **Why the audit missed it.** The first Duo plan counted `UIScreen.main.bounds` as zero uses in the app's `Sources` folder. The use was in the app's own Swift package, outside the scanned root.
- **Fix.** A `WindowBox` plus a zero-size `UIViewRepresentable` that records the face's own window (T1). Unknown window counts as visible. Zero size so `ImageRenderer` captures have no placeholder.
- **Second instance.** A calendar draft check used `UIScreen.main.bounds.height`; it now reads the foreground-active scene's key window height, `.infinity` when there is none.
- **Guard.** A unit test scans the app's sources and the package's sources for `UIScreen.main.bounds` and `UIScreen.main.nativeBounds`, fails on any hit, and fails on a scan under 500 files.
- **Known residual.** A face whose window is not yet known counts as visible and is not re-checked until its next geometry change (accepted in review).

## 2. Toolchain: Xcode 27.1 final is required, and looks like a beta

- **Fact.** The Duo code uses iOS 27.1 SDK symbols (`ToolbarContent.axisBehavior`, `GeometryProxy.reservedRegions`); Xcode 27.0 fails to compile it (exit 65).
- **Trap.** Both release scripts refused Xcode 27.1 final (27A9275) as a beta, because Apple numbered the whole 27.1 cycle in the seed range. Fix: a seed-range build counts as beta only when the Xcode bundle has no Mac App Store receipt.
- **Apple source.** Tech talk 111461 names Xcode 27.1 as required for the Duo simulator.

## 3. System bars and the side column

- **Context.** Up to 1.0.6 the app shipped a hand-built tab bar. Session 111462: hand-built bars are excluded from vertical placement.
- **Done.** Apple's tab bar and toolbar buttons replaced the custom chrome. On the folded cover iOS moved the tab bar into the side column unprompted.
- **App buttons.** + / Search and Chat use `.axisBehavior(.verticalPreferred)` behind `#available(iOS 27.1, *)`. An earlier pass had concluded iOS 27.1 had no API for this; it does. Chosen order: Close first, then the primary action, tab bar at the foot, icon only.
- **Accessibility gotcha.** SwiftUI's `.accessibilityHidden(true)` did not reach a hosted `UITabBar`; set `tabBar.accessibilityElementsHidden = true` on the controller.
- **Apple check.** Confirmed: only container-owned bars move, and symbol items go vertical by default. `.verticalPreferred` is strictly needed only for custom-view items or to prefer the vertical bar when both exist; harmless on symbol items. Apple also offers `visibilityPriority`, `ToolbarOverflowMenu` and `toolbarVerticalCompressionBehavior`, none used yet (`apple-guidance.md` section 3).

## 4. The navigation bar takes no room on the Duo

- **Symptom.** With its buttons gone to the side column, Home's greeting rose under the date and Email's switcher overlapped its title, because both subtracted a constant 54 pt bar.
- **Fix.** A bar probe measures where the content's safe area starts in the window less the window's top inset, writes `navigationBarHeight`, and dispatches the write to main rather than writing observed state in layout (T4). Content adds the missing clearance.

## 5. Rounded corner on the open Duo

- **Report.** The date at the top touched the curved corner of the open screen.
- **Cause.** Open (and partly folded), the status bar joins the side column, so the window's safe area starts at the top edge. Measured on the Duo simulator: window 951 x 669, `safeAreaInsets` top 0, trailing 84, bottom 34.
- **Fix.** UIKit's corner-adapted safe area (`window.edgeInsets(for: .safeArea(cornerAdaptation: .vertical))`, iOS 26) reports top 16 there and 0 on iPhones; the header drops by the difference (T5). Vertical, not horizontal, so the leading column stays aligned with the cards.
- **Follow-on.** The chat sheet's top did not include that drop and covered the header's foot; fixed by adding the same drop.

## 6. The board splits at the fold

- **Decision.** On the open Duo the board shows two lanes, one either side of the fold.
- **Fix.** The fold comes from `proxy.reservedRegions(kind: .division, options: .includeInactive)` (T2), so lanes hold whether the hinge is flat or bent and the board does not reflow as it moves. Packing still runs once at 12 columns; the result is cut at even rows no card spans and dealt to the shorter lane. No lanes if either side is under 343 pt (the narrowest iPhone board, 375 pt wide less margins).
- **Partly folded.** Bent, the region reports `isActive` and its width (logged `Fold(midX: 475.5, width: 40.0, isBent: true)`); both lanes take the narrow width either side of the band, which holds no content.
- **Tests.** Unit tests feed synthetic regions into the lane resolver.
- **Apple check.** Consistent with Apple: a vertically scrolling grid under a book-pose fold should shift off the fold (App Store Today does), use even columns and hold steady via `.includeInactive` (an Apple engineer described the same trick). Apple's split views go about 35/65 flat to 50/50 folded; the board's lanes are steady instead, which Apple also allows.

## 7. Details open beside the board

- **Decision.** On the open Duo a card's detail and the Email reader open trailing the fold; everywhere else they stay page sheets.
- **First fix.** A detail presenter hands the detail up as a preference and also keeps the `.sheet` attached with `.constant(nil)` while split, so a fold mid-detail hands it across without losing state. A host reads the fold and draws the pane. Close leads the side column.
- **Limits recorded.** The pane takes the surface's environment, not the tile's; a draft delete's `dismiss()` is a no-op outside a presentation, so it also posts the close request.
- **Apple check.** Apple's tools for "detail beside the list" are `NavigationSplitView` and `ArrangementView(.split)`. The pane hand-rolls the same thing, and its two recorded limits come from that. Apple also suggests `ContentUnavailableView` for an empty detail on unfold.
- **Decided in 1.1.1 (section 16).** `ArrangementView(.split)` was rejected: on a closed Duo held in landscape it shows only the primary, so a detail held as the secondary leaves the tree and loses its state on every fold. `NavigationSplitView` keeps a permanent detail column that halves the open board. The pane became a one-tree host (T12).

## 8. The chat sheet on the open Duo, and fold or unfold with chat open

- **Report.** A marketing capture of the open Duo held wide: chat opened as a short card across the middle third, the proposal card cut off.
- **Cause.** The panel hung its top from the chat button and kept a 128 pt bottom gap; moving the button into the side column had put it 174 pt lower, so the card ran 208 to 541 pt of 669.
- **Fix 1.** The panel geometry treats a button at or below the bar line as side-column: the card starts at the bar line, stops short of the column on whichever edge it runs, rests 24 pt off the bottom. Five geometry tests.
- **Fix 2.** The button is measured only on tap and hidden while the sheet is open, so a fold laid the card out from the other screen (858 pt wide on the 466 pt cover). In 1.1.0 a resize closes the sheet, collapsing to a frame-free chip.
- **Why no UI test.** The app is portrait-only, so `XCUIDevice.orientation` cannot pose the Duo; `XCUIScreen.main` is the cover, black while open; the sheet's identifier matched several elements with misleading frames. The check is a capture on the open Duo.
- **Apple check.** Apple says system menus, popovers and alerts are repositioned to stay visible across a fold (they do not close; TT463 5:23). Closing on resize was an assumption about Apple's behaviour; it remains a valid fallback for a custom panel. A system sheet with `presentationPlacement` would avoid the class of bug.
- **Superseded in 1.1.1.** The chat sheet stays open through a fold, re-placed from the side column measured on the current screen (T6). Kept open naively, the card collapsed into the column's first slot (over Add card) because the target fell back to slot 0. Fix: record the chat button's offset in the column order at open, clear it on every open, compute only from a frame inside this pose's column. The camera in the chat is stubbed in simulator fold tests with a DEBUG launch argument.

## 9. Board header and foot

- **Report.** Too much empty space at the top; two dates shown; a solid block behind the tab bar where a blur was expected.
- **Fix.** The header drops its date; on the open Duo, where the greeting clears the character, it rises onto the character's line (0 on every iPhone). The first version landed 32 pt low on the open Duo because it ignored the measured bar height and an 8 pt board offset; measured after the fix: greeting centre 38.0 against the header's 38.08.
- **Foot trap.** `.clipped()` after `.ignoresSafeArea(.container, edges: .bottom)` still clipped at the safe-area foot, and Home's edit mask cut at the same line. Either alone changed nothing on screen; both had to be opened (T8).
- **Missed test.** The mask change broke a render test that the affected-test selector had already picked; it just was not run.

## 10. Scroll edges: Apple's soft blur

- **Report.** The top and bottom edges were still not transparent on a store build.
- **Fix.** Open the top (the clip and Home's mask start 200 pt above the frame at rest; the edit band's cut is kept while editing) and add `.scrollEdgeEffectStyle(.soft, for: [.top, .bottom])`. Chosen by one probe build with five launch modes on an iPhone 17 Pro simulator (iOS 27.0, dark): the default style blurred only the status-bar strip and nothing under the tab bar; automatic bar background or keeping the bottom safe area added nothing. Captured on the iPhone (light and dark) and the closed Duo; the open Duo after-shot was not taken (Device Hub did not bring the Duo window back after a reboot).
- **Open item.** On the CLOSED Duo no bar takes room, so `.soft` has no bar region and cards pass sharp under the header. Accepted for 1.1.0 and deferred.
- **Tried in 1.1.1.** A probe build showed `safeAreaBar(edge: .top)` with `Color.clear` content gives NO edge effect, and `.hard` changes nothing (the first try differed from the old build by 1,666 px). A bar that draws something works: a `Color.primary.opacity(0.001)` fill, with the content's top padding reduced by its height (T14). Then taps on the header reached cards scrolled under it (a throwaway test opened a card). The fix makes the band take taps wherever it has height, on the cover and the open Duo, pinned by a UI test.
- **Apple check.** Apple's docs back the route: `safeAreaBar(edge:)` (iOS 26, a `VerticalEdge` overload exists) insets the safe area and "extends the edge effect of any scroll views affected by the inset safe area". The HIG allows full-width tops only for non-scrolling UIs and keeps scrollable content inset, so sharp cards under the cover's top edge are the expected result of having no bar there, not an Apple bug.

## 11. Email chips under the side column

- A horizontal scroll view draws into the safe area it touches, which on the folded cover is the side column, so filter chips ran under the clock and Search. The chip row clips to its own frame (T9).

## 12. Onboarding splits at the fold (1.1.1)

- **Reports.** The open screen was too wide; on the cover the buttons needed to move down to make room for the text, and the messages stacked so high the board was hidden; open, the content should sit on one side, not centred.
- **Fix.** A split resolver: vertical region (held wide) splits side by side, horizontal region (turned tall) stacks, bent band width honoured, nothing changes with no fold. Subject (character or board) on one side, conversation and buttons on the other, capped at 440 pt (a full-width button turned tall looked far too long).
- **Cover.** Speech bubbles keep a 120 pt scrollable window with a fade; the button lift is clamped to the room under the copy.
- **Trap: a lift that chased itself.** Measuring the buttons' top including the lift just applied made the cover go black at 99% CPU (`sample` showed the body and the geometry binder spinning). Fix: measure the unlifted top inside the geometry transform and ignore changes under 0.5 pt (T10).

## 13. Sign-in splits at the fold; Back in Apple's bar (1.1.1)

- **Bar.** A `UINavigationBar` used standalone, because onboarding cannot sit in a `NavigationStack` (the character's anchor preferences do not cross its hosting boundary, and the board steps mount the app's tab root). Title in a `titleView` label that scales to fit, font capped at 20 pt; a unit test checks every step name at AX5 on a 390 pt width (T11). This changed the plain iPhone too, and was called out as such.
- **State trap.** The first cut switched between a column and a `switch` over the split, so the sign-in control stack moved structurally and lost its `@State`: a fold mid sign-in dismissed the email sheet, cleared `isSigningIn` and dropped the Sign in with Apple nonce. Fix: one tree with `AnyLayout`, the hero an optional earlier sibling (T7).
- **Overflow.** A turned Duo half is about 470 pt tall; the column scrolls when it outgrows it.
- **Apple check.** The state fix matches Apple engineers' advice exactly (branch-swapped containers lose state on fold; `ArrangementView` is their one-tree answer). The bar does not: Apple says content of a hand-built `UINavigationBar` "will never go vertical", so on the cover and the open Duo held wide onboarding's Back sits in a horizontal top bar while every other screen's Back leads the side column. This is Apple's bar class without Apple's placement.

## 14. Verification lessons

- **Device Hub.** Quitting it shut down 5 simulators at once, including ones other test runs were using.
- **Simulator boots closed.** `--display=1` is the cover (1398 x 2034 px, 466 x 678 pt), `--display=3` the inner screen (2007 x 2853 px, 669 x 951 pt; a capture held wide comes back 2853 x 2007); the off display captures black. A `screenConfig ... power off/on` experiment left both black; shutting down and booting by UDID restored them.
- **Poses.** Only Device Hub's pose buttons fold the device. A small local helper pressed them through macOS Accessibility. A generic accessibility tap tool could not press into Device Hub's hosting view; `idb` could not load SimulatorKit under Xcode 27.1.
- **Six poses.** The capture script shot, in order: Open, Partially Open, Closed (all in the default orientation, reset first by pressing Closed and rotating until the window image read upright), then Rotate Right and Closed turned, Partially Open turned, Open turned. On its 3x2 contact sheets: open and partly open held wide carry the side column; the closed cover is tall with its own side column; the turned cover shows the portrait-only app sideways; partly open and open turned are tall with buttons back in a top bar and the tab bar at the foot. The blank-character report in section 1 was pose 5.
- **UI tests.** The split UI tests decided split versus sheet from window width (> 700) and waited at named states for a capture script (ready and ack files). In automation they only ever ran the sheet branch, because the simulator boots closed. Width alone was also wrong: an iPhone held sideways is about 956 pt, so a helper now names the device by model (`iPhone19,4` from `SIMULATOR_MODEL_IDENTIFIER`, else `uname` on a device). A header UI test writes every `XCUIScreen.screens` entry because `main` is the cover.
- **Account-dependent tests.** A render test failed 8 times on the Duo simulator because the simulator was signed in to an account whose stored character style differed; it passed 27/27 on an unsigned simulator. Proposed fix: an identity override the test sets.
- **Capture seam.** A DEBUG launch argument opens the sign-in screen directly, because no existing launch argument reached it.
- **Snapshots.** A pinned Duo snapshot device was planned and never built; the existing pinned snapshot matrices were skipped for 1.1.0.
- **Apple check.** Apple confirmed there is no XCUITest, simctl or devicectl API to fold or rotate the Duo simulator, so driving Device Hub's buttons is the only automation; Device Hub also has a continuous hinge-angle control the helper did not drive.
- **Machine load.** Three booted simulators for one test run exhausted memory; since then: one booted simulator per run, iPhone half then Duo half.
- **Runtime.** The iOS 27.1 simulator runtime supports only the iPhone Duo device type, so the iPhone half ran on an iPhone 17 Pro on iOS 27.0. That differential run sorted 8 failures as Duo-host artifacts.
- **Render tests on the Duo host.** Six hero-card render failures were the test's mount, not the card: the window mounted in the host's first scene took the cover's insets (right 84, bottom 34) and drew a 200 pt card at x = -42, clipped mid-glyph. Queued fix: `safeAreaRegions = []` on the test's hosting controller (one other suite already does it); not yet proven for the render suite.
- **Fold proofs.** A shell script drives Device Hub between `test-without-building` stages. Its first verdicts were wrong: a wait helper returned 1 from its timeout line and nothing folded; it matched only `** TEST SUCCEEDED **` while `test-without-building` prints `** TEST EXECUTE SUCCEEDED **`, so six passing stages read as failed; early exits and an unmatched stage filter printed PASS; Xcode pruned the result bundle before the failure could be read. Now: a `FOLD_SKIPPED:` marker the test gate fails on unless explicitly opted out, both success strings, per-stage `-resultBundlePath`, `-collect-test-diagnostics never`, `-parallel-testing-enabled NO`, a verdict printed from an EXIT trap. It still sleeps 4 to 6 s after each press, and one UI test's 90 s handshake falls through on timeout.
- **Identifiers.** An identifier on a plain stack replaced the children's own; a `.contain` nested in another `.contain` flattened an inner identifier away; a drag "on the board" on the cover began under the header and opened a different screen (inferred: identifier placed after `safeAreaBar`).
- **XCUI tab bars.** On the cover and open held wide the system draws tab items in the side column as plain buttons; `app.tabBars` is empty there, so tests fall back to `app.buttons`.
- **Device Hub window.** The per-device window vanished while the simulator stayed booted; reopening with `open -g -a "$(xcode-select -p)/../Applications/DeviceHub.app"` brought it back within about 5 s without taking focus.
- **Orientation.** A fold driver script read the window shape plus a status-ring corner and a canvas pixel to tell which way the simulator was turned. It is tuned to one app; write your own check against something your app always draws.
- **Test selection.** Three tests were missed twice by hand-picked pre-commit subsets, though the affected-test selector had picked them.

## 15. App Store

- **Decision.** Submit iPhone Duo screenshots. Slot: outer 1398 x 2034, inner 2007 x 2853 (or rotated), optional until April 2027 for apps on the iOS 27.1 SDK (the spec page's wording; Apple's news post reads broader, see E-STORE-2). It adds an "optimized for iPhone Duo" badge.
- **Frames.** Open board (two lanes), Calendar beside the board, a card detail beside the board, Email reader beside the list, closed board. Recommendation: open-screen frames only, because the cover truncated "Sync with Sa..." and faded a time pill. Chat was held until the chat-sheet fix shipped.
- **Capture gotcha.** On the Duo an XCUITest `screenshot` step lands late; time grabs from a logged "tour started" marker and use `xcrun simctl io <device> screenshot --display=3`.
- **Staleness.** Store frames captured before the header change and the open edges had to be recaptured before upload.
- **Copy.** What's New: "On iPhone Duo, open it up and your day spreads across both halves, with details opening right beside it." Promotional text for Duo launch week, with a dated revert.
- **Display type.** `APP_IPHONE_DUO`, read from the live App Store Connect API's enum list via an invalid-type probe; it is not in Apple's published OpenAPI spec (4.5.1) or the ScreenshotDisplayType page. The upload read back COMPLETE.
- **Upload hygiene.** The captures carried an alpha channel (all 255) and were flattened to RGB as a precaution. Seven frames inherited from the previous version were removed from the new set.
- **Framed set.** Apple Design Resources' iPhone Duo bezel, used unmodified; its opening is exactly 2853 x 2007 at (120, 120), so a raw `--display=3` capture fits pixel for pixel; output flattened to RGB.
- **Retake lesson.** After a retake, one frame still showed the old header with the date, because it was captured before a code change: check EVERY frame in a set after a retake.
- **Hinge-crossing frames.** Two submitted frames had content crossing the hinge.
- **Paywall.** The Pro paywall needed a fix on the open Duo turned tall: at rest 5 of 9 rows sat above the main button. The submission waited for that build.
- **Release script.** The release script still listed Xcode 27.0 as a release toolchain and 26.6 as a legal fallback, and never read the archived `DTSDKName`, so it could have shipped without the Duo code (E-STORE-1).

## 16. The 1.1.1 Duo audit and the one-tree pane

- **Audit.** Found, among others: the chat panel across the fold, `dismiss()` doing nothing in the pane, sign-in, permission primes and the email list across the fold, the note camera probably opening as a card, no VoiceOver focus move, announcement, escape or return for panes, a pane springing in under Reduce Motion, text capped at xxLarge on readers, scene-phase handlers, an 85 pt empty foot sized for a tab bar, `UIScreen.main.scale` in an attachment view, and no RTL or leading-column geometry tests. A completeness pass added the share extension (`preferredContentSize` 420 x 540, hand-placed buttons), widgets without extra-large, Live Activities (not applicable) and an OAuth `presentationAnchor` taken from `connectedScenes.first`. Two items sat on paths nothing reaches and needed no fix.
- **One-tree pane.** Each presenter hands its detail up as a preference; one overlay draws it with only its frame changing: beside (vertical fold), fullScreen (cover or turned tall, with the side column) or closed. A hinge probe uses `UIHingeInteraction`; `update.hinge` was non-nil on the Duo in every pose, closed included. Other iPhones keep `.sheet`, attached all the time and bound to `.constant(nil)` while in-tree (T12).
- **Close in the tree.** A close action in the environment (sheet: `dismiss()`; tree: clear the item), claimable by a view with its own close logic, keyed by selection id (T13). Review found no test for the delete path at first.
- **Review blockers.** Clean-up only ran from the pane's own close, so a direct clear skipped `onDismiss` and the deferred action fired at the next close; fix: a rule watching the item id. One process-wide "pane open" flag drove the column's Close across tabs; fix: a per-surface set, every post carrying its surface, cleared on host disappear too. Two X buttons showed; the view's X is hidden in-tree.
- **Re-taps and refusals.** Opening a detail returns early when that source is already open; a second detail on a surface closes the first through its route; a route that refuses makes the newcomer yield (mutation-tested).
- **Scope calls.** A pinned standalone bar changed plain iPhones (about 44 pt lost on the paywall; Close moved), called out as a broadening; the onboarding bar was then limited to the Duo. The header band takes taps. The note row and the email reader's action row ran under the side column; six controls folded into one Attach menu. The text cap was lifted for details and the reader.
- **Paywall held tall (estimates).** A decision half of about 465 pt and a pinned purchase stack of about 200 pt leave about 250 pt for plans and terms; measure before choosing. The paywall UI test needs per-pose visibility assertions.
- **Accessibility.** Pane wrapped as a container, escape calls the close route, focus returns to the source about 350 ms after close, the covered surface leaves the tree; under Reduce Motion the transition gets its own `.opacity.animation(.easeOut(duration: 0.2))`. A human VoiceOver walk is still owed.

## 17. Two Duo-only regressions the iPhone could not catch

- **`accessibilityFocused` blanked offscreen renders.** Returning VoiceOver focus to the source attached `.accessibilityFocused($focus)` to content that render tests draw with `ImageRenderer`; the body never ran and a render test came out blank. Guard: a test that off-tree content still renders offscreen.
- **Hinge flag flipped identity.** The first fix gated the modifier on a hinge flag that starts false and flips true via `DispatchQueue.main.async` after the first report. Email's list got a new identity after the tab appeared, its `onDisappear` ran after the new list's `onAppear`, and Search vanished. The gate now uses something fixed per host, pinned by a test (T15). The Duo fold proofs caught it; the iPhone 17 Pro passed.
