# Trying the skill on a fresh app

A step-by-step plan for running this skill on an app it has never seen, so the trial both converts the app and shows where the skill is wrong or missing something. Record results in the template at the end; every finding should name the edge-case ID (`edge-cases.md`) it matched, or say "new".

Budget: half a day for steps 1 to 4 on a small app; the pose matrix grows with the number of changed screens.

Every `xcrun simctl` and `xcodebuild` form below was checked against `xcrun simctl help <subcommand>` and `xcodebuild -help` on Xcode 27.1 (2026-10-23). Re-check with the same help commands if a flag fails.

## 1. Preflight

1. **Toolchain.** `xcodebuild -version` shows Xcode 27.1, build 27A9275 or later. With several Xcodes installed, `xcode-select -p` points at 27.1, and every command below uses that path.
   Plan for getting it before the trial starts. A Mac App Store Xcode updates only from the App Store's Update button (there is no command-line installer, so someone at the keyboard clicks it); a developer.apple.com download needs the account owner to sign in. Xcode 27.1 then lacks the iOS 27.1 simulator runtime: `xcodebuild -downloadPlatform iOS` fetched it (7.85 GB, build 24A94425), and the iPhone Duo device type appears only once it is installed (trial on a second SwiftUI app, 2026-10-24, Xcode 27.1 27A9275, iOS 27.1 runtime 24A94425). The iOS 27.0 runtime stays installed for the control run.
2. **Runtimes and ids.** `xcrun simctl list runtimes` shows iOS 27.1 and iOS 27.0, each line ending in its runtime id. Get the two ids you need before creating anything:
   ```bash
   xcrun simctl list runtimes | grep -F 'iOS 27.1'        # ... - com.apple.CoreSimulator.SimRuntime.iOS-27-1
   xcrun simctl list devicetypes | grep -F 'iPhone Duo'   # iPhone Duo (com.apple.CoreSimulator.SimDeviceType.iPhone-Duo)
   ```
   Confirm 27.1 supports only iPhone Duo. With `jq` (optional; recent macOS ships `/usr/bin/jq`, older systems may not):
   `xcrun simctl list runtimes -j | jq -r '.runtimes[] | select(.version|startswith("27.1")) | .supportedDeviceTypes[].name'`
   Without `jq`: `xcrun simctl list runtimes -j | awk '/"version" : /{f=($0 ~ /"27\.1/)} f && /"name"/'` prints the device type names, then the runtime's own name last.
3. **Simulators (three, one booted at a time).**
   - **Sim pool rules first.** If the team has a simulator pool policy (a keep-list, a per-person ledger, "never create simulators"), take devices from the pool and do not create or delete any. Reuse an existing Duo when one is present and free: `xcrun simctl list devices "iPhone Duo"` (a search term filters by name, per `simctl help list`).
   - **Duo:** an iPhone Duo on iOS 27.1 with a unique name. Only if no pool rule forbids it and none is yours: `xcrun simctl create "iPhone Duo (trial)" com.apple.CoreSimulator.SimDeviceType.iPhone-Duo com.apple.CoreSimulator.SimRuntime.iOS-27-1` (use the ids step 2 printed). Rename a shared one only if you own it.
   - **Control:** a current iPhone on iOS 27.0. It runs only the `#available` fallbacks (E-TEST-1).
   - **Smallest plain iPhone:** choose by the app's deployment target and the runtimes installed, not by a fixed model. Read the target (`xcodebuild -showBuildSettings -scheme <S> | grep -F IPHONEOS_DEPLOYMENT_TARGET`), pick the oldest installed runtime at or above it, list the iPhones that runtime supports (either form above with its version in place of 27.1), and take the narrowest. simctl does not print screen widths: confirm the width from the first screenshot (pixels divided by the scale) or the DEBUG logger in step 7. On the Mac this was written on (2026-10-23), the 27.0 runtime offered iPhone SE (2nd and 3rd generation) and iPhone 12 mini and 13 mini, which are 375 pt wide, not the 390 pt iPhone 17e (width from Apple's device specs, not measured here; unverified).
   Verify each UDID: `xcrun simctl list devices | grep -cF <UDID>` prints 1.
4. **Device Hub.** `open -g -a "$(xcode-select -p)/../Applications/DeviceHub.app"`. Boot the Duo, confirm its window is on screen and shows Closed / Partially Open / Open / Rotate Right. Never quit Device Hub (it shuts down every simulator on the Mac; `verification.md`).
5. **Pose control.** Poses come only from the Device Hub pose controls (click them, or script them through macOS Accessibility). No simctl, devicectl or XCTest API folds the Duo (`verification.md`). A script should target the exact window title and refuse when two windows match. Scripting needs Accessibility and Screen Recording for the terminal. Preflight with one harmless press and a `--display=1` capture.
6. **Software keyboard.** Long-lived sims simulate a hardware keyboard, so text fields never raise the software keyboard (E-INPUT-3). simctl has no keyboard subcommand (`xcrun simctl help` lists none), so this is manual:
   - Device Hub Settings, General, Keyboard: turn off "Simulate hardware keyboard by default". Its own description says it sets "the initial value of hardware-keyboard simulation for newly connected devices" and that it "can be overridden" per device (strings in the Device Hub 27.1 binary). Where the per-device override sits in the Device Hub window is (unverified); look for a keyboard control in the device window or menu bar.
   - Or use a Duo you own and may wipe: `xcrun simctl erase <UDID>` (shut down first; erases all content and settings). Never erase a pool or shared sim.
   - Check it worked: tap a text field and confirm the software keyboard is on screen (a UI test can assert `app.keyboards.count > 0`).
7. **DEBUG size logger.** A fresh app has no window-size log, and several steps below read one. Add a DEBUG-only logger at the root (remove it at the end, or keep it behind `#if DEBUG`). SwiftUI, on the root view:
   ```swift
   #if DEBUG
   .overlay { GeometryReader { g in Color.clear
       .task(id: "\(g.size)|\(String(describing: hSize))|\(phase)") {
           print("DUO size=\(g.size) insets=\(g.safeAreaInsets) h=\(String(describing: hSize)) phase=\(phase)") } }
       .ignoresSafeArea() }   // with @Environment(\.horizontalSizeClass) var hSize, @Environment(\.scenePhase) var phase
   #endif
   ```
   The `.ignoresSafeArea()` makes the size the full window. Whether `g.safeAreaInsets` still reports the insets under it is (unverified); if it prints zero, add a second reader without `.ignoresSafeArea()`. A modifier inside `#if DEBUG` needs Swift 5.5 or later.
   UIKit, in the `UIWindowSceneDelegate` (`windowScene(_:didUpdateEffectiveGeometry:)` is iOS 26 and later, per the 27.1 SDK `UIWindowScene.h`):
   ```swift
   #if DEBUG
   func windowScene(_ s: UIWindowScene, didUpdateEffectiveGeometry previous: UIWindowScene.Geometry) {
       print("DUO size=\(s.effectiveGeometry.coordinateSpace.bounds.size) h=\(s.traitCollection.horizontalSizeClass.rawValue) state=\(s.activationState.rawValue)")
   }
   #endif
   ```
   Read it with `xcrun simctl launch --console <UDID> <bundle id>` (blocks and prints the app's stdout and stderr) or `--stdout=<file>`. If you cannot change the app, read sizes from screenshots instead (pixels divided by 3 on the Duo: cover 1398 x 2034 px is 466 x 678 pt).
8. **Build, install, launch (the baseline).** On a clean checkout of the fresh app, note the commit, current deployment target, the Xcode the app last shipped with, and the lifecycle. Then build it unchanged with 27.1:
   ```bash
   UDID=<duo udid>; SCRATCH=<run folder>
   xcodebuild -scheme <S> -destination "id=$UDID" -derivedDataPath "$SCRATCH/dd" build > "$SCRATCH/build.log" 2>&1; rc=$?
   grep -F '** BUILD' "$SCRATCH/build.log"          # read the verdict, not only rc
   APP=$(ls -d "$SCRATCH"/dd/Build/Products/*-iphonesimulator/*.app | head -1)
   BID=$(/usr/libexec/PlistBuddy -c 'Print :CFBundleIdentifier' "$APP/Info.plist")
   xcrun simctl boot "$UDID"                         # skip if already booted
   xcrun simctl install "$UDID" "$APP"
   xcrun simctl launch "$UDID" "$BID"
   ```
   Add `-workspace <W>.xcworkspace` or `-project <P>.xcodeproj` when the folder has more than one. Leave signing on if the app uses entitlements (keychain, push, HealthKit, App Groups): a build with `CODE_SIGNING_ALLOWED=NO` strips them and the app fails at runtime, not at build time. Keep derived data out of iCloud-synced folders. With several app targets, pick the `.app` by name instead of `head -1`.
   **Launch blocker:** an app without the UIScene lifecycle built on the iOS 27 SDK does not launch [TN3187] (E-LIFE-1). If the launch fails or the app exits at once, check for `UIApplicationSceneManifest` in Info.plist and a scene delegate (or a SwiftUI `App`). If missing, record the baseline as "does not launch on the 27 SDK", take the "before" captures from the last shipped Xcode's build on the 27.0 control instead, and make the scene migration the first conversion step (SKILL.md section 3).
   Capture the first screen on the cover and open held wide: this is the "before".
9. **Proxies first (SKILL.md, Verification).** Before the Duo, run the app on Apple's two other resize tools [PDUO]: "The iOS resizable simulator in Device Hub" (a plain iPhone sim's resizable display in Device Hub; scripting it is unverified, E-TEST-26), and "iPhone Mirroring on macOS 27. Resize the window to extreme sizes in both directions." Mirroring needs macOS 27 on the Mac and a real iPhone; skip it and say so if either is missing. Note what breaks there (stale frames, lost state on resize, `UIScreen.main` reads); the side column, asymmetric insets and the fold still need the Duo.
10. **Scratch.** One folder for captures, logs and derived data per run, named by date and sha. Delete it afterwards, and shut down by UDID only the sims you booted.

## 2. Quick audit (SKILL.md section 1)

1. Inventory: targets (`xcodebuild -list`), extensions (`rg -l NSExtensionPointIdentifier -g '*.plist'`), languages present, local packages, vendored binaries, lifecycle (`@main ...: App` or scene manifest). Record N/A for absent features (widgets, Live Activities, camera, web).
2. Run every grep from SKILL.md section 1 with no `--type` filter. For each grep record the hit COUNT before reading. A zero is a claim: confirm the path exists and the pattern is fixed-string where it should be.
3. Binary check: `nm -u <each vendored binary> | grep -c UIScreen`.
4. Storyboards and assets: regular-width variants.
5. Pick archetypes (SKILL.md section 2) and list the edge-case IDs to check.
6. Write the audit table (template below) before changing any code.

## 3. Convert

Follow SKILL.md section 3 in order. After each step, rebuild and launch on the Duo (closed) and the 27.0 control. Keep `implementation-notes.md` in the app's repo for decisions and surprises. Commit per step so a regression can be bisected.

Checkpoints:
- After lifecycle and toolchain: the app launches on both sims with no UIKit assert (E-LIFE-1).
- After `UIScreen.main` removal: the guard test is red on the old commit and green now.
- After bars: every toolbar item has a symbol and a title; screenshot the cover to confirm buttons are in the side column.
- After the fold work: pure geometry tests include offset, RTL, leading-column and inactive-width-0 cases.

## 4. The pose matrix

Device Hub buttons give six poses: Closed, Partially Open, Open, each in the default orientation and after one Rotate Right. Reset rotation first (press Closed, Rotate Right until the default; confirm from the DEBUG size logger (step 1.7) or the capture shape, not only pixels, E-TEST-15).

| # | Pose | How | Display | What must hold |
|---|---|---|---|---|
| 1 | Cover upright | Closed, default | 1 | Side column trailing; no truncation; nothing under the column; no empty tab-bar foot |
| 2 | Cover turned | Closed, Rotate Right | 1 | Column may be on the left; portrait-only reads sideways (decide if acceptable) |
| 3 | Open wide | Open, default | 3 | 951 x 669; top 0, 84 on the column edge; second half used; corners clear |
| 4 | Partly folded wide (book) | Partially Open, default | 3 | 40 pt band free of text and controls; both halves usable |
| 5 | Partly folded turned (horizontal fold) | Partially Open, after Rotate Right | 3 | Horizontal fold; views below 678 pt render; stable bottom half holds the controls; no side column; paywall plans, price, terms and Restore visible (E-LAYOUT-9, E-STORE-10) |
| 6 | Open tall | Open, after Rotate Right | 3 | 669 x 951; views right of 466 and below 678 render; paywall plans, price, terms and Restore visible (E-LAYOUT-9, E-STORE-10) |

The six button poses do not include tabletop (laptop) or tent. Those need Device Hub's continuous hinge and 3D rotation controls (`verification.md`), which no script drives; do at least one tabletop check by hand for apps with media or games, and record it as a separate row.

Then, per critical flow (sign-in, onboarding, main list and detail, editor or composer, paywall and Restore, settings, any camera, playback or game screen):

1. Open the flow on the cover (pose 1) and put it mid-task: a sheet open, text typed (software keyboard up, step 1.6), a detail open, playback running.
2. Unfold to Open (pose 3). Check: same screen, same state, text and focus intact, nothing placed from the cover's frames.
3. Partially Open (pose 4), then back to Open.
4. Fold to Closed. Check again.
5. Repeat 1 to 4 starting from the turned poses (2 to 5 or 6).
6. Record both displays during at least one transition per flow (one display per pass, since simctl records one at a time; section 6).

### Conditions, once per release per flow

Set each one, run the flow on the cover and open, then restore it. Settings that change the whole sim persist across launches (E-TEST-21), so read the current value first and put it back.

| Condition | How on the Duo simulator |
|---|---|
| Split View, app on the LEFT; 50/50 | Manual: drag the app by its home indicator in Device Hub until a drop zone appears, put a second app beside it, drag the divider to the middle [TT461 7:44; LAB1013 46:05]. No simctl command. |
| Open tall with a pinned PiP video above the app | Manual: start PiP in Safari or TV, use the pin control, then open the app (`verification.md`, Conditions). No simctl command. |
| RTL language | `xcrun simctl launch $UDID <bundle id> -AppleLanguages '(ar)' -AppleLocale ar_SA` (quote the parentheses; unquoted, bash rejects them and zsh fails with 'number expected'). Per launch only. |
| Long strings (double-length pseudolanguage) | Scheme, Run, Options, App Language: Double-Length Pseudolanguage. From the command line, `xcrun simctl launch $UDID <bundle id> -NSDoubleLocalizedStrings YES` (a long-standing Foundation argument, not in simctl help; unverified on 27.1). Check at 466 pt on the cover. |
| Reduce Motion | Manual: Settings app in the sim, Accessibility, Motion, Reduce Motion. `simctl ui` has no toggle (its help lists only appearance, increase_contrast, content_size; E-TEST-22). |
| Reduce Transparency | Manual: Settings app, Accessibility, Display & Text Size, Reduce Transparency. No simctl toggle (E-TEST-22). Expect the opaque plate behind the column. |
| Dynamic Type AX5 | `ORIG=$(xcrun simctl ui $UDID content_size)`, then `xcrun simctl ui $UDID content_size accessibility-extra-extra-extra-large`; restore with `xcrun simctl ui $UDID content_size "$ORIG"` and check it prints the original (E-TEST-21). Per test instead: launch argument `-UIPreferredContentSizeCategoryName UICTContentSizeCategoryAccessibilityXXXL` (community technique, E-TEST-21). Cover and open. |
| Dark Mode | `xcrun simctl ui $UDID appearance dark`; restore with `appearance light` (or the value `xcrun simctl ui $UDID appearance` printed first). An app that stores its own appearance setting may ignore it. |
| Increase Contrast | `xcrun simctl ui $UDID increase_contrast enabled`; restore with `disabled`. |
| Bold Text, Button Shapes | Manual: Settings app, Accessibility, Display & Text Size. No simctl toggle (E-A11Y-6). Check cover text at 466 pt and that custom buttons show an edge. |
| Two instances side by side | If the app supports multiple scenes: open a second window and place both in Split View (manual, as above). |
| Live Activity running | If the app has one: start it, then fold and unfold. |

Control pass: the same screens on the 27.0 iPhone and the smallest iPhone (step 1.3). Anything that changed there is either a bug or a named broadening.

## 5. Per-archetype checks

Run only the rows that apply. Each lists the edge cases to confirm or rule out.

| Archetype | Must check |
|---|---|
| Old UIKit / storyboard | Launches with scene lifecycle (E-LIFE-1); constraints to safe area, no 0.5 splits (E-LAYOUT-2); regular variants (E-LAYOUT-1); action sheets anchored (E-PRES-9) |
| SwiftUI list + detail | Detail survives the cover (E-PRES-1); close works in-tree (E-PRES-2, 3); re-tap and per-tab Close (E-PRES-4, 5); one X (E-PRES-6); VoiceOver and Reduce Motion (E-A11Y-2); `.inspector` column to sheet keeps edits (E-PRES-10); TipKit popovers on column items (E-PRES-11) |
| Forms, editors, chat | Keyboard placement in Split View and below PiP (E-INPUT-1); accessory bar stays on the keyboard (E-INPUT-2); draft and focus through a fold (E-INPUT-3); bottom rows clear the column (E-BARS-5); hardware keyboard shortcuts, pointer, Full Keyboard Access (E-INPUT-5); drag and drop between two instances (E-INPUT-4) |
| Paywall / IAP / sign-in | Stacked axis visibility (E-LAYOUT-9); every pose shows price, terms, Restore (E-STORE-10); auth and purchase sheets anchored to the right scene (E-LIFE-5) |
| Custom chrome | `safeAreaBar` draws and takes taps (E-BARS-1, 2); bar region measured (E-BARS-3); no tab-bar foot (E-BARS-6); iPhone unchanged (E-BARS-7); toasts clear the column, camera and Island (E-BARS-10); status bar glyphs readable over the column (E-BARS-11) |
| Games / Metal | Aspect, no bare letterbox, HUD intact, state kept (E-ARCH-1); drawable = bounds x 3 (E-ARCH-2); corner controls (E-ARCH-3) |
| Media | Forced rotation removed (E-ARCH-4); playback through fold, Split View close, PiP (E-ARCH-5); mini-player foot (E-BARS-8); external display and AirPlay closed and open (E-LIFE-13) |
| Camera / scanner | Full-screen picker (E-PRES-8); direction and mirroring; preview rects after resize (E-ARCH-6, device only) |
| Web / hybrid | Per-side safe-area insets, breakpoints at 466 / 669 / 951, viewport segments test (E-ARCH-7); Safari web extension popups (E-LIFE-14) |
| React Native / Flutter / Unity | Bars drawn by the framework (E-ARCH-8); window vs screen dimensions; lifecycle (E-LIFE-1); Compose MP, Skip, Capacitor, MAUI (E-ARCH-12) |
| Maps / drawing / sensors | Pin off the band, leading sheet (E-ARCH-9); strokes through resize (E-ARCH-10); per-pose meaning (E-ARCH-11) |
| Extensions / widgets / Live Activities | Each target audited (E-LIFE-8); activity regions and bar overflow (E-LIFE-9); App Clips, controls, snippets (E-LIFE-14) |
| Documents / PDF / QuickLook | Book-pose paging and nothing on the band (E-LAYOUT-14) |
| Calls / alarms / SharePlay | Incoming call, alarm and SharePlay through a fold (E-LIFE-15) |
| Any app (always) | Long and double-length strings at 466 pt (E-LAYOUT-13); Display Zoom on the cover (E-LAYOUT-12); Voice Control and Switch Control on column items (E-A11Y-5); Bold Text, Button Shapes, Increase Contrast, Dark Mode (E-A11Y-6); vendored code searched (E-TEST-27); Designed for iPhone on Mac / visionOS if offered (E-LIFE-16) |

## 6. Capture commands

```bash
UDID=<duo udid>
xcrun simctl io $UDID enumerate                                   # confirm display ids and names once
xcrun simctl io $UDID screenshot --display=1 cover.png            # cover
xcrun simctl io $UDID screenshot --display=3 inner.png            # inner (rotated 2853 x 2007 held wide)
xcrun simctl io $UDID screenshot --display=1 --mask=alpha cover-mask.png   # corner checks

# Transitions: ONE display per pass. simctl records one display at a time; a second
# recordVideo while one runs exits 16 "Host recording is already in progress"
# (measured 2026-10-23, Xcode 27.1). Run the same transition twice, once per display.
# CAUTION: --display=3 (inner) saved ONE frame in two takes on Xcode 27.1, while
# --display=1 recorded normally. Check each file with:
#   ffprobe -v error -count_frames -select_streams v:0 -show_entries stream=nb_read_frames <file>
# If the inner take has 1 frame, screen-record the Device Hub window while it is on screen.
# simctl writes "Recording started" to stderr after the first frame (simctl help io),
# so redirect stderr and poll it.
for D in 1 3; do                       # 1 = cover, 3 = inner
  xcrun simctl io $UDID recordVideo --display=$D --mask=black transition-d$D.mov 2>rec-d$D.err & P=$!
  for i in $(seq 50); do grep -qF 'Recording started' rec-d$D.err && break; sleep 0.2; done
  grep -qF 'Recording started' rec-d$D.err || { echo "RECORDING DID NOT START (display $D)"; cat rec-d$D.err; }
  # ... put the app back in the starting pose, then press the transition in Device Hub ...
  kill -INT $P; wait $P                # SIGINT, only the PID captured above; simctl finalizes the file
done
# For both displays in one take, screen-record the Device Hub window instead.

# Control iPhone (single display):
xcrun simctl io <control udid> screenshot ctl.png
```

`--mask=alpha` is not supported for `recordVideo` (use `black` or `ignored`; `simctl help io`). The display that is off records black. Never stop a recording with `pkill` or `killall`: other sessions may be recording too.

Name files `<flow>-<pose#>-<appearance>-<sha>.png`. Beside each layout claim, quote the DEBUG logger's line (window size, safe area, size class, scene state; step 1.7) and, where the app reads them, the fold region and bar region. Before / after pairs use the same pose and appearance.

For UI tests: model check for the device (`iPhone19,4`), width for the pose; `-parallel-testing-enabled NO` for pose stages; FOLD_SKIPPED fails the gate (`verification.md`, Tests).

## 7. What "done" looks like

- Audit table complete: every grep has a count, every hit fixed or justified, every target audited or N/A.
- App launches on the Duo and the 27.0 control with the 27.1 SDK; archive shows `DTSDKName` iphoneos27.1 for the app and every appex; `MinimumOSVersion` unchanged.
- Guard tests (any `UIScreen.main`) red on the baseline, green now.
- Every changed screen captured in all six poses and on both control iPhones; every critical flow survives fold and unfold mid-task from both orientations, with the keyboard up where relevant.
- Proxies run first (resizable simulator; iPhone Mirroring if the Mac has macOS 27), findings recorded.
- Conditions pass (Split View left, 50/50, pinned PiP, RTL, double-length strings, Reduce Transparency, Reduce Motion, AX5, Dark Mode, Increase Contrast, Bold Text, Button Shapes), and every sim-wide setting is restored (`xcrun simctl ui $UDID content_size` and `appearance` print the originals).
- Fold proofs (if written) PASS with zero FOLD_SKIPPED, run with the driver committed.
- No change on plain iPhones except named broadenings.
- If shipping: Duo screenshots at exact sizes, RGB, captured from the release sha, uploaded with `APP_IPHONE_DUO` and read back COMPLETE.

## 8. Results template

Copy this into the trial's notes and fill it in. Send the "Skill feedback" section back so the skill can be updated.

```markdown
# Duo trial: <app name> @ <sha>, <date>

## Setup
- Xcode: <version/build>   Duo sim: <name/UDID>   Control: <name/UDID, iOS 27.0>   Small: <name/UDID>
- App: <SwiftUI|UIKit|storyboard|RN|Flutter|Unity|web|mixed>; lifecycle: <App|scenes|AppDelegate window>
- Targets: <list, with N/A features>   Archetypes: <list>

## Audit
| Grep / check | Count | Real issues | False positives | Edge case |
|---|---|---|---|---|
| UIScreen.main / mainScreen | | | | E-LIFE-2 |
| idiom / NavigationView / size class | | | | E-LAYOUT-1 |
| Vendored pass (`--no-ignore`, SourcePackages/checkouts, binaries) | | | | E-TEST-14, E-TEST-27 |
| ... | | | | |

## Edge cases that fired
| ID | Where (file:line or screen) | Pose(s) | Detected by (grep / pose / test / review) | Fixed? | Notes |
|---|---|---|---|---|---|
| E-PRES-1 | | 1 after 3 | pose | yes | |

## Edge cases checked and not present
<IDs, with how each was ruled out. Include the ids tagged (review) that apply: E-LIFE-13 to E-LIFE-16, E-LAYOUT-12 to E-LAYOUT-14, E-BARS-10, E-BARS-11, E-PRES-10, E-PRES-11, E-INPUT-4, E-INPUT-5, E-A11Y-5, E-A11Y-6, E-ARCH-12, E-TEST-27>

## New issues (not in edge-cases.md)
| What broke | Pose / condition | Detect | Fix | Verify | Source / evidence |
|---|---|---|---|---|---|

## Pose matrix
| Screen / flow | 1 | 2 | 3 | 4 | 5 | 6 | Fold mid-task | Split L | 50/50 | PiP | RTL | RT | RM | AX5 | 27.0 iPhone | Small |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

## Skill feedback
- Wrong or stale guidance: <section, what it said, what was true, evidence>
- Missing guidance: <what you had to work out yourself>
- Greps that missed real issues / produced noise: <pattern, why>
- Steps that took longest: <step, minutes, why>
- Edge cases marked (unverified) that you measured: <ID, result>
```
