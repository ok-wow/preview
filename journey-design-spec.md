# okCOMPUTER Journey Design Spec

**Version:** 1.0 | **Date:** 2026-03-25 | **Status:** Design Complete — Ready for Implementation Planning

---

## Context

okCOMPUTER v0.6.4 has four fragmented UI surfaces (HUD, popover, main window, notification window) with 183 documented issues across visual design, interaction, architecture, performance, and information architecture. The core problems: users don't know where to look, the same info appears in multiple surfaces, there's no coherent journey from signup through feature discovery, and the onboarding teaches mechanics instead of value.

This spec defines the redesigned user journey: a 3-surface model with the HUD as the primary interface ("the dock"), an RPG-style "Build Your Chief of Staff" onboarding, and progressive feature disclosure.

---

## 1. Surface Architecture: 2 Surfaces (HUD/Panel is One)

### 1.1 HUD/Panel — "The Dock"

**Single responsibility:** The always-present entry point. Shows real-time activity, expands for responses and actions, collapses to a compact indicator. **The HUD and panel are the same window.** The HUD morphs into the full panel view — pill → card → full panel as one continuous shape. Clicking the HUD expands it into the panel. The tray icon also triggers the same expansion. There is no separate "popover window."

**The HUD is one continuous shape** that morphs between states. When expanded, the entire pill becomes the content card, which can further expand into the full panel. It animates from pill (9999px radius, compact) to card (16px radius, content-sized) to full panel as a single element.

#### States

| State | Visual | When | Mouse Behavior |
|-------|--------|------|----------------|
| **Dormant** | Small pill (56px) with status text | Nothing happening (95% of time) | Click-through except pill |
| **Idle (hover)** | Pill shows agent avatar + "okCOMPUTER" | Hover over dormant pill | Interactive. Click → panel. Double-click → main app |
| **Active** | Waveform bars, thinking dots, tool icons | Dictating, AI processing, tools executing | Click-through for background |
| **Expanded** | Full content card (responses, approvals, suggestions) | AI responds, approval needed, TTS playing | Fully interactive |
| **Recording** | Persistent red indicator + time | Meeting recording active | Coexists with all other states as badge |

#### Click Interactions

| State | Single Click | Double Click | Long Press |
|-------|-------------|-------------|------------|
| Dormant | Re-show last content or open panel | Open main window | Enter reposition mode |
| Idle | Open panel | Open main window | Enter reposition mode |
| Expanded | Interact with content | — | — |
| Recording | Expand to show controls | Open main window | Enter reposition mode |

#### Status Messages (Dormant Pill)

The dormant pill is not just a bar — it shows contextual status text:
- `● okCOMPUTER` (idle, connected)
- `● Meeting in 2 min` (amber, pulsing)
- `● Recording · 12:34` (red, pulsing)
- `● Thinking...` (purple)
- `● Adding note...` (purple)
- `● Note saved ✓` (green, brief)
- `● Processing request...` (purple)
- `● Failed — tap to retry` (red)

#### Auto-collapse

- Voice result: 30s timeout, hover resets
- Suggestion: 30s timeout
- Approval: persists until resolved
- Recording: persists until stopped
- All: collapse to dormant on timeout

#### Performance

- No framer-motion `layout` prop (RAF polling)
- TimerBorder must use ref + direct DOM manipulation, not React state per frame
- Audio visualization gated on `isDictating` only
- Transparent window: minimize repaints

#### Screen Share

HUD and panel hidden from screen share participants via `setContentProtection(true)`.

### 1.2 Panel View (HUD Fully Expanded) — "The Console"

**The panel is the HUD expanded to full size.** Same window, same surface. Settings, history, configuration, onboarding, and anything requiring extended interaction.

**Triggered by:** Tray icon click, HUD single-click (when dormant/idle), or automatically when attention items arrive.

**Structure:** Single scrollable view with contextual sections. No tabs (except during recording).

#### Sections (Default State)

1. **Header** — Agent avatar + name, gear icon for settings
2. **Upcoming** (conditional) — Next 2-3 calendar meetings with time. Alert banner for meetings in < 5 min.
3. **Recent Activity** (always visible) — Unified chronological feed of meetings, notes, dictations, commands. Filter chips: All | Meetings | Notes | Commands.
4. **Keyboard Shortcuts** (compact) — `fn` Dictate · `fn`+`⇧` Voice note · `fn`+`⎵` Ask
5. **Quick Actions** (fixed at bottom) — Three buttons: 🎙 Record | 📝 Note | 💬 Chat (Option B labels)

#### Sections (Recording State)

Panel transforms entirely for the meeting context:

1. **Header** — Meeting title, platform, participants, duration. REC badge. Minimize button (—).
2. **Two full-width tabs** — "Meeting" and "Insights" (purple dot when new insights available). Tabs span full width using existing design system tab component.
3. **Meeting tab:**
   - Upcoming meeting alert (if next meeting < 5 min)
   - Live transcript (scrollable, speaker-labeled)
   - Scratchpad (fixed at bottom — user types meeting notes)
4. **Insights tab:**
   - Vault suggestions (related documents surfaced proactively)
   - Action items detected
   - Key moments
   - All items → "Open in vault →" on rollover
5. **Bottom controls** (both tabs) — ⏸ Pause | ⏹ Stop (same size, side by side)
6. **Privacy indicator** — "👁‍🗨 Hidden from screen share"

**Pause vs Mute:** Button is **Pause** (stops all transcription). Mute is platform-specific (Zoom/Meet) and doesn't control Recall SDK audio capture.

**Minimizable:** Panel can be collapsed via minimize button. When minimized, HUD pill shows recording status + time. Click tray or HUD to reopen.

#### Sections (After Voice Command)

Panel shows latest conversation context at top:
- User's question + agent's response
- Tool execution summary (service icons)
- "Open conversation →" link to main app
- Below: normal recent activity feed

#### Rollover Destinations

Every object in the panel links to the main app:

| Object | Rollover | Destination |
|--------|----------|-------------|
| Meeting card | Open in app → | Main window → meeting detail |
| Voice note | Open in vault → | Main window → vault → note |
| Command card | Open conversation → | Main window → conversation thread |
| Upcoming meeting | Open → | Main window → meeting prep view |
| Proactive insight | Open in vault → | Main window → vault → source doc |
| Dictation card | Open in vault → | Main window → vault → entry |

#### Settings

Gear icon in header → panel slides to settings view (same panel):
- Dictation language/sensitivity
- TTS voice/speed/enabled
- Permission tier
- Keyboard shortcuts
- HUD position/appearance
- Calendar sync
- "Account settings →" link to main app

### 1.2 Main Window — "The Workspace"

**Single responsibility:** Deep content requiring a full-size viewport.

**Never opens during onboarding.** First open is earned — user clicks something in the panel that needs the workspace (transcript, vault item, conversation).

- Loads okwow.ai web app
- Navigation guards (okWOW + OAuth origins only)
- Opens explicitly, never unbidden
- Panel provides escape hatches (rollover links) into the main window

### 1.3 Notification Window — ELIMINATED

All notification types absorbed:

| Current Type | New Home |
|-------------|----------|
| `meeting-detected` | HUD: amber attention pill → Panel: auto-opens with Record card |
| `voice-command-result` | HUD: expanded card with response + TTS controls |
| `command-approval` | HUD: expanded approval card |
| `upcoming-meeting` | HUD: status text "Meeting in 2 min" → Panel: upcoming section |
| `error` | HUD: error pill with retry capability |
| `info` / `tool-status` | HUD: brief status text (auto-dismiss) |

---

## 2. Onboarding: Build Your Chief of Staff

### Philosophy

> You're not onboarding a user. You're helping them create a character — their AI chief of staff.

The setup is the tutorial level. The interview IS the first real interaction. Permissions happen in the background as "agent setup." The empty prompt box problem is solved because the agent speaks first.

**Setting expectations:** "For the first few days, [agent name] will be learning how you work. The more you use it, the better it gets."

### The Flow

#### Step 0: Auth + Install
User signs up on okwow.ai, downloads desktop app, installs. App launches, tray icon appears, HUD shows dormant pill. Click tray → panel → login flow.

#### Step 1: Build Your Agent (HUD expands to panel, ~2 min)

The HUD expands into the full panel view to show the agent creation screen. This is the first time the user sees the HUD "open up" into its panel form.

**Creation fields:**
1. **Name** — text input, default: "Indi"
2. **Avatar** — preset options (bitmap, vector, photo styles) + upload own + generate with AI (nanobana/Gemini). Default avatars provided.
3. **Voice preference** — "How should [name] communicate?" Voice (TTS) or Text
4. **Your role** — text input: "What's your role?" (e.g., "Founder/CEO", "Engineering Manager")
5. **Work style** — "What does your day look like?" Meetings-heavy / Deep work / Mix of both

**In parallel (HUD shows assembly checklist):**
- Permissions requested with human explanations:
  - "Microphone — so you can record meetings and take voice notes"
  - "Speech Recognition — so [name] can understand you"
  - "Accessibility — so dictated text goes into your apps"
- Calendar connect: "Connect Google Calendar so [name] can prepare for your meetings"
- Each permission: checkmark appears in HUD as granted
- Keyboard shortcuts taught as they become available: "✓ Microphone ready — hold `fn` to dictate"

#### Step 2: The Interview (In panel view, 2-5 min, skippable)

Agent speaks first. No empty prompt box. Available via voice (HUD shows waveform when user holds fn) OR text (chat input in panel). Follows Indi v7.0 principles:

1. Agent researches user (web search, connected tools if available)
2. Opens with a specific observation + genuine curiosity (not "how can I help?")
3. Follows user's thread through 3-5 turns
4. Mirrors tone — casual if casual, formal if formal
5. User can ask questions about capabilities
6. Context gathered feeds into agent's memory

**Voice or text:** "Would you prefer to chat via voice?" If voice, user holds fn to speak, agent responds via TTS. If text, standard chat input.

**Skip option:** "Skip — I'll do this later →" always visible. Context builds gradually through usage instead.

**HUD during interview:** Shows agent avatar + status ("Indi is listening", waveform when user speaks).

#### Step 3: First Real Action — Not "You're Set"

After the interview (or skip), the agent guides the user to actually DO something — build, explore, or think through an idea. The goal is to show the power of what they can do, not just tell them they're ready.

- **If interview happened:** Agent has context. Proposes a specific action: "You mentioned the API migration — want me to pull up the relevant docs and start a conversation about the approach?" or "Your next meeting is in 2 hours — want to record it and I'll surface insights?"
- **If skipped:** Agent proposes a low-friction first action: "Try asking me something — hold `fn`+`space` and say what's on your mind right now." or "I noticed you have a meeting at 2pm — want me to record it?"

The onboarding IS the first real use. The agent keeps guiding from here — progressively introducing features as the user works.

**When does the main app open?** Not during onboarding. The first app open happens when the user clicks something in the panel that needs the full workspace — a meeting transcript, vault item, or conversation thread.

### Phase 2 (Future): Agent Identity

- Give the agent an email address and password
- Provision emails for the agent as a communication channel
- Agent can sign up for services (Slack, Amazon) and receive emails
- Avatar generation via nanobana/Gemini: bitmap, photorealistic, vector styles

---

## 3. Progressive Disclosure

Features unlock based on usage, not time. Tiers are implicit — no UI saying "you unlocked tier 3."

### Tier 1: Voice (after setup)
- Dictation (Fn → text insertion)
- Voice notes (Fn+Shift → save to vault)
- Basic voice commands (Fn+Space → agent responds)
- Panel: recent activity, keyboard shortcuts
- HUD: listening, thinking, response states

### Tier 2: Meetings (calendar connected or first recording)
- Meeting recording with live transcript
- Panel transforms to recording view (Meeting/Insights tabs)
- Calendar integration (upcoming meetings section)
- Meeting summaries synced to vault
- Proactive intelligence begins (vault matches during meetings)

### Tier 3: Agency (after 5+ commands, permission tier adjusted)
- AI commands with tool access (file ops, browser, GitHub, etc.)
- Approval cards for sensitive actions
- Permission tier settings exposed in panel
- Command history in recent activity

### Tier 4: Proactive (emerges with usage)
- Context-aware document surfacing
- Proactive suggestions in HUD during meetings
- Insights tab populated with relevant vault matches
- Not a feature to "unlock" — compounds with vault content

### Story Arc

**Meet your voice → Build memory → Record conversations → Let it act → It anticipates you**

### Bypass

"Skip — Unlock Everything" always available for power users. No one is punished for wanting full access immediately.

---

## 4. Key Design Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Surface count | 2 (HUD/panel is one window, + main window) | HUD morphs into panel. Notification eliminated. |
| HUD expansion | Single morphing shape | No pill-below-overlay; one continuous element |
| Panel structure | Contextual sections, no tabs | Eliminates 5-tab redundancy; sections appear/disappear based on state |
| Recording panel | Two tabs (Meeting + Insights) | Focused on meeting context; transcript + scratchpad + insights |
| Recording button | Pause (not mute) | Mute is platform-specific; pause clearly stops all transcription |
| Quick actions | Fixed at bottom | Separation between content and actions; always accessible |
| Action labels | Option B: Record / Note / Chat | Compact, icon-reinforced, learnable quickly |
| Onboarding | RPG agent creation + interview | Teaches value through interaction, not feature tour |
| Permissions | Human explanations during agent setup | "So you can record meetings" not just "grant microphone" |
| First app open | Earned via panel click | Main window never opens during onboarding |
| Progressive disclosure | Usage-based, implicit tiers | No gates; features introduced when relevant |
| Insights destination | Open in vault | Simpler, less context-switching; app has sharing functionality |
| Panel minimizable | Yes, during recording | HUD shows recording status when panel minimized |
| Screen share | HUD + panel hidden | Privacy protection for meeting participants |

---

## 5. Critical Files to Modify

### Eliminate
- `src/renderer/windows/notification/` — entire directory
- `src/main/windows/notification.ts` — NotificationWindow class
- All `notification:*` IPC channels

### Major Rewrites
- `src/renderer/windows/dictation-indicator/DictationIndicator.tsx` — HUD dock state machine (replace 328-line god-component)
- `src/renderer/windows/dictation-indicator/hooks/useDictationIPC.ts` — decompose 717-line god-hook
- `src/renderer/windows/popover/PopoverApp.tsx` — replace tab routing with contextual sections
- `src/renderer/windows/popover/components/OnboardingView.tsx` — replace 3-step wizard with agent creation flow
- `src/main/windows/dictation-indicator.ts` — HUD window management for morphing behavior
- `src/renderer/windows/dictation-indicator/HUDShell.tsx` — single morphing container (no pill + overlay)
- `src/renderer/windows/dictation-indicator/types.ts` — new HUD state types, eliminate 26-prop interface

### New Files Needed
- Agent creation components (name, avatar, voice preference, role, work style)
- Panel section components (AttentionBar, QuickActions, ActiveContext, Upcoming, RecentFeed)
- Recording panel components (MeetingTab, InsightsTab, Scratchpad)
- Interview chat component (for onboarding conversation)
- HUD status text resolver (contextual status messages)
- Onboarding store (agent profile, setup progress, interview context)

### Reuse
- `src/renderer/index.css` — design tokens (extend, don't replace)
- `src/renderer/stores/app-store.ts` — Zustand store (extend with agent profile)
- `src/main/ipc/handlers/` — IPC handlers (redirect notification channels to HUD/panel)
- `src/main/services/` — all services remain (auth, sync, sidecar, dictation, commands)
- `src/main/recall/` — Recall SDK integration unchanged

---

## 6. Verification Plan

### End-to-End Tests
1. **Fresh install → onboarding**: App launches → tray icon → click → login → agent creation in HUD → permissions granted → interview → panel shows normal state
2. **Dictation flow**: Hold Fn → HUD morphs to active (waveform) → release → thinking → expanded (response) → auto-collapse
3. **Meeting recording**: Detect meeting → HUD shows amber status → click tray → panel shows attention card → Record → panel transforms to Meeting/Insights tabs → live transcript flows → Stop → panel returns to default
4. **Voice command**: Fn+Space → speak → HUD shows thinking → tools → expanded response → auto-collapse → click HUD → re-shows response
5. **Approval flow**: Command needs permission → HUD expands with approval card → approve → command executes → HUD shows result
6. **Panel minimize during recording**: Minimize → HUD shows "Recording · 12:34" → click tray → panel reopens with transcript
7. **All rollover links**: Every card in panel opens correct destination in main window
8. **Screen share protection**: During recording, verify HUD and panel hidden from screen share

### Performance Tests
- HUD idle CPU: < 1% when dormant
- HUD expansion animation: < 16ms per frame (60fps)
- Panel open latency: < 200ms from tray click to content visible
- No RAF loops when HUD is dormant
