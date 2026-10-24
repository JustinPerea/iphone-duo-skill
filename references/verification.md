# Verifying an app on the iPhone Duo

Measured on Xcode 27.1 (27A9275) with the iPhone Duo simulator, October 2026; updated 2026-10-23 with fold-proof lessons from a shipping app (`mitamaton-case-study.md`) and the verified edge-case sweeps (`edge-cases.md`, IDs E-TEST-n). Recheck anything version-specific.

## Toolchain

- Xcode 27.1 final. Confirm with `xcodebuild -version` (27A9275 or later). 27.0 cannot compile `reservedRegions` or `axisBehavior`.
- If a release script rejects 27.1 final as a beta, check how it decides: the 27.1 cycle sits in Apple's seed number range. One working rule: a seed-range build is a beta only if the Xcode bundle has no `Contents/_MASReceipt/receipt`.
- `xcodebuild -version` proves the toolchain, not the binary. After archiving, read `DTSDKName` and `DTXcodeBuild` from the app's and every appex's Info.plist and refuse export below iphoneos27.1; after upload, check `sdkBuild` from ASC `buildBundles` (E-STORE-1). Also assert `MinimumOSVersion` did not change (E-STORE-11).
- Build with an explicit `-derivedDataPath` outside iCloud-synced folders.
- Several Xcodes installed? Use the selected one by path everywhere, Device Hub included (below).

## Simulators

- The iOS 27.1 runtime's only device type is iPhone Duo (model id `iPhone19,4`); a non-Duo 27.1 sim cannot be created. Check: `xcrun simctl list runtimes -j | jq '.runtimes[] | select(.version|startswith("27.1")) | .supportedDeviceTypes[].name'`. Plain-iPhone controls run on iOS 27.0, which only executes your `#available` fallbacks. A failure red on the Duo and green on the 27.0 pin is a Duo-host artifact: file it as such, do not skip it, do not change app code for it (E-TEST-1).
- The Duo sim offers Touch ID, not Face ID (E-TEST-23).
- Give each Duo a unique name (`xcrun simctl rename <UDID> "iPhone Duo (<run-name>)"`): Device Hub titles windows "<name> – iOS <runtime>" (an en dash), so two plain "iPhone Duo"s cannot be told apart (E-TEST-5).
- One booted simulator at a time per workstream; run the iPhone half and the Duo half in sequence; shut down by UDID only what you booted.

## The Duo simulator

- Boots CLOSED: the app runs on the cover.
- Displays: enumerate once per session with `xcrun simctl io <UDID> enumerate` (the profile has 1 cover, 3 inner with native orientation 270, plus 2 TVOut, 4 CarPlay, 5 Resizable; `--display` also takes names). `screenshot --display=1` is the cover (1398 x 2034 px); `--display=3` the inner screen (2007 x 2853 px, returned rotated 2853 x 2007 held wide). The display that is off captures black.
- Corner masks: `screenshot --mask=alpha` (or `black`) masks a raw capture, so Device Hub window crops are not needed for corner checks. Cover corners: radius 8 at UL/LL (the hinge side, inferred), 59 at UR/LR, which change side after a rotation; inner 55 all round. `--mask=alpha` is not supported for `recordVideo` (use `black` or `ignored`) (E-TEST-16).
- Do not power displays on or off with `screenConfig`: it left both capturing black until a shutdown and boot by UDID.
- Folding and turning: only Device Hub changes the pose: the pose buttons (Closed, Partially Open, Open, Rotate Right) in the device window's action bar, the strip that appears over the device when the pointer is on it (a first-time tester could not find them from "canvas buttons"; trial, 2026-10-24), and per tech talk 111461 and the 9/17 group lab a continuous hinge angle and 3D rotation (book, laptop, tent). No `simctl`, `devicectl` or XCTest API does (Apple confirmed in the 10/13 lab), and `XCUIDevice.orientation` cannot pose the Duo. To automate, script the Device Hub pose controls through macOS Accessibility; target the exact window title; a press that matches two windows must refuse. `XCUIDevice.shared.orientation` does not turn the app either: with a portrait-only Info.plist the window stayed 951 x 669 (measured 2026-10-23); the Duo turns the app only when the device turns, so use Device Hub's Rotate Right.
- An AX press returns success when the press is SENT, not when the pose finishes. After every press, wait for the app's window size to change or the target display's capture to go from black to content (E-TEST-5, E-TEST-6). The caller needs the Accessibility grant, and Screen Recording for window captures: preflight both.
- Device Hub launch: `open -g -a "$(xcode-select -p)/../Applications/DeviceHub.app"`. `-g` keeps the user's Space; the full path avoids launching another Xcode's Device Hub (a community report says an older one lacks posture controls; unconfirmed).
- Device Hub's per-device window can vanish while the sim stays Booted and Device Hub keeps running. Before every fold run, check it with Quartz (`kCGWindowName` matches the device AND `kCGWindowIsOnscreen`); if missing, reopen as above (back within about 5 s); if still missing, print FOLD_SKIPPED, never pass (E-TEST-4). Earlier, after a shutdown and `simctl boot`, the window sometimes did not return at all. If Device Hub "stops answering" (System Events or an Accessibility pose driver sees zero windows while Quartz still lists them with `kCGWindowIsOnscreen` false), check first whether its windows sit on another desktop (Space): activating Device Hub brought them back and they answered within about 1 s (measured 2026-10-23). Activating switches the visible desktop, so on a shared Mac ask the person at the keyboard first.
- Without the Accessibility grant (System Events reports `UI elements enabled` false) a script cannot press the poses. Ask the person at the keyboard to press each one, and before capturing wait until the app's own DEBUG log shows the new window size: in one trial, launching too soon after a pose change captured a blank launch screen.
- Transitions update in passes. Measured: folding from open tall to the turned cover, the horizontal size class became compact one layout pass before the window size changed, and the side column reported on the leading edge for one pass before settling trailing; unfolding sent 18 hinge updates from 2.23 rad to pi. Code that reads two of these values at once must recompute when either changes.
- Rotation persists across pose changes and reboots. Before a capture run, press Closed and Rotate Right until the default orientation is back. A fold driver script can read window shape plus a status-ring corner to tell the rotation; that heuristic depends on the app (hidden status bar, green UI near corners, dark canvas, sidebar state), so on a new app run it at all four rotations first, prefer the app's own DEBUG log of window size, and fail on UNKNOWN (E-TEST-15).
- Never quit Device Hub. It sends `device_shutdown` to every booted simulator on the Mac. To find who shut a sim down, grep the CoreSimulator log (`Library/Logs/CoreSimulator/CoreSimulator.log` in your home folder) for `device_shutdown` and read the `from peer` process.
- `idb ui tap` fails under Xcode 27.1 (SimulatorKit moved). Prefer DEBUG launch arguments that open a screen or state directly.
- `xcodebuild test-without-building` shuts down a simulator it had to boot itself; boot the Duo before the run.
- Framed window captures (`screencapture -l <window id>`) change size with Device Hub's sidebar; crop constants assume one sidebar state.
- Long-lived sims have a hardware keyboard attached, so text fields never raise the software keyboard. Run form fold proofs on an erased Duo or with the hardware keyboard disconnected (E-INPUT-3).
- The sim renders both displays at 60 Hz; its fps says nothing about real hardware. `xctrace` traces against the sim came back empty in one shipping app's runs: measure with body-eval counters or signposts.

## Proxies

Apple's Prepare page lists three resize tools: the iOS resizable simulator in Device Hub, iPhone Mirroring "on macOS 27" (resize the window to extreme sizes in both directions; an older macOS does not offer resizable Mirroring), and the iPhone Duo simulator in Xcode 27.1 [PDUO]. iPad windowing also reproduces most resize bugs (forum 847644 mentions iPad contexts as an aid). Proxies catch stale frames, branch-swapped state loss and `UIScreen.main` reads; the side column, asymmetric side insets and the fold need the Duo. Scripting the Resizable display through `simctl io screenConfig geometry` is unverified (E-TEST-26).

## Poses to check

For every changed screen:

| Pose | What to look for |
|---|---|
| Closed, held tall (default) | Cover at 466 x 678 pt with status bar, toolbar buttons and tab bar in a trailing side column: copy truncation, buttons over text, content sharp under top chrome (no bar region), horizontal scrollers and bottom rows under the column, empty foot where a tab-bar reservation used to be |
| Closed, turned wide | The cover honours supported orientations: a portrait-only app reads sideways. The column follows the camera and may be on the left |
| Partly open, held wide (default) | Bent fold band (measured 40 pt) free of content, including headers, summary lines, lists with nothing selected, centred sign-in and permission screens |
| Partly open, turned tall | Horizontal fold, stacked halves, views below 678 pt still render; no side column; paywalls keep plans and terms visible above a pinned stack (check the paywall here AND open turned tall, E-LAYOUT-9) |
| Open, held wide (default) | 951 x 669 pt, safe area top 0, 84 on the column edge; corner collisions at the top; split layouts; presentations anchored to column buttons; popovers and action sheets anchored |
| Open, turned tall | 669 x 951 pt, no side column; views right of x 466 or below 678 still render (`UIScreen.main`); stacked halves; paywalls keep plans, price and terms visible above the pinned CTA stack (same check as partly open turned) |

Then the same screen on an iOS 27.0 iPhone and the smallest supported iPhone, to prove nothing moved there (or that a change there is a named broadening).

Also, per Apple and the sweeps:

| Condition | What to look for |
|---|---|
| Split View, app on the LEFT (drag by the home indicator) | Column and inset on the leading edge; nothing hard-coded to trailing |
| Split View 50/50 on the inner display | About 475 x 669 per app; nothing clipped |
| Open, held tall, video pinned at the top (start PiP in Safari or TV, use the pin control) | App runs in a short wide window below it; CTAs reachable; composer on the keyboard |
| Right-to-left language | Which side the column is on (HIG says the hardware side; `toolbarVerticalEdge` is "based on locale and device"; unmeasured, so log the value); layout follows the logged edge; split columns flip; region frames mirrored |
| Reduce Transparency on | Opaque plate behind the column |
| Reduce Motion on (Settings app; no simctl toggle) | Panes fade, no springs or slides |
| AX5 text, cover and turned halves and open | No truncation; detail and reader text actually scales; line length capped |
| Keyboard up, every pose | Accessory bar stays on the keyboard; key actions not in overflow; composer on the keyboard top |
| Inner camera active | Floating views off the `.occlusion` region |
| Fold with a list scrolled midway | Visible position kept |
| Fold in Split View | The app you did not touch last may go to the background |
| Live Activity running (apps with ActivityKit) | Bar items still visible; nothing near the Island collides |
| Two instances (apps with multiple scenes) | Each acts only on its own window |
| Bold Text, Button Shapes (Settings > Accessibility > Display & Text Size), Increase Contrast and Dark Mode (simctl ui; restore after) | Cover text still fits at 466 pt; custom chrome shows edges; contrast holds behind the column (E-A11Y-6) |
| Voice Control / Switch Control (device) | Every column item has a name; scanning reaches the column and both panes (E-A11Y-5) |

Pose transitions, not just poses: record them on BOTH displays. simctl records only one display at a time: a second `recordVideo` while one runs exits 16 with "Host recording is already in progress" (measured 2026-10-23, Xcode 27.1). So run the transition once per display (`--display=1`, then repeat it with `--display=3`), or record the Device Hub window to get both in one take. Wait for "Recording started" on stderr before pressing; stop the recording by its own PID with SIGINT (E-TEST-18). The inner display does not record reliably: `recordVideo --display=3` saved a single frame (2006 x 2852, 0.07 s) in two takes on 2026-10-23, started closed and started open, while `--display=1` recorded normally (423 frames in 18.7 s). `screencapture -v -l<window>` of the Device Hub window gives black frames unless the window is on screen. So inner-display video needs the Device Hub window visible and an attended screen recording (QuickTime, or `screencapture -v` of the on-screen window); there is no known unattended path on Xcode 27.1. Check every take with `ffprobe -count_frames` before trusting it.

Fold transitions: with every sheet, popover, panel, form, keyboard, in-progress flow and playback open, fold and unfold. Nothing may lose state, and nothing may stay positioned from the other screen's frames.

When screens are many, a priority order: cover; open wide; partly open wide; partly open turned; one cover-open-cover transition with state open. Then Split View left, turned cover, RTL, AX5 on the cover, Reduce Transparency, Reduce Motion once per flow (E-TEST-25). The order only sequences the work and permits no skipping: open turned and closed turned still run for every changed screen (E-TEST-25's "Tier 3" is not optional), and Done still needs all six poses.

## Tests

- **Unit, pure geometry.** Fold-to-layout, panel rects, close and pane rules as pure functions. Feed synthetic `(frame, isActive)` regions: vertical, horizontal, bent, flat, off-view, too-narrow sides, no region, AND a view offset from the window origin, a fold partly outside the view, an RTL-mirrored frame (and `.fixed`), the column on the leading edge, an inactive division of width 0 (E-TEST-9). Use the measured numbers (951 x 669, safe top 0, trailing 84, fold mid 475.5, band 40).
- **Source guards.** Scan every source root (app, packages, extensions, every language) for any `UIScreen.main` read, not only `bounds` and `nativeBounds` (a `.scale` read slipped through in the worked example), ignoring comments, failing on any hit and on a suspiciously small file count.
- **Device vs pose in UI tests.** Decide the DEVICE by model: `ProcessInfo.processInfo.environment["SIMULATOR_MODEL_IDENTIFIER"] == "iPhone19,4"`, else `uname()` machine on hardware (in the sim uname names the Mac). Use `app.windows.firstMatch.frame.width` only for the POSE, paired with the model. An iPhone held sideways is about 956 pt wide, wider than the open Duo's 951, so width alone sends it down the Duo branch. One shared helper (E-TEST-2).
- **Screens.** `XCUIScreen.main` is the cover and is black while open: iterate `XCUIScreen.screens` and identify each by pixel size (1398 x 2034 cover; 2006 x 2852 inner, one pixel short of `simctl`), never by index alone.
- **Queries.** One `NSPredicate` query for label checks; walking `allElementsBoundByIndex` races a loading screen. On the Duo, side-column tab items are plain buttons, not `app.tabBars`: try `app.tabBars.buttons[name]`, then `app.buttons[name]`. Put unique identifiers on leaf controls; `.accessibilityElement(children: .contain)` before a container's identifier, never nested in another `.contain`; scroll-view identifiers before bar modifiers; drags from an explicit coordinate below any band. Dump `app.debugDescription` at the state when unsure (E-TEST-12).
- **Fold proofs.** Pause at named states and wait for an external script (write `ready-<step>`, wait for `ack-<step>`). Requirements, each learned from a false green (E-TEST-3, E-TEST-8):
  - commit the driver to the repo;
  - each test logs which branch ran and checks the window really changed after the handshake;
  - a fold the test needed and did not get throws `XCTSkip("FOLD_SKIPPED: ...")`; the gate greps the log body and FAILS unless an explicit opt-out (for example `FOLD_SKIP_OK=1`) is set; device-level skips (not a Duo) happen before anything opens and carry no marker;
  - handshake timeout is `XCTFail`, never a fall-through;
  - wait for an observable change (window frame, then a DEBUG accessibility value the app publishes once layout settles), not fixed sleeps; re-resolve elements after it (E-TEST-6);
  - run pose stages with `-parallel-testing-enabled NO` (and `-disable-concurrent-destination-testing`) against the UDID whose window you drive (E-TEST-7);
  - accept `\*\* TEST (EXECUTE )?SUCCEEDED \*\*` plus the exit status captured before any pipe; give every stage its own `-resultBundlePath`; pass `-collect-test-diagnostics never` between poses;
  - print the verdict from an EXIT trap; every early exit records its reason; zero stages run is FAIL; FAIL exits non-zero.
- **Render and snapshot tests on the Duo host.** A `UIWindow` mounted in the host's first scene inherits the cover's insets (right 84, bottom 34) and shifts content (a 200 pt card at x = -42). Set `safeAreaRegions = []` on the test's hosting controller or mount with no scene insets (queued in the worked example, not yet proven; confirm the Duo-only reds clear while the 27.0 pin stays green) (E-TEST-10). `ImageRenderer` lays a `ScrollView` out empty and renders nothing for content carrying `.accessibilityFocused`: render the content inside the scroll view and add a guard against flat cells (E-TEST-11, E-A11Y-3).
- **Snapshots.** Pin each suite to a device model and runtime, and fail (not skip silently) when the pin is missing in its own phase. A Duo snapshot suite can only cover the pose the sim is in (it boots closed); cover open layouts with views rendered at the measured sizes (951 x 669 with trailing 84 and a leading-84 variant; 669 x 951 with horizontal bars) and synthetic regions. Normalise rotation before diffing inner captures.
- **Accessibility.** `try app.performAccessibilityAudit(for: [.textClipped, .dynamicType, .hitRegion, .contrast])` in each pose a test reaches, at default and an AX size (E-TEST-20). Set text size per test with `-UIPreferredContentSizeCategoryName UICTContentSizeCategoryAccessibilityXXXL` (community technique); if you use `simctl ui <UDID> content_size`, read and restore it in a trap, because it persists (E-TEST-21). VoiceOver order: `XCUIDevice.shared.voiceOverService` (iOS 27.0; `moveForward()` and `currentSpeech()` return an `XCUIVoiceOverOutput`), unmeasured across a fold; a human VoiceOver walk is still needed (E-A11Y-1).
- **Account and shared state.** If a verdict depends on what the simulator is signed in to, give the test its own override seam.
- **Swift Testing selection.** A suite-level `-only-testing` filter can run zero free-function tests and still print success. Select functions by name and read the executed count.
- **Affected tests.** Never hand-pick a subset after touching shared Duo plumbing (presenters, chrome, clips, masks). The worked example missed three tests twice that way while its selector had picked them. Use the selector as the floor, and since it does not follow source-to-source references, also run every render or snapshot suite that draws the changed presenter (E-TEST-13).
- **Previews.** No Duo-specific preview API. Preview at the measured sizes with injected regions; never against `UIScreen.main`.
- **Camera.** The simulator has no camera; from Xcode 27.1 it launches camera apps so their chrome can be checked. Stub the camera behind a DEBUG launch argument for fold tests. Capture, direction changes, scanner rects and the capture accessory need a device.

## CI

Hosted runners cannot pose the Duo (no Device Hub GUI, no AX or Screen Recording grants; no Apple statement either way, and runner images were not checked for 27.1). Split the work (E-TEST-19):
- CI: pure geometry units plus cover-pose UI tests on a Duo destination named by UDID; fail when it is absent, never fall back to another device.
- A local GUI Mac: fold proofs with Device Hub.
- Each phase prints a count line (`GATE_COUNT <phase> <n>`); a phase whose folds all skipped fails unless opted out.
- The release checklist requires a local fold-proof PASS on the release sha.

## Physical device

No Device Hub, a person folds, `SIMULATOR_MODEL_IDENTIFIER` is absent, no touch on both displays at once, different hinge update rates. Make fold tests wait (long `waitForExistence` on a window-size change) and name the requested pose; keep the uname fallback. Device-only checklist: side-column reach and targets, the band in book and laptop poses, fold with forms and sheets open, camera direction and mirroring, scanner rects, motion-sensor meaning per pose (E-TEST-24). Apple: "Always validate button targets, vertical bar positioning, and navigation flows on physical iPhone Duo devices" (forum 847644).

## Captures for review

- Capture before and after on the same pose and appearance.
- An XCUITest screenshot step is a late timing marker on the Duo: it fires after the state you wanted. Time grabs from a state the app logs (or a ready file), then capture with `simctl io <UDID> screenshot --display=3` (inner) or `--display=1` (cover).
- For a layout claim, log the measured numbers (window size, safe area, fold region, bar region) in DEBUG and quote them beside the capture.
- When a fix depends on Apple's styling (scroll edge effects, glass, `safeAreaBar` presence), probe the options in one build with several launch modes and capture each, then remove the probe.

## Done

- Audit greps (no `--type` filter) clean or each hit justified; every target audited or marked N/A.
- Guard tests red on the old code, green now.
- Every changed screen seen in all six poses, on a 27.0 iPhone and the smallest supported iPhone.
- Fold and unfold mid-task lose nothing (sheets, panes, forms, keyboard, playback).
- Fold proofs PASS with zero FOLD_SKIPPED and every stage run.
- The archive links the 27.1 SDK (app and every appex); `#available(iOS 27.1, *)` fallbacks compile and behave on older iOS; MinimumOSVersion unchanged.
