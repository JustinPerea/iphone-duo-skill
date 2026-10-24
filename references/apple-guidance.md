# Apple's guidance for iPhone Duo, with sources

Compiled 2026-10-22 from Apple's primary pages and Apple's own videos; updated 2026-10-23 with corrections and additions from the verified edge-case sweeps (`edge-cases.md`). Every claim carries a source key; the keys are listed at the end. Read the source before relying on a detail that matters, and recheck anything version-specific.

Source strength, strongest first: reference documentation (doc pages with a platform availability line) > HIG and the "Preparing your app" article > code shown on a tech talk's developer.apple.com page > what Apple engineers said in a tech talk or group lab. Group-lab answers are engineers speaking live; several said "don't quote me". Treat them as strong hints, not contracts.

API status markers:
- **(doc)** a reference page exists on developer.apple.com, with the availability shown.
- **(talk code)** shown in code on the tech talk's page, no reference page found on 2026-10-22.
- **(unverified)** named by an engineer in a lab, no page found. Do not write it into code without checking the SDK.

---

## 1. Device model

- Two displays, each with its own front camera. The outer camera is always visible, in a corner, vertically aligned with the side controls. The inner camera sits behind the display and is hidden until a camera is active. [HIG]
- Size classes [TT461 2:46]:

  | Display | Horizontal | Vertical |
  |---|---|---|
  | Outer, portrait | compact | regular |
  | Outer, landscape | compact | compact |
  | Inner (any orientation) | regular | regular |

  The inner display is the first phone with regular x regular [LAB0917 52:55]. The idiom is still phone [LAB0917 52:55].
- Orientation: the outer display behaves like any iPhone and honours supported orientations; the inner display does not honour them [TT461 3:30; LAB0916 28:10]. The current orientation is still exposed as a trait and in the environment on the inner display, but Apple recommends deciding by width and size class instead [LAB0916 45:41].
- `UIRequiresFullScreen` is still honoured, but the app still resizes when the device opens or closes, and "your app will scale on the inner display, including in split view multitasking" [TT461 4:37-4:52]. So the key does not prevent Split View or regular-width layouts. Apple does not say whether "scale" means stretched rendering or a resize (unverified). Apple suggests supporting landscape on the cover for tent use [TT461 3:36].
- Building with the iOS 27 SDK (the next major release after iOS 26) requires the UIScene lifecycle: "otherwise, your app won't launch" [TN3187]. SwiftUI `App` apps already use scenes; old UIKit, React Native/Expo, older Flutter and Unity hosts may not.
- Once an app links the iOS 27 SDK it is resizable; there is no opt-out [LAB0916 47:12].
- SDK ladder [PDUO, fetched 2026-10-23; LAB0916 23:31, 37:16]:
  - iOS 26 SDK or earlier: "won't adapt to all layouts". Open, the app sits "in the center of the display, with empty space surrounding the window"; closed, the outer display shows it "to the left of the status bar and camera". The labs describe this as roughly an iPhone mini ratio.
  - iOS 27 SDK: "resize[s] to fill most of the inner display on iPhone Duo, but still avoid[s] the status bar on the right edge".
  - iOS 27.1 SDK: "extend[s] to use the full display, with toolbars and tab bars appearing vertically below the status bar". "Before shipping an app build with the iOS 27.1 SDK, make sure your app's content adapts to the bars becoming vertical."
  - The PREP article did not render for a fetch on 2026-10-23, so whether it repeats the ladder is unverified.
- Main screen: avoid `UIScreen.main`. Its reference page marks it deprecated since iOS 26.0. The deprecation summary says "Use [`UIWindowScene.screen`] instead", and the discussion says to use a screen "found through context", for example the `screen` property of the window scene that manages the view's window. TT461 4:03 still says "will be deprecated in a future release", and the reference page outranks the talk [doc, fetched 2026-10-23]. Use the environment, the trait collection or the scene's bounds; if a screen is needed, `window?.windowScene?.screen`; replace `UIScreen.main.scale` with `traitCollection.displayScale` [TT461 4:16, 9:25 talk code].
- Corners: use the iOS 26 concentricity APIs (`ConcentricRectangle`, `UICornerConfiguration`) [TT461 4:30 talk code]. On the outer display the corners away from the hinge are rounder than the others, and there are corners where apps never had one before [LAB0917 41:21].
- The status bar is bidirectional: vertical in the side column when held wide, horizontal in the top corner when the inner display is held tall [LAB0917 42:08].

## 2. Postures and what changes

- People hold the device closed, partly folded like a book, set down like a laptop with the inner display up, standing on its edges (tent), and open flat. [HIG; TT466]
- Do not design one layout per pose. Compact width (outer) and regular width (inner) "give you the fundamentals for every pose". [HIG] Apple's own teams learned that filling every pose with something novel was a mistake; support the staples first [LAB0916 41:06; LAB1015 59:35].
- Closing with an app full screen on the inner display moves the same app instance to the outer display; it is a resize, not a relaunch [LAB0917 3:04; LAB1013 43:47]. `onAppear` and `onDisappear` are not expected to fire again; scene phase can change [LAB1013 43:47].
- Closing while two apps share the inner display in Split View: the system promotes the one the person last interacted with to the outer display and backgrounds the other (a scene phase change). Reopening soon restores both; after a while the promoted app takes the whole inner display. Apple does not publish the timing. [LAB0917 3:52; LAB1013 45:19]
- The outer display cannot create new windows; that is reserved for the inner display, so scene creation is dynamic. Use the window scene activation action (`UIWindowScene.ActivationAction`, doc, iOS 15), which hides itself when new windows are unavailable, and handle errors when requesting a scene. [TT464 3:48]
- Multiple instances of an app run like on iPad and share the same `@AppStorage` [TT464 3:38; LAB0916 57:37-57:44]. If the app supports multiple windows on iPad it gets them on the Duo. Both instances are foreground active, so `connectedScenes.first` and `keyWindow` can return the other one; two scenes of one app share one audio route [LAB0917 33:53].
- `@Environment(\.supportsMultipleWindows)` is documented as true on macOS and on iPadOS with the multiple-scenes key; nothing says it tracks the Duo cover's inability to make windows (unverified). Use `UIWindowScene.ActivationAction` and error handling instead [doc].
- Picture in Picture works as on other iPhones, plus a control that pins the video to the top of the screen in open portrait; another app then runs "at the bottom of the screen" and the video does not occlude it, a stacked layout apps handle like any resize [TT464 3:17; TT466 3:22; LAB1013 1:00:47-1:01:34].
- Closing can change scene phase; end long-running work only on `.background` (see also `edge-cases.md` E-LIFE-6).
- Partly folded may turn out to be transient rather than a dominant pose; Apple says it does not know yet [LAB0917 26:47].

## 3. Vertical bars (the side column)

What moves [HIG; PREP]:
- On the outer display, and on the inner display held wide, navigation bars, toolbars and tab bars move to one vertical bar along the side, together with the status bar and the Dynamic Island. The inner display held tall keeps horizontal bars.
- The bar is anchored to the hardware: on the outer display it stays aligned with the camera, so after a rotation it can be on the left [LAB0916 13:40]. In Split View on the inner display each app puts its bar on its outer edge, so the left app's bar is on the left [HIG].
- In right-to-left languages, per the HIG, the bar stays with the hardware: controls on the vertical axis "stay on the same side in right-to-left languages" [HIG]. But the `toolbarVerticalEdge` page says "the system determines the edge based on locale and device" [doc, fetched 2026-10-23], and no RTL run has been measured on the Duo. Read the value instead of assuming a side; treat the RTL side as unmeasured. Split view columns do flip, so in RTL the list sits between the bar and the detail, and the bar's items still come from the detail column [HIG; LAB1015 29:13, 31:28].
- The vertical bar also scrolls away: standard navigation bars minimize on scroll by default on iPhone Duo. UIKit controls this with `UINavigationItem.navigationBarMinimization` (doc, iOS 27.0); a SwiftUI toolbar behavior exists but was not named (unverified). [LAB1013 59:05; LAB1015 26:56]

What qualifies [PREP; TT462]:
- Only bars owned by navigation containers: SwiftUI `toolbar(content:)` on a `NavigationStack` or `NavigationSplitView`, a `TabView`; UIKit `UINavigationController`, `UITabBarController`, `UISplitViewController`. Content of a hand-built `UIToolbar`, `UINavigationBar` or `UITabBar` is never considered: those stay horizontal [PREP; TT462 2:39 talk code; LAB0917 49:50]. `UINavigationController` also supports bottom toolbar items, which do move [LAB1013 30:51].
- An item goes vertical only if it has an icon. A title-only item stays horizontal. A custom view stays horizontal unless you set `axisBehavior(.verticalPreferred)`. Vertical items show the icon; overflow shows icon and title. Give every item both. [PREP; HIG]
- Per context: inspectors keep horizontal bars; in a split view the sidebar and content columns get horizontal bars and the detail column gets the vertical bar [PREP].
- Sheets: on the outer display sheet bars are vertical by default. On the inner display a centred or leading sheet gets a horizontal toolbar and a trailing sheet a vertical one. [PREP] Apple disabled the vertical bar in some control-heavy sheets (Writing Tools, Notes) where it felt distracting [LAB0917 46:43].

APIs (all doc unless marked):
- `axisBehavior(_:)` on `ToolbarContent`, iOS 27.1: `.horizontalOnly`, `.verticalPreferred`. With both bars present, `.verticalPreferred` prefers the vertical one. UIKit `UIBarButtonItem.axisBehavior`. [doc; TT462 8:08] `.verticalPreferred` stays horizontal on other iPhones, so no device check is needed (an `#available(iOS 27.1, *)` check still is) [LAB1013 33:52].
- `visibilityPriority(_:)` with `ToolbarItemVisibilityPriority` (`.automatic`, `.low`, `.high`, custom `init(lowerThan:)`/`init(higherThan:)`), iOS 27.0. Items overflow bottom to top by default. UIKit `UIBarButtonItem.visibilityPriority`. [doc; HIG]
- `ToolbarOverflowMenu`, iOS 27.0: content always goes in the system overflow menu. UIKit `UINavigationItem.additionalOverflowItems`. Do not ship your own overflow menu beside the system one. [doc; HIG; LAB1013 32:21]
- `toolbarVerticalCompressionBehavior(_:)`, iOS 27.1: `.automatic`, `.prefersTabBar`, `.prefersToolbarItems`. UIKit `UINavigationItem.verticalBarCompressionBehavior` with `UIVerticalBarCompressionBehavior` (`.prefersBarItems`, `.prefersTabBar`). Default keeps the tab bar (navigation-focused); task-focused views keep toolbar items. [doc; HIG]
- `toolbarVerticalBehavior(_:)`, iOS 27.1, `.disabled` falls back to horizontal bars. Resolved by the window or presentation: a `NavigationStack` uses its top view, a `TabView` its selected tab, a `NavigationSplitView` its trailing column. Use only for UIs better served by horizontal bars (a full-screen video player, a calculator); treat it as stable, do not toggle per view state; to hide bars use `toolbarVisibility(_:for:)`. UIKit `UIViewController.preferredVerticalBarBehavior`. [doc] The launch screen still shows the vertical status bar; there is no Info.plist key [LAB1015 21:39].
- `toolbarVerticalEdge` environment value, iOS 27.1, `nil` where no vertical bar is ever placed; UIKit `UITraitCollection.verticalBarEdge`. Reports the preferred edge even when no bar is visible. [doc] Measured in a trial (Xcode 27.1, 2026-10-24): it is non-nil on the cover and the inner display held wide and nil held tall, but after turning the open Duo tall it stayed non-nil for one layout pass before going nil, and folding to the turned cover it first reported the leading edge, then trailing. Swap only padding, placement or visibility on it, never view identity (T15).
- Placements: Back or Close at the top via `ToolbarItemPlacement.cancellationAction` (UIKit `leadingItemGroups`); prominent actions like Done via `.topBarPinnedTrailing`, iOS 27.0 (UIKit `pinnedTrailingGroup`). Group items with `ToolbarItemGroup`, do not add fixed spacing. Badges (`.badge(7)`, `UIBarButtonItem.badge`) keep an item visible longer. [PREP; HIG; TT462 5:00, 5:24, 9:27]
- Hero or background images can extend under the vertical bar with `backgroundExtensionEffect()` (iOS 26) or `UIBackgroundExtensionView`. [PREP]
- Sidebar on the inner display: `TabView { }.defaultTabBarPlacement(.sidebar)` (doc) or UIKit `tabBarController.sidebar.preferredPlacement = .sidebar` [TT461 5:44 talk code]; Health uses the sidebar-adaptable tab view style to force it [LAB1015 18:35].

Custom bars, if you truly cannot use the system ones:
- Read `toolbarVerticalEdge` / `verticalBarEdge` for the side, and the region the system bar would draw in. Apple described the region API without naming it [LAB0917 9:13; LAB1013 1:02:08]. A matching symbol exists, `UIView.LayoutRegion.bar(onEdge:extent:)` (iOS 27.1, page has no description; `bar(onEdge:)` is iOS 27.2). Mitamaton shipped it in 1.1.1: measured x 903 / top 120 open and top 170 on the cover, matching the system glass at 902.8 / 119.3. Measure it in `didMoveToWindow`, `layoutSubviews` and on `verticalBarEdge` trait changes from a probe that fills its surface; a zero-size probe only measured when other layout reached it. SwiftUI's `safeAreaBar(edge:)` was named as the lower-level tool for a custom side bar [LAB1015 29:13].
- Apple warns there is no API to copy the system tab bar's behaviour (labels shown on touch-down while scrubbing, lensing, compression), and the vertical bar's height changes with rotation, PiP and Split View [LAB0916 15:11; LAB0917 50:37, 51:24].
- Music and Podcasts put two buttons (lyrics, vocal attenuation, translation) in the bar region [LAB0917 9:58].
- Items whose text carries information (a cart showing a total) are better kept in the horizontal bar: "if the text does contain meaningful information, like a cart button displaying a total dollar amount, it's better to keep the control in the horizontal bar" [TT462 9:49-9:58].
- The bar overflows "as other competing UI appears like the keyboard or when using picture in picture and open portrait" [TT462 11:48-11:55]; side controls also share space with Live Activities and the status bar [TT466 5:37-5:51]. Set `visibilityPriority`.
- A left hamburger drawer stays on the left on the outer display for continuity; users expect Back in the vertical bar under the status bar in every app; if the drawer acts as an overflow menu, move its items into the system overflow [LAB1015 22:55-25:57].
- Cross-platform frameworks that draw their own tab bars and headers (React Native JS tabs, Flutter widgets) produce custom bars by construction, so the rules above apply to them (inference from [PREP; TT462 2:39]).

## 4. Safe areas, margins and occlusions

- Asymmetric safe areas, layout margins and content insets: vertical bars give leading or trailing insets, not top and bottom. Never assume opposite sides are equal ("look for a 2x on a width"). Compute width as `bounds.inset(by: safeAreaInsets).width`. [TT461 7:30 talk code; LAB0916 55:33; LAB0917 21:29]
- Foreground and interactive content inside the safe area; backgrounds may fill and extend behind bars. [TT461 6:29]
- Full-width layouts are fine for visual, non-scrolling UIs (Calculator) as long as nothing hits the Dynamic Island or status bar; you can let a header or background span the full width while scrollable content stays inset. [HIG]
- Corner adaptation: to avoid only the camera or status bar rather than the whole side inset, read the corner-adapted region (the iOS 26 `edgeInsets(for: .safeArea(cornerAdaptation:))` family); Apple recommends it for games and tool palettes, and it also handles iPad window controls [LAB0917 41:21; LAB1013 37:41]. This matches trap T5 in `SKILL.md`.
- Reduce Transparency draws an opaque plate behind the vertical bar, slightly narrower than its safe area; test with it on [LAB0916 30:27].
- `safeAreaBar(edge:alignment:spacing:content:)`, iOS 26, comes in a `VerticalEdge` and a `HorizontalEdge` form. It insets the safe area by the bar AND extends the edge effect of any scroll view affected by that inset. [doc] This is the documented way to give custom top chrome a scroll-edge region. Observed in a shipping app (Mitamaton), not documented: a bar whose content draws nothing (`Color.clear`, `EmptyView`) produced no edge effect, and `.hard` did not help; a 0.001-opacity fill, content at opacity 0, or visible content all did. The bar host also passes touches through to content under it; make the bar take taps (`mitamaton-case-study.md` section 10).
- The keyboard frame from `keyboardFrameEndUserInfoKey` is "in the screen's coordinate space" and "might be different when your app isn't full screen in Split View"; Apple points to `keyboardLayoutGuide` (iOS 15) instead [doc, fetched 2026-10-23].

## 5. The fold: reserved regions

Reference: `GeometryProxy.reservedRegions(kind:options:layoutDirectionBehavior:)` and `UIView.reservedRegions(kind:options:)`, iOS 27.1 [doc RR; RRF]. The skill's earlier spelling `reservedRegions(kind:options:)` is the UIKit one; SwiftUI has the extra `layoutDirectionBehavior` parameter with a default.
- Kinds: `.division` (the fold) and `.occlusion` (the Dynamic Island, outer camera always, inner camera only while active, iPad window controls). [RR]
- By default only active regions return. A division is active when partly open ("partially open or closed" in Apple's wording) and inactive when fully open, so a default query returns no division when flat. `.includeInactive` returns it with `isActive == false`, so you can leave room and avoid a jump when it activates. [RR] The session says an inactive division has width zero [TT463 7:27].
- Frames are in the view's own coordinates, include partial overlaps (can be negative or past the size), and already include the region's `margins`. [RR]
- RTL: frames are mirrored by default to match a flipped layout; pass `LayoutDirectionBehavior.fixed` for unmirrored frames. [RR]
- Each region has a stable `ID` and is `Identifiable`. [RR]
- In a custom `Layout`, compute from (0,0) and add `bounds.origin` only when placing. [RR]
- Where to check [RR]: most apps never need to. Split views, tab bars, alerts, context menus and sheets already adjust. Scrolling containers (lists, tables, scroll views, collection views) do NOT split their content at the fold; a row can run across it. Check yourself in custom overlays, floating controls and explicit-frame or custom-`Layout` views, in the view that positions the content.
- Pick the highest-level API that fits: system components, then arrangement views, then reserved regions, then the hinge angle [LAB0916 19:43; LAB1015 13:28].

Design rules for the fold [HIG; TT463; LAB1013; LAB1015]:
- "Avoid extreme layout changes as people fold the device. Move only what's necessary to keep elements visible and easy to tap." Favor small adjustments over rearranging controls. [HIG]
- Grids: even column counts; keep outer margins and widen the spacing at the hinge.
- Do not displace continuous scrolling content (articles, feeds, lists). Scrolling perpendicular to the hinge works: content can be scrolled off the fold. Scrolling parallel to it does not, so shift that content off the fold (Messages bubbles, App Store Today grid) or change the scroll axis (Preview switches to horizontal two-page paging when folded like a book). Horizontal carousels in Music, Podcasts and TV scroll straight through the fold. [LAB1013 48:25; LAB1015 47:27]
- Destination: book pose, alerts go trailing (toward where the outer display continues); seated (laptop), visible-at-a-distance content goes top and interactive controls bottom on the stable half. Small elements such as context menus and the edit menu stay on the side of their source view. Apple's own controls tend toward the right. [TT463; LAB1013 7:11]
- Flat versus folded: Apple's split views sit around 35/65 flat and move to 50/50 when folded; a natural two-column layout makes the fold transition small. Reading `.includeInactive` lets a layout reserve the gap before the fold activates. [LAB1013 47:39, 50:41]
- Revenue or primary buttons that sit in the content area (Buy, Start trial) are the textbook case for moving a single element with reserved regions [LAB0917 35:59].

## 6. Arrangement views (Apple's container for two related views)

Reference: SwiftUI `ArrangementView`, UIKit `UIArrangementViewController`, iOS 27.1 [doc AV; UAV; TT463].
- Lays out a primary and a secondary view from the size classes, aspect ratio and active division regions. Works on every iPhone and iPad, not only Duo. [AV; LAB0917 56:43]
- `.arrangementViewStyle(.split)`: side by side when wider than tall (primary leading), stacked when taller (primary top); keeps both clear of the fold; a closed Duo held wide shows only the primary. `.arrangementViewStyle(.overlay)`: primary layered over secondary when flat or closed; when partly open, secondary leading or top and primary trailing or bottom. [AV; PREP]
- Restrict axes: `.split.axes(.vertical)` or `.horizontal`, and the overlay style's `axes(_:)` [AV]. If the split cannot use its axis it shows a single view [TT463 12:41; the AV page does not say this].
- Sizing: `splitArrangementLayoutRatio(_:)`, `splitArrangementLayoutRatio(minHorizontal:idealHorizontal:maxHorizontal:minVertical:idealVertical:maxVertical:)`, `splitArrangementLayoutSize(...)`, `splitArrangementFixedLayoutSize(horizontal:vertical:)`; overlay anchoring `overlayArrangementEdge(_:)`; read layering with the `overlayArrangementZIndex` environment value (UIKit `state(for: .primary)?.zIndex`). UIKit: `setViewController(_:for: .primary)`, `updateArrangement(.split.axes(.horizontal))`. [AV refs; TT463 11:26-14:21 talk code]
- Use it when the layout already is an `HStack`/`VStack` (split) or a `ZStack` (overlay). Keep navigation containers outside it (wrap it in the `NavigationStack`), and never put it inside a list, scroll view, grid or `NavigationSplitView`. [HIG; AV] The reverse holds too: "Also avoid placing some container views inside an arrangement view, such as a navigation split view or tab view." [AV, fetched 2026-10-23]
- One tree only while both views are in it: "When someone is holding a closed iPhone Duo in landscape orientation, the split arrangement only shows the primary view" [AV, fetched 2026-10-23]; on the cover held tall `.split` stacks the two; a style restricted with `.axes()` shows a single view on an axis it leaves out [TT463 12:41]. A detail held as the secondary can therefore leave the tree, and lose its `@State`, when the device closes or turns. Mitamaton chose a one-tree overlay host instead for details that must survive the cover (`mitamaton-case-study.md` section 16).
- While both views stay in it, state survives arrangement changes; this replaces `if split { HStack } else { VStack }` switching, which Apple engineers name as the common cause of state loss on fold [LAB0917 19:11; LAB1013 40:43].
- Apple's TV app nests arrangement views: video full screen with controls to the side when folded like a book, video top and controls bottom when seated. "Almost every single app has an arrangement view." [LAB1015 42:07; LAB0917 55:57]
- No sample code project for arrangement views or reserved regions was found on 2026-10-22; the talk pages carry the snippets. A community write-up exists (Natalie Panferova, mentioned in [LAB1015 44:25]); not reviewed here.

## 7. Split views, sheets, popovers, presentations

- `NavigationSplitView` / `UISplitViewController` collapse to one column closed and expand open, and adapt their widths and margins to the fold automatically. [HIG; TT461 5:06]
- Alerts, context menus, popovers and sheets move to avoid the fold by themselves; lightweight presentations are repositioned to stay visible. [HIG; RR; TT463]
- `presentationPlacement(_:)` (iOS 27.0) puts a sheet on the leading or trailing edge, for example to keep a map visible; only sheets respect it. UIKit `UISheetPresentationController.preferredPlacement`. Maps and Find My use a leading sheet on the inner display and in closed landscape. [doc; LAB0916 33:30; LAB0917 47:29]
- A tab bar embedded in a sheet (Find My) can stay with the sheet when every tab feeds that sheet; separate them when each tab has its own sheet. [LAB0916 52:31]
- `DismissAction`: "The dismiss action has no effect on a view that isn't currently presented" [doc, fetched 2026-10-23]. A detail drawn in-tree (a pane, a full-screen overlay) needs a close action from its presenter.
- Popover adaptation follows the horizontal size class, so on the regular-width inner display `.popover`-style presentations may stay popovers (documented UIKit adaptivity; not confirmed on the Duo). Set `popoverPresentationController.sourceItem` (iOS 16) whatever the idiom; `presentationCompactAdaptation(_:)` is iOS 16.4 [docs, fetched].
- `UIImagePickerController`: "Apple recommends that you present the camera interface only full screen" (written for iPad's regular size class) [doc, fetched].
- System menus and popovers are repositioned across a fold rather than closed [TT463 5:23]; a custom panel should do the same.
- When a second column appears on unfold, think about what goes in the detail slot. Auto-selecting can surprise people (an unread conversation would send a read receipt); `ContentUnavailableView` is the suggested empty state. [LAB1013 41:29, 43:00]

## 8. Hinge angle

- SwiftUI: `.onHingeChange { old, context in }`; `context.hinge` is `nil` on a device with no hinge; `hinge.status == .partiallyOpen`; `hinge.angle` is an `Angle`. Reset any angle-driven value in the `else` branch or it sticks. (talk code, TT464 1:44-2:17; no reference page found)
- UIKit: `UIHingeInteraction(updateHandler:)` added to a view; `update.hinge` is `nil` when the hierarchy provides no hinge updates; `UIHinge.Status` is `.closed`, `.partiallyOpen`, `.fullyOpen`, `.unknown`; `UIHinge.angle`. iOS 27.1. (doc)
- Engineers' advice: reaching for the raw angle is rarely right; reserved regions or an arrangement view usually fit better [LAB0916 19:43].
- "The rate and granularity of hinge angle updates are system policy ... don't depend on a particular update frequency or precision"; hinge data drives interactions and effects, layout uses the arrangement and region APIs [forum 847644].
- Observed in Mitamaton: `update.hinge` was non-nil on the Duo in every pose, closed included, and nil on other iPhones, so it can tell a Duo apart. It reports asynchronously after first layout; never gate view identity on it (`mitamaton-case-study.md` section 17).
- CoreMotion: asked whether the reference frame is fixed to the chassis or follows the active display, an engineer answered "it's magic you don't need to worry about" [LAB0916 58:04-58:23] (unverified on a device). The harder question is what "level" means per pose; Apple's Measure team found it hard [LAB1015 59:17-59:55].

## 9. Cameras and the outer display

- Front camera discovery returns a virtual front camera that follows the display the app is on; `isVirtualDevice` is true and `activePrimaryConstituent` names the streaming camera. Apps that capture from "the front camera" keep working. [CAM]
- For full control capture from `builtInOuterUltraWideCamera` and `builtInInnerUltraWideCamera`, and follow which way each faces with `AVCaptureDeviceDirectionCoordinator(view:deviceTypes:changeHandler:)`, created and handled on the main actor, with every built-in type you use (rear cameras too; leave out the virtual front camera). `position` says where a camera sits, not where it points. [CAM; TT465 4:06] (`AVCaptureDeviceDirectionCoordinator` is named in the article and talk code; its own reference URL returned 404 on 2026-10-22.)
- Decide mirroring from the direction map, not `position`; set `automaticallyAdjustsVideoMirroring = false` before `isVideoMirrored`. Mask the preview during a switch. Reconfigure one input rather than running a multicam session. [CAM]
- Use the rotation coordinator for UI rotation instead of tracking orientation yourself ("fake auto rotation" breaks across poses) [LAB0917 22:14; LAB1013 18:39]. Also `AVCaptureDevice.dynamicAspectRatio` (doc), `videoGravity`, and turning off `isCameraSensorOrientationCompensationEnabled` for performance [TT465 7:30-8:34]. Open, the rear sensor is portrait on a landscape screen: letterbox for photography, fill for AR [LAB1013 19:24].
- Showing content on the outer display is only for apps with an active camera session, full screen on the inner display: `.sceneAccessory { CameraCaptureAccessory(isEnabled:) { ... }.onAvailabilityChange { } }` (SwiftUI, iOS 27.1, doc), UIKit `UISceneAccessory` with the `windowCameraCaptureAccessory` session role. Register it on the camera view; availability is system controlled (gone in Split View, when closed, when capture stops); keep every essential control on the inner display. [ACC; TT464 5:43; LAB0917 38:18]
- Simulator: no camera; from 27.1 it launches camera apps so their chrome can be tested [LAB0917 45:12].
- Preview-dependent conversions (`rectOfInterest` from `metadataOutputRectConverted`, overlay frames) depend on the preview layer's size and gravity, so they go stale on any resize; recompute after layout (general AVFoundation behaviour, not stated in Apple's Duo material).

## 10. Scroll position, text and state across a fold

- `UICollectionView`, `UITableView`, SwiftUI `List` and `UITextView` keep the visible position across the resize (index path or text anchor). Custom text rendering needs a semantic anchor saved before the change; never restore a raw Y offset. SwiftUI lazy layouts were not confirmed. [LAB1013 57:33; LAB1015 34:30-37:33]
- Opening shows more items beside the anchored one; Apple considers that correct (open to see more of the app). [LAB1013 39:19]

## 11. Accessibility and Dynamic Type

- Large accessibility sizes look best opened, so do not cap Dynamic Type on detail, reader and editor text there [LAB0916 29:14-29:42]; keep line lengths readable with the readable content guide (`UIView.LayoutRegion.readableContent(cornerAdaptation:)` in the iOS 26 family) rather than stretching text across 951 pt. [LAB0916 28:57; LAB1013 27:01]
- VoiceOver runs on two displays at once. [LAB0916 29:42] Custom layouts that cross folding regions "may need to explicitly define accessibility containers or adjust sort priorities" [forum 847644]. `XCUIDevice.shared.voiceOverService` (iOS 27.0: `enable`, `moveForward`, `moveBackward`, `currentSpeech`, `moveIn`, `moveOut`) automates traversal [XCUIVoiceOverService.h, Xcode 27.1 SDK].
- Verify Dynamic Type across closed, open and partly folded [forum 847644]. `performAccessibilityAudit(for:)` (iOS 17) checks `.textClipped`, `.dynamicType`, `.hitRegion`, `.contrast` [XCUIApplication.h].
- Reduce Transparency changes the vertical bar (section 4).
- Give tab and toolbar items icons; the side tab bar shows icons and reveals labels only while touched. [LAB0917 6:53]
- Apple's material says nothing Duo-specific about Dynamic Type at the cover size beyond the general guidance.

## 12. Keyboard, drag and drop, pencil

- Correction (2026-10-23): Apple's material does cover the keyboard. Accessory bars "should remain attached to the keyboard rather than moving to the vertical axis" [TT462 10:01]. The keyboard can force the vertical bar to overflow [TT462 11:50]. A focused search field over the keyboard adapts its width and position as the device folds [TT463 4:55-5:09]. The keyboard frame is in screen coordinates (section 4).
- Hardware keyboard and drag and drop across halves: not covered. Treat them as ordinary resizable-app behaviour and test on the device or simulator. Whether first responder and IME composition survive a fold in a stable view tree is untested.
- Apple Pencil is not supported at launch [LAB1015 37:33].
- Touch input on both displays at once in the L pose is not supported [LAB0917 5:23].

## 13. Testing

- Simulator: Xcode 27.1, iPhone Duo in Device Hub. It can change the hinge angle continuously and pose the device (closed, book, open, laptop, tent) and rotate it in 3D. [TT461 1:17; LAB0917 43:40]
- No XCUITest, simctl or devicectl API sets the hinge or rotates the Duo simulator; Apple asks for feedback describing the use case (snapshot tests, transitions) [LAB1013 33:52].
- Apple's Prepare page lists three resize tools: "The iOS resizable simulator in Device Hub", "iPhone Mirroring on macOS 27. Resize the window to extreme sizes in both directions.", and "The iPhone Duo simulator in Xcode 27.1", which it calls "the best way to test the full native experience" [PDUO]. Resizable Mirroring therefore needs a Mac on macOS 27. Forum 847644 calls Device Hub's Resizable iPhone "a reasonable approximation" and mentions iPad contexts as an aid.
- Before or besides the Duo simulator: resize the app under iPhone Mirroring on a Mac, or run it in iPad windowing; most bugs reproduce there (Apple's TV team found most of theirs that way). What only Duo shows: asymmetric side insets, the vertical bar, the fold. [LAB0916 21:13, 53:18; LAB0917 4:38]
- Test the pinned-PiP stacked layout and 50/50 Split View, and note that the vertical bar's height changes with rotation, a pinned video and Split View [LAB0917 50:59].
- Test Split View on both sides (drag the app by its home indicator in Device Hub; a drop zone appears), because the bar and insets switch sides. [TT461 7:44; LAB1013 46:05]
- Test transitions between poses, not only the poses. [LAB0917 43:40; LAB0916 42:38]
- Test with Reduce Transparency, RTL and the largest accessibility sizes.
- Xcode 27.1's "App Resizability" agent skill (formerly the UIKit modernization skill) now covers SwiftUI and iPhone Duo and migrates main-screen reads [TT461 9:19; LAB1013 44:33].
- A physical device matters for camera work and for judging ergonomics; most apps can be tested end to end in the simulator [LAB1013 11:46]. "Always validate button targets, vertical bar positioning, and navigation flows on physical iPhone Duo devices" [forum 847644].
- Simulator facts measured 2026-10-23: the iOS 27.1 runtime supports only the iPhone Duo device type (non-Duo controls run on 27.0); the Duo profile declares Touch ID, not Face ID, and model id `iPhone19,4`; `simctl io screenshot --mask=alpha` masks raw captures (cover corners 8 and 59, inner 55); display ids 1 (cover) and 3 (inner, native orientation 270) [simctl help; profile and capabilities plists].

## 14. Things Apple's first-party apps did (examples, not rules)

- Music opens into lyrics when unfolded and leaves them when closed [LAB0917 17:41].
- FaceTime weights its UI toward the inner camera and moves the self-view away from the active camera occlusion [LAB0917 29:51].
- Weather added a landscape two-column layout for Duo without arrangement views [LAB1015 45:11].
- Search stays at the bottom on both displays (unlike iPad's top right) [LAB1015 1:02:38].
- Live Activities expand from the bottom of the Dynamic Island on Duo and are narrower; check the constrained-width environment (value not named) [LAB1015 20:26-20:41]. The outer camera region "expands into the Dynamic Island for Live Activities" [HIG].
- Podcasts' now-playing view splits held wide and puts the transcript inline turned tall, each part in one region [TT463 9:36-10:26].
- Preview switches to horizontal two-page paging when folded like a book [LAB1015 47:27-48:30].
- Carousels sized from "the device width subtracting maybe some padding" should cap their aspect so a wider screen shows more content [LAB1013 29:00-29:27].
- AVKit video players stack content above controls when seated [LAB1013 37:27].
- Extra-large widgets fill half the open display in StandBy [LAB1013 15:34].

## 15. Games, web content, other app types

- Games: "Make your game playable in every device pose ... Prefer changing the aspect ratio over letterboxing or pillarboxing in games; if you can't avoid letterboxing or pillarboxing, add artwork to the padding area." Games may lock orientation provided they fill the screen [HIG, fetched 2026-10-23]. Support any aspect ratio and display size; move controls down when the device sits on a surface [LAB1013 58:05-59:03]; read the corner-adapted area for controls [LAB1013 38:10]; handheld console layouts via `ArrangementView(.overlay)` [LAB1013 36:39-37:31]; mini games may turn the vertical bar off [LAB1015 22:36].
- Web content: "web standards include support for hinge position and ... WebKit supports that", with no more detail [LAB1013 36:06-36:27]; look to existing JavaScript foldable standards, no WebKit engineer present [LAB0916 38:48-39:26]. Viewport Segments and Device Posture support in Duo WebKit is unverified; asymmetric safe areas apply to `env(safe-area-inset-*)` per side [HIG].
- Hardcoded sizes: "`bounds.height == 844` or a list of known iPhone sizes won't match iPhone Duo" [PDUO]. "Size your views relative to their container rather than to fixed iPhone dimensions"; "Don't use UIDevice.userInterfaceIdiom or UIInterfaceOrientation for layout decisions in your UIKit app" [PREP].
- The outer display "is actually wider than most other iPhones and shorter in the vertical height" [LAB1015 39:58-40:16].

## 16. App Store Connect and release

- Screenshot slot: cover 1398 x 2034 / 2034 x 1398, inner 2007 x 2853 / 2853 x 2007; no scaled fallback listed for the Duo; "Images can't include alpha channels or transparencies" [SHOT, fetched 2026-10-23].
- Requirement wording conflict: "Starting in April 2027, screenshots will be required for any app using the iOS 27.1 SDK or later" [SHOT] versus "any apps or games submitted will need to include screenshots for iPhone Duo" [NEWS; WHATS]. Plan for the stricter reading; recheck both.
- Submitting Duo screenshots adds the product-page badge [PDUO]. Standalone assets "are reviewed against your app's latest version" [ASCUP]. Do not misrepresent the interface [G233]; more than half of each device set must show the app in use, judged per localization [G233].
- ASC API display type: `APP_IPHONE_DUO`, read from the live API on 2026-10-22; missing from the published ScreenshotDisplayType page and OpenAPI spec 4.5.1 (`mitamaton-case-study.md` section 15). `BuildBundle` exposes `sdkBuild`, `platformBuild`, `minimumOsVersion` [BB].
- App previews: Duo accepts only 886 x 1920 / 1920 x 886, 15-30 s, 30 fps max; "If a preview video uses an aspect ratio different from your screenshots, it appears in the 'A Closer Look' section"; the spec labels 2034 x 1398 and 2853 x 2007 "19.5:9", likely a doc error [PREV].
- Removing a screenshot removes it for every size using scaled versions; assets can be deleted only while Prepared for Submission [ASCUP]. That a new version copies the previous sets forward is Mitamaton evidence only.
- Preview tool: check assets "on different devices and orientations ... including on iPhone Duo" [WHATS; PDUO]. Creative assets (header 21:9 3840 x 1646 or 16:9 5244 x 2950, search 3:2 or 16:9, videos 5-30 s muted by default, no alpha) [CRE]; keep key art in the template safe area [ABP].
- Featuring: "Submit a featuring nomination ... you can specify that your app is optimized for iPhone Duo, including support for all device poses" [PDUO]; minimum lead time 3 weeks; type and related apps cannot change after submitting [NOM].
- Promotional Text changes "without requiring an updated submission", 170 characters; What's New 4000 characters, localizable [PVI].
- TestFlight: external testing reviews the first build added ("Subsequent builds may not require a full review"); builds last 90 days [TF].
- Device available 2026-10-23 [NEWS].

---

## Sources

| Key | Source |
|---|---|
| HIG | https://developer.apple.com/design/human-interface-guidelines/designing-for-iphone-duo |
| PREP | https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo |
| RR | https://developer.apple.com/documentation/swiftui/reservedregion |
| RRF | https://developer.apple.com/documentation/swiftui/geometryproxy/reservedregions(kind:options:layoutdirectionbehavior:) and https://developer.apple.com/documentation/uikit/uiview/reservedregions(kind:options:) |
| AV | https://developer.apple.com/documentation/swiftui/arrangementview |
| UAV | https://developer.apple.com/documentation/uikit/uiarrangementviewcontroller |
| CAM | https://developer.apple.com/documentation/avkit/choosing-a-camera-by-the-direction-it-faces |
| ACC | https://developer.apple.com/documentation/avfoundation/registering-a-camera-capture-accessory-on-iphone-duo |
| doc | the symbol's page under https://developer.apple.com/documentation/swiftui/ or /uikit/ (for example `.../swiftui/view/toolbarverticalbehavior(_:)`, `.../swiftui/environmentvalues/toolbarverticaledge`, `.../swiftui/view/safeareabar(edge:alignment:spacing:content:)`, `.../uikit/uihingeinteraction`, `.../swiftui/cameracaptureaccessory`) |
| TT461 | Prepare your app for iPhone Duo, https://developer.apple.com/videos/play/tech-talks/111461/ |
| TT462 | Raise the bar with iPhone Duo, https://developer.apple.com/videos/play/tech-talks/111462/ |
| TT463 | Strike a pose with adaptive layouts on iPhone Duo, https://developer.apple.com/videos/play/tech-talks/111463/ |
| TT464 | Leverage multiple displays and scenes on iPhone Duo, https://developer.apple.com/videos/play/tech-talks/111464/ |
| TT465 | Build a great camera experience for iPhone Duo, https://developer.apple.com/videos/play/tech-talks/111465/ |
| TT466 | Design for iPhone Duo, https://developer.apple.com/videos/play/tech-talks/111466/ |
| LAB0916 | "iPhone Duo Group Lab \| 9/16/26 \| Meet with Apple", Apple Developer on YouTube, published 2026-09-17, https://youtu.be/0zp4gAgC6TI |
| LAB0917 | "iPhone Duo Group Lab \| 9/17/26 \| Meet with Apple", Apple Developer on YouTube, published 2026-09-17, https://youtu.be/zAaPDbDKvaU |
| LAB1013 | "iPhone Duo Group Lab \| 10/13/26 \| Meet with Apple", Apple Developer on YouTube, published 2026-10-14, https://youtu.be/GWz8siwQk8M |
| LAB1015 | "iPhone Duo Group Lab \| 10/15/26 \| Meet with Apple", Apple Developer on YouTube, published 2026-10-15, https://youtu.be/kmf3gkQMex4 |
| TN3187 | https://developer.apple.com/documentation/technotes/tn3187-migrating-to-the-uikit-scene-based-life-cycle |
| PDUO | https://developer.apple.com/iphone-duo/prepare/ (Apple's Prepare page, distinct from the PREP article) |
| SHOT | https://developer.apple.com/help/app-store-connect/reference/app-information/screenshot-specifications |
| PREV | https://developer.apple.com/help/app-store-connect/reference/app-information/app-preview-specifications |
| ASCUP | https://developer.apple.com/help/app-store-connect/manage-app-information/upload-app-previews-and-screenshots |
| G233 | https://developer.apple.com/help/app-review/guideline-reference/2-3-3-screenshots |
| BB | https://developer.apple.com/documentation/appstoreconnectapi/buildbundle |
| NEWS | https://developer.apple.com/news/?id=kkphp5qo (2026-10-05) |
| WHATS | https://developer.apple.com/app-store/whats-new/ |
| NOM | https://developer.apple.com/help/app-store-connect/manage-featuring-nominations/nominate-your-app-for-featuring/ |
| CRE | https://developer.apple.com/help/app-store-connect/reference/app-information/creative-assets-specifications |
| ABP | https://developer.apple.com/app-store/asset-best-practices/ |
| PVI | https://developer.apple.com/help/app-store-connect/reference/app-information/platform-version-information |
| TF | https://developer.apple.com/testflight/ |
| forum 847644 | Apple Developer Forums thread 847644, group-lab summary (fetched 2026-10-23) |
| SDK headers | Xcode 27.1 SDK `XCUIAutomation.framework/Headers/` (XCUIVoiceOverService.h, XCUIApplication.h, XCUIAccessibilityAuditTypes.h, XCUIScreen.h) |
| sim plists | `/Library/Developer/CoreSimulator/Profiles/DeviceTypes/iPhone Duo.simdevicetype/Contents/Resources/{profile,capabilities}.plist` |

The four group labs are part of Apple's "Design and build for iPhone Duo" playlist on the Apple Developer YouTube channel. Lab timestamps (minutes:seconds) are from the YouTube captions.
