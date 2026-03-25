# okCOMPUTER Comprehensive Audit

**Version:** 0.6.4 | **Date:** 2026-03-24 | **Purpose:** Catalog all bugs, UX issues, and architectural debt to inform the desktop app overhaul.

---

## Table of Contents

1. [Surface Catalog](#1-surface-catalog)
2. [Issue Registry](#2-issue-registry)
3. [User Journey Map](#3-user-journey-map)
4. [Responsibility Matrix](#4-responsibility-matrix)
5. [IPC Flow Map](#5-ipc-flow-map)
6. [Runtime Testing Checklist](#6-runtime-testing-checklist)
7. [Forward-Looking Summary](#7-forward-looking-summary)

---

## 1. Surface Catalog

### 1.1 HUD (Dictation Indicator)

**Role:** Always-visible floating overlay at bottom-center of screen. Shows real-time AI activity, dictation state, recording status, approvals, TTS playback, and proactive suggestions.

**Window:** Transparent, always-on-top, unfocusable panel (400px wide). Created at startup when authenticated.

**Layer Priority System:** `dictation > clarification > approval > error > ambient > idle`

**Components:**
- 10 pill types: Idle, Listening, Thinking, Tools, Response, TTS, TTSResponse, Error, RecordingIdle, DocumentToast
- 2 interrupt types: ApprovalCard, ClarificationForm
- 1 expanded view: CompressedTurn (AI response detail)
- 1 proactive view: SuggestionView (with markdown, sources, question input)

**Key Behavioral States:**
- Idle: 56px pill with drag handle + optional clipboard paste button
- Dictation: Soundwave visualization bars with mode label (cmd/reply/note)
- AI Processing: Thinking dots, tool execution icons, response text
- TTS: Waveform animation with optional pause/stop controls
- Recording: Red indicator with pause/stop/screen-share-hide controls
- Approval: Full card with approve/deny buttons and path access options
- Clarification: Multi-step form with checkboxes and text input
- Expanded: Hover-triggered expanded view showing full voice result or suggestion

**Architecture Issues:**
- `useDictationIPC.ts` is 717 lines managing ~20 state fields and ~10 refs — the single largest source of complexity
- `DictationIndicator.tsx` is 328 lines with 15+ state fields — violates the 150-line component rule
- `HUDLayerRendererProps` has 26 fields — severe prop drilling
- Three overlapping timer systems (hover 300ms, leave 350ms, voice/suggestion 30s) with no centralized state machine
- `setIgnoreMouseEvents` toggled from multiple places (hover hook, parent effect, repositioning) — race conditions possible

**Performance Issues:**
- `TimerBorder.tsx` runs `requestAnimationFrame` loop for 30 seconds, calling `setProgress` (React state) on every frame — ~1800 re-renders of the pill subtree on a transparent composited window
- `useAudioVisualization` runs 25fps state updates during dictation
- `DragHandle` runs RAF loop during 600ms hold for border-trace animation

**Critical Interaction Bugs:**
- ClarificationForm has a text input but the window has `focusable: false` — input may not be typeable (P0)
- Dismissing approval (X button) silently drops the tool call without notifying AI backend (P1)
- Dismissing clarification form does not notify AI backend — conversation stalls (P1)
- Errors auto-dismiss in 10s with no retry or details action (P1)
- `removeAllListeners` removes ALL channel listeners, not just this hook's (P2)

**Visual Issues:**
- `prose-compact` CSS class referenced in SuggestionView but undefined anywhere — markdown has no styling (P1)
- TTS pause/stop buttons are 20x20px — below 44px minimum (P1)
- Source title buttons in SuggestionView look clickable but are no-ops (P1)
- Fixed window sizes don't match content — clipping or empty space (P2)
- Jira tools show Linear icon (incorrect branding) (P2)
- Multiple components use tiny click targets (16-24px) on a floating overlay (P2)

### 1.2 Popover (Menu Bar Panel)

**Role:** Detail panel opened from tray icon click. Tabs: Home, Meetings, Notes-to-self, Dictation, okCOMPUTER. Contains onboarding wizard.

**Window:** Frameless, always-on-top `panel` type with vibrancy (400x740, resizable 360-480 wide). Lazy-created on first tray click.

**Mutual Exclusivity:** Popover and main window are never visible simultaneously.

**Tabs:**
- **Home:** Record + Note CTAs (or live transcript when recording), upcoming meetings, recent activity feed with filter pills and search
- **Meetings:** Detection alert, recording controls, search, upcoming meetings, past meetings
- **Notes-to-self:** Composer + searchable note list
- **Dictation:** Dictation status, shortcuts, dictation history
- **okCOMPUTER:** Computer-use activity log with grouped voice commands

**Onboarding (3 steps):**
1. System Permissions (mic, speech recognition, accessibility) with auto-recheck every 3s
2. Try Dictation Modes (must try all 3: Fn, Fn+Shift, Fn+Space — coercive gating)
3. Vault Info (auto-completes immediately — non-interactive filler)

**Critical Issues:**
- Onboarding teaches mechanics, not value — no aha moment (P0)
- Meeting recording (primary use case) absent from onboarding entirely (P0)
- DetectionAlert nested "Record" button has no onClick — only wrapper div is clickable (P0)
- `apiEnvironment` defaults to `'staging'` in preferences.ts (TODO not resolved) (P0)
- EnvironmentToggle (dev tool) visible to all unauthenticated users (P1)
- Tab content unmounts on switch — all local state (search, scroll, drafts) lost (P1)
- Must try all 3 dictation modes to proceed in onboarding — coercive (P1)
- System Preferences URL scheme may be deprecated on macOS 13+ (P1)
- "okCOMPUTER" header text looks like a title, not a navigable button (P1)
- Five tabs too many for 360-480px popover (P1)
- No clear primary action on HomeTab — Record and Note have equal weight (P1)
- Filter pills in Home duplicate the tab bar navigation (P1)
- Virtual/in-person recording toggle is a 14px icon — nearly invisible (P1)
- Recording controls duplicate what's on the HUD (P1)
- Resize handle invisible (opacity-0 + 1.5s delay) (P1)
- Clicking items: meetings open main window, notes/voice switch tabs — inconsistent (P1)
- MeetingsTab substantially overlaps with HomeTab (P1)
- Transcript and chat actions open external browser, leaving the app (P1)
- Draft lost when switching tabs (P1)
- Dictation and okCOMPUTER tab banners look like buttons but aren't clickable (P1)
- Clear confirmation modal exists but no button triggers it (dead code) (P1)
- `dangerouslySetInnerHTML` in VoiceCommandGroup — potential XSS (P1)
- Uses `localStorage` in PopoverFooter (violates "never use localStorage") (P1)
- `showInactive()` means no keyboard input until user clicks into popover (P1)

**Information Architecture Issues:**
- Home tab duplicates content from all other tabs
- Notes-to-self and Dictation tabs filter the same data source — could be merged
- Virtual vs in-person recording difference is unexplained
- Live transcript in popover is less useful than main window or HUD versions
- Only 2 shortcuts shown with no way to manage all
- Shortcuts section is read-only with no path to configuration

### 1.3 Main Window

**Role:** Full BrowserWindow loading okwow.ai web app. Used for meetings, settings, chats, and insights.

**Window:** 1200x800, hiddenInset titlebar, lazy-created on first show. Loads web app URL directly.

**Key Behaviors:**
- Navigation guards: only okWOW + OAuth origins allowed
- Window.open handler: internal links stay, external open in browser
- Logout interception via webRequest
- Cookie watcher for auth sync (accessToken cookie)
- 12px drag region injected via executeJavaScript

**Issues:**
- Settings requires full main window — no lightweight settings surface in popover or HUD (P1)
- Two completely different design systems (web DS vs desktop Tailwind tokens) (P1)
- Drag region injected via executeJavaScript could conflict with web app z-index (P2)
- Cmd+, always focuses main window, takes seconds to load settings (P2)
- Settings unreachable without selected workspace (P2)
- No shared component library across windows (P2)

### 1.4 Notification Window

**Role:** Ephemeral toast notifications at top-right. Shows meeting detection, voice results with TTS, errors, and actions.

**Window:** Frameless, transparent, always-on-top panel (358x104 base, expands). Lazy-created on first notification.

**Notification Types:** upcoming-meeting, meeting-detected, voice-command-result, command-approval, tool-status, tool-confirmation, info, error

**Key Behaviors:**
- Auto-dismiss: 15s default, overridden per notification
- TTS linger: 10s after playback finishes
- Expand on hover: shows full markdown content + TTS controls
- Text input mode: replaces card for reply
- Countdown bar: visual timer for auto-dismiss

**Critical Duplication with HUD:**
- TTS playback visualization shown in BOTH notification and HUD simultaneously (P0)
- TTS controls exist in BOTH notification (TTSControlBar) and HUD (TTSResponsePill) (P1)
- Voice result text displayed in BOTH notification (expanded card) and HUD (CompressedTurn) (P1)
- Auto-listening indicator shown in BOTH notification and HUD (P1)

**Other Issues:**
- Countdown bar pauses on hover but never resumes — freezes until dismiss (P2)
- Text input fixed at 140px, no auto-grow (P2)
- Dropdown positioning may overflow notification window (P2)
- "Open Chat" navigates silently — user may not realize they've been switched (P3)

---

## 2. Issue Registry

### P0 — Blocks Users

| ID | Surface | Category | Issue |
|----|---------|----------|-------|
| H-1 | HUD | architecture | `useDictationIPC.ts` is 717 lines managing ~20 state fields — unmaintainable |
| H-2 | HUD | interaction | ClarificationForm text input in unfocusable window — input may not be typeable |
| P-1 | Popover | onboarding | Onboarding teaches mechanics, not value — no aha moment |
| P-2 | Popover | onboarding | Meeting recording (primary use case) absent from onboarding |
| P-3 | Popover | interaction | DetectionAlert nested "Record" button has no onClick — only wrapper clickable |
| P-4 | Popover | architecture | `apiEnvironment` defaults to `'staging'` — TODO not resolved |
| N-1 | Notification | info-arch | TTS playback visualization in BOTH notification and HUD simultaneously |

### P1 — Confuses Users

| ID | Surface | Category | Issue |
|----|---------|----------|-------|
| H-3 | HUD | architecture | DictationIndicator.tsx is 328 lines with 15+ state fields |
| H-4 | HUD | interaction | setIgnoreMouseEvents race: brief click-through gap during transitions |
| H-5 | HUD | interaction | Three overlapping timer systems with no centralized state machine |
| H-6 | HUD | architecture | TTS audio plays during dictation with no visual indicator |
| H-7 | HUD | interaction | Dismissing approval silently drops tool call without notifying backend |
| H-8 | HUD | interaction | onMouseLeave sets click-through even when HUD should remain interactive |
| H-9 | HUD | visual | TTS pause/stop buttons 20x20px — below 44px minimum |
| H-10 | HUD | info-arch | Errors auto-dismiss in 10s with no retry/details action |
| H-11 | HUD | performance | TimerBorder RAF loop for 30s = ~1800 state updates |
| H-12 | HUD | interaction | No keyboard support in ApprovalCard — no Tab order, no Enter |
| H-13 | HUD | info-arch | Dismissing clarification doesn't notify AI backend — conversation stalls |
| H-14 | HUD | visual | `prose-compact` CSS class undefined — suggestion markdown unstyled |
| H-15 | HUD | interaction | Source title buttons in SuggestionView are no-ops |
| P-5 | Popover | onboarding | EnvironmentToggle (dev tool) visible to all users |
| P-6 | Popover | architecture | Tab content unmounts on switch — local state lost |
| P-7 | Popover | onboarding | Must try all 3 dictation modes to proceed — coercive |
| P-8 | Popover | onboarding | System Preferences URL may be deprecated on macOS 13+ |
| P-9 | Popover | interaction | "okCOMPUTER" header looks like title, not navigation |
| P-10 | Popover | info-arch | 5 tabs too many for 360-480px |
| P-11 | Popover | info-arch | No clear primary action on HomeTab |
| P-12 | Popover | info-arch | Home filter pills duplicate tab bar |
| P-13 | Popover | interaction | Virtual/in-person toggle is 14px icon — invisible |
| P-14 | Popover | info-arch | Recording controls duplicate HUD |
| P-15 | Popover | interaction | Resize handle invisible |
| P-16 | Popover | info-arch | Item clicks: meetings open main, notes switch tabs — inconsistent |
| P-17 | Popover | info-arch | MeetingsTab overlaps HomeTab |
| P-18 | Popover | info-arch | Actions open external browser, leaving app |
| P-19 | Popover | interaction | Note draft lost on tab switch |
| P-20 | Popover | interaction | VoiceTab + ComputerTab banners look clickable but aren't |
| P-21 | Popover | architecture | Clear modal unreachable — dead code |
| P-22 | Popover | interaction | `dangerouslySetInnerHTML` potential XSS in VoiceCommandGroup |
| P-23 | Popover | architecture | Uses localStorage (violates rules) |
| P-24 | Popover | architecture | showInactive() prevents keyboard input until click |
| P-25 | Popover | architecture | Tray fallback icon is unlabeled 18px circle |
| M-1 | Main | info-arch | Settings requires full main window — no lightweight access |
| M-2 | Main | visual | Two different design systems (web DS vs desktop tokens) |
| N-2 | Notification | info-arch | TTS controls duplicated with HUD |
| N-3 | Notification | info-arch | Voice result displayed in BOTH notification and HUD |
| N-4 | Notification | info-arch | Auto-listen indicator in BOTH notification and HUD |
| P-26 | Popover | info-arch | RecordMeetingButton virtual vs in-person difference unexplained |
| P-27 | Popover | interaction | ResizableTranscriptPanel resize handle: invisible with 1.5s delay |
| P-28 | Popover | interaction | Meeting actions (transcript, chat) use openExternal instead of main window |

### P2 — Looks Bad / Friction

*(98 additional P2 issues documented across surfaces — see detailed surface audits above. Key themes: small click targets, inconsistent typography, arbitrary Tailwind values, missing hover affordances, inconsistent navigation patterns, cross-window component imports, magic numbers, missing confirmations, memory leaks.)*

### P3 — Minor

*(~40 P3 issues documented — dead code, naming inconsistencies, defensive patterns, index-as-key, minor perf items.)*

**Totals:** 7 P0 | 38 P1 | ~98 P2 | ~40 P3

---

## 3. User Journey Map

### 3.1 First Launch

```
Install app → Double-click
  ↓
macOS permission prompts (no context provided)
  ↓
Tray icon appears in menu bar (only visual cue)
  ↓
User must discover tray icon (no tooltip, no bounce)
  ↓
Click tray → Popover opens (lazy-created)
  ↓
"Sign in to get started" + [Open okCOMPUTER] button
  + EnvironmentToggle (dev tool visible to all) ← BUG
  ↓
Main window opens → okwow.ai login page
  ↓
OAuth flow (browser round-trip)
  ↓
Auth completes → completeLogin()
  → HUD appears (bottom-center)
  → Main window shows web app (focused)
  → Popover updates with auth
```

**Gap:** After auth, the user lands on the full web app main window. The popover (with onboarding) is hidden behind it. The user must re-click the tray to discover onboarding.

### 3.2 Onboarding

```
Click tray → Popover shows OnboardingView
  ↓
Step 0: System Permissions
  → Grant mic, speech recognition, accessibility
  → Polls every 3s, deeplinks to System Preferences
  → ⚠️ URL scheme may fail on macOS 13+
  ↓
Step 1: Try Dictation Modes
  → Must try all 3: Fn (dictate), Fn+Shift (voice note), Fn+Space (command)
  → ⚠️ Coercive: blocks progress even if user only wants meetings
  → No explanation of WHY each mode matters
  ↓
Step 2: Vault Info
  → Static info card. Auto-completes immediately.
  → "Paste from clipboard", "Meeting highlights saved automatically"
  → ⚠️ Not interactive — wasted step
  ↓
"Get started" → onboardingCompleted: true → Home tab
  ↓
HOME TAB: Record button + Note button + upcoming meetings + recent activity
  → ⚠️ No guidance on what to do next
  → ⚠️ No introduction to meeting recording
  → ⚠️ Dead end — the product story never starts
```

### 3.3 Feature Discovery

| Feature | How Discovered | Gap |
|---------|---------------|-----|
| Voice dictation | Onboarding Step 1 (Fn key) | Teaches mechanic, not value |
| Voice notes | Onboarding Step 1 (Fn+Shift) | Taught but never shown result |
| AI commands | Onboarding Step 1 (Fn+Space) | Taught but never shown result |
| Meeting recording | Home tab red button | Not in onboarding at all |
| Calendar sync | Meetings tab prompt | Only if user navigates there |
| Computer use | Computer tab | No introduction, just activity log |
| Proactive intelligence | HUD suggestion appears during meetings | Completely emergent — no setup |
| Notes | Notes tab | No trigger to go there |
| Settings | Cmd+, or popover gear icon | Opens full web app — heavyweight |

### 3.4 Daily Use Flows

**Meeting Recording:** Detect meeting (Recall SDK) → notification toast or popover alert → user clicks Record → live transcript in popover → user clicks Stop → summary generated → synced to okwow.ai

**Voice Dictation:** Hold Fn → sidecar speech recognition → text inserted at cursor → release Fn

**AI Command:** Fn+Space → speak command → HUD shows thinking/tools → response in HUD + notification → TTS speaks result

**Approval:** AI command needs permission → HUD shows ApprovalCard → user approves/denies → command executes or rejected → result in HUD

**Note:** Open popover → Notes tab → click composer → type → save. Or: Fn+Shift → speak → note saved.

---

## 4. Responsibility Matrix

| Information / Action | HUD | Popover | Main Window | Notification | Duplication? |
|---------------------|:---:|:-------:|:-----------:|:------------:|:------------:|
| Recording state indicator | ✅ | ✅ | ✅ | — | 3-way broadcast (intentional) |
| Recording controls (pause/stop) | ✅ | ✅ | — | — | **YES** |
| Meeting detected alert | — | ✅ | ✅ | ✅ | 3-way (different actions) |
| Live transcript | — | ✅ | ✅ | — | 2-way |
| Meeting list | — | ✅ | ✅ | — | 2-way |
| Calendar / upcoming | — | ✅ | ✅ | toast | — |
| Start recording | — | ✅ | — | ✅ (action) | 2-way |
| Dictation state | ✅ | ✅ | ✅ | — | 3-way broadcast |
| Dictation history | — | ✅ | — | — | — |
| AI thinking/tools | ✅ | ✅ | — | — | 2-way |
| AI voice result | ✅ | — | — | ✅ | **YES — key duplication** |
| TTS playback state | ✅ | — | — | ✅ | **YES — key duplication** |
| TTS controls | ✅ | — | — | ✅ | **YES — key duplication** |
| Auto-listen indicator | ✅ | — | — | ✅ | **YES** |
| Command approval | ✅ | ✅ | — | — | 2-way |
| Activity log | — | ✅ | ✅ | — | 2-way |
| Settings | — | navigate | ✅ | — | — |
| Onboarding | — | ✅ | — | — | — |
| Proactive suggestions | ✅ | — | — | — | — |
| Text reply to chat | — | — | — | ✅ | — |
| Notes CRUD | — | ✅ | — | — | — |

**Key duplications to resolve:**
1. Voice result: HUD (CompressedTurn) + Notification (expanded card) — user sees same response in two places
2. TTS controls: HUD (TTSResponsePill pause/stop) + Notification (TTSControlBar) — two control surfaces
3. Recording controls: HUD (RecordingIdlePill) + Popover (RecordingControls) — two pause/stop surfaces
4. Auto-listen: HUD shows "listening" + Notification shows "Listening..." pulse — confusing

---

## 5. IPC Flow Map

### Broadcast Patterns

```
Recording state change
  → broadcastRecordingState()
    → Main window (send)
    → Popover (send)
    → HUD (send)

AI message event
  → IF main window focused: skip HUD
  → ALWAYS: send to Popover
  → IF main NOT focused: send to HUD

TTS state change
  → Notification window (sendTTSState)
  → HUD (send)

Approval resolved
  → broadcastApprovalResolved()
    → HUD (if main not focused)
    → Popover
    → Deduplication via recentlyBroadcasted Set
```

### Cross-Surface Navigation

| Trigger | From → To | How |
|---------|-----------|-----|
| Click "okCOMPUTER" header | Popover → Main | `window:open-main` IPC, popover hides |
| Click meeting card | Popover → Main | `window:open-main-with-meeting`, popover hides |
| Click "Open Chat" | Notification → Main | `window:open-chat`, notification dismisses |
| Click dock icon | macOS → Main | `app.on('activate')`, popover hides |
| Cmd+, | Any → Main | Global menu accelerator, main focuses |
| Tray click | Any → Popover | Popover toggles, main hides if visible |

### IPC Domains

17 handler domains in `src/main/ipc/handlers/`: recording, auth, meetings, calendar, ai, sync, window, app, dictation, commands, shortcuts, preferences, folders, vocabulary, apps, style, tools, intelligence.

---

## 6. Runtime Testing Checklist

These scenarios require running the app to verify:

- [ ] **Fresh launch (no auth):** Walk through entire first-run flow. Does the user know what to do? Can they find the tray icon?
- [ ] **Onboarding completion:** After completing all 3 steps, does the user know what to do next? Is there a dead end?
- [ ] **Record a meeting:** Start Zoom/Meet/Teams. Does detection fire? Does the notification appear? Can the user start recording? Are recording controls discoverable?
- [ ] **Voice command + result:** Fn+Space, speak a command. Where does the result appear? Are both HUD and notification showing the same thing? Is TTS playing from both surfaces?
- [ ] **Approval during recording:** While recording a meeting, trigger a command that needs approval. Does the HUD show the approval card? Does it interfere with the recording indicator?
- [ ] **Proactive suggestion during meeting:** During a recording, wait for a proactive suggestion. Does the HUD show it? Can the user interact with it? Does the question input work?
- [ ] **HUD hover during TTS:** While TTS is speaking, hover over the HUD. Does it expand? Can you pause/stop? Does leaving the HUD cause it to collapse while TTS is still playing?
- [ ] **Popover while notification showing:** Open the popover while a notification is visible. Do they compete for attention? Which surface is primary?
- [ ] **macOS Space switching:** While the HUD is visible, switch to a full-screen app. Is the HUD visible on other spaces? Does it interfere with fullscreen?
- [ ] **Stage Manager:** With Stage Manager enabled, open the popover. Does it appear in the correct stage? Does it interfere with app grouping?
- [ ] **HUD reposition:** Long-press the HUD drag handle. Does repositioning work smoothly? Does the position persist across restart?
- [ ] **ClarificationForm input:** Trigger a clarification form on the HUD. Can you actually type in the text input? (focusable: false issue)
- [ ] **Tab switching state loss:** In popover, search for something in Notes tab, switch to Meetings tab, switch back. Is the search query preserved?
- [ ] **DetectionAlert Record button:** When a meeting is detected, click the visible "Record" text in the alert. Does it work? (Known bug: only wrapper div is clickable)
- [ ] **Virtual/in-person toggle:** In the Record meeting button, find and click the Monitor icon toggle. Can you see it? Is the behavior difference clear?

---

## 7. Forward-Looking Summary

### 7.1 Proposed Surface Roles (HUD-as-Dock Model)

| Surface | Role | One-Sentence Purpose |
|---------|------|---------------------|
| **HUD** | The Dock | The always-present entry point: shows real-time activity, expands for responses/approvals, collapses to a compact indicator, single-click opens panel, double-click opens main app. |
| **Menu Bar Panel** | The Console | The HUD's detail view: settings, history, configuration, onboarding, and anything requiring extended interaction or browsing. |
| **Main Window** | The Workspace | Deep content: full transcripts, chat threads, insights, account settings. Opens explicitly, never unbidden. |

**Notification Window: ELIMINATE.** All notification types absorbed by HUD (ephemeral alerts, TTS state) and Menu Bar Panel (actions requiring interaction, meeting detection prompts).

### 7.2 Candidate "Wow Moments"

1. **First dictation that actually types:** Hold Fn, speak, watch text appear in their active app. Requires: text insertion working, active app detected. Impact: immediate, tangible, magical.

2. **First voice command that does something:** "Hey okCOMPUTER, what's on my calendar today?" → response read aloud with TTS. Requires: calendar connected, AI backend responsive. Impact: "it understands me and can do things."

3. **First meeting summary:** After a 30-min meeting, get an AI-generated summary with key moments. Requires: full recording + transcription + summary. Impact: "this saves me 30 minutes of notes." (Delayed — not immediate)

4. **First proactive suggestion:** During a meeting, okCOMPUTER surfaces a relevant document from the vault. Requires: populated vault, active meeting, relevant content. Impact: "it's paying attention." (Emergent — can't be staged in onboarding)

**Recommendation:** Wow moment #1 (dictation) for the onboarding, with #2 (voice command) as the follow-up. These are immediate, require minimal setup, and demonstrate the HUD-as-dock concept directly.

### 7.3 Information Architecture Proposal

**Move to HUD:**
- Voice results (currently also in notification) — HUD is the primary AI response surface
- TTS state + controls — unified in HUD, no second surface
- Meeting detection brief alert — HUD attention indicator
- Error display with action capability (retry, details)

**Move to Menu Bar Panel:**
- Recording controls (currently duplicated in HUD + popover) — panel is the control surface
- Meeting detection action card (Record button) — panel is the interactive surface
- Full conversation history — scrollable, searchable
- Settings section (lightweight, desktop-specific)
- Onboarding wizard
- Notes/dictation/shortcuts management

**Keep in Main Window:**
- Full meeting transcripts
- Chat threads
- Insights
- Account/workspace settings

### 7.4 Notification Elimination Plan

| Current Notification Type | New Home |
|--------------------------|----------|
| `meeting-detected` | HUD: amber attention pill → Panel: auto-opens with Record card |
| `voice-command-result` | HUD: response pill + TTS waveform (no separate notification) |
| `command-approval` | HUD: approval card (stays, but improved) |
| `upcoming-meeting` | HUD: brief indicator → Panel: upcoming section |
| `error` | HUD: error pill with retry capability |
| `info` / `tool-status` | HUD: brief toast pill (auto-dismiss) |

### 7.5 Progressive Disclosure Tiers

**Tier 0 — Presence:** Install → tray icon + HUD idle pill. Sign in.

**Tier 1 — Voice:** Onboarding teaches Fn key. First dictation = wow moment. HUD shows listening/thinking/response. This is the hook.

**Tier 2 — Memory:** Voice notes start accumulating. Proactive suggestions begin (requires vault content). User learns: okCOMPUTER remembers.

**Tier 3 — Meetings:** Calendar sync prompt (contextual, not forced). First meeting recording + summary. User learns: okCOMPUTER listens with you.

**Tier 4 — Agency:** AI commands gain tool access. Permission tier unlocks from Restricted to Default. Approval cards appear. User learns: okCOMPUTER can act.

**Tier 5 — Proactive:** Emergent behavior as vault fills. App context suggestions, calendar reminders. Not a feature to "unlock" but a behavior that compounds.

**Story arc:** "Meet your voice" → "Build your memory" → "Record your conversations" → "Let it act" → "It starts anticipating you."
