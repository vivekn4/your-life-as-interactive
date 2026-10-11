# PLAN.md — Your Life As: Interactive

**Repo:** `vivekn4/your-life-as-interactive` · **Type:** branching-narrative browser game · **Date:** 2026-10-11
**Concept:** Playable, choice-driven versions of the "Your Life As" YouTube format (cinematic second-person life stories). Pick a scenario — "Your Life As a ___" — and live it through meaningful choices with multiple endings. BitLife-style stat mechanics meet cinematic storytelling.

## 1. Architecture

Single-file vanilla JS (`index.html`, no build step, ES modules not required — one `<script>` block to keep `file://` and GitHub Pages trivially working).

```
index.html
├── <style>      — cinematic VN theme (dark, neon-gold, serif display + sans body), responsive
├── <canvas id="fx"> — particle layer (ambient motes, chapter-burst, ending FX)
├── Screens (divs, toggled): #menu / #game / #ending
│   ├── #menu: title, VN tag, scenario cards (#scenario-list), endings gallery (#gallery),
│   │          AI Storyteller toggle (#ai-toggle), reset saves (#reset-saves), mute (#mute-btn)
│   ├── #game: chapter card (#chapter-kicker, #chapter-title), narration (#narration),
│   │          vignette line (#vignette), stat bars (#stat-health/-nerve/-allies/-fortune + fills),
│   │          delta chips (#deltas), choices (#choices), event modal (#event-modal),
│   │          quit (#btn-quit), restart (#btn-restart)
│   └── #ending: rank banner (#ending-rank), title (#ending-title), text (#ending-text),
│                stats recap (#ending-stats), again (#btn-again), menu (#btn-menu)
├── #toast — transient messages
└── <script> — ENGINE + STORY DATA + AUDIO + PARTICLES
```

**Engine flow (state machine):**
`MENU → (pick scenario) → CHAPTER[i] → narration → choice[0..n] → result → … → event → CHAPTER[i+1] … → EPILOGUE → evaluate endings → ENDING → gallery unlock → MENU`

**State object:** `{ scenarioId, chapterIdx, nodeIdx, stats:{health,nerve,allies,fortune}, flags:[], history:[{chapter,choiceId,optionId}], aiMode, endingsSeen }`
- Persisted run: `yla_interactive_save` (localStorage). Gallery: `yla_interactive_endings` = `{scenarioId:[endingIds]}`. Settings: `yla_interactive_settings` = `{aiMode, muted}`.
- Stats clamped 0–100, start at 50. Choices apply `effects` deltas + set `flags`. Some options carry `req:{stat,min}` → rendered disabled with lock note.

**Story data format** (per scenario): `chapters[]` of `{id,name,narration,vignette,choices:[{id,lead,options:[{id,text,req?,effects,flags,result}]}],events:[...]}`; final chapter `{id:'epilogue', endings:[{id,title,rank,when(state),text}]}`. Endings evaluated in order; first `when()` true wins (tragic last as default).

## 2. Scenario & ending design (3 launch scenarios)

1. **Deep-Sea Diver** (North Sea saturation diver) — themes: pressure, loyalty vs payday, the deep. Flags: `saved_mara`, `took_wreck_job`, `kept_secret`.
   - Triumphant: legend of the rigs (nerve≥65 && (allies≥55 || saved_mara)). Bittersweet: alive, retired, haunted (health≥30). Tragic: default (lost/disgraced).
2. **Medieval Knight** (Hundred-Years-War-ish) — themes: oath vs conscience, mercy vs orders. Flags: `spared_village`, `won_duel`, `broke_oath`.
   - Triumphant: lord & legend (allies≥60 && !broke_oath). Bittersweet: survives, scarred, forgotten (health≥30). Tragic: default.
3. **K-Pop Idol** (trainee → debut) — themes: fame vs self, burnout, loyalty. Flags: `signed_brutal`, `went_indie`, `confessed`, `protected_member`.
   - Triumphant: icon on own terms (fortune≥60 && nerve≥55). Bittersweet: famous but hollow OR quiet happy life (flag-dependent). Tragic: default (burnout/canceled).

Each playable chapter: narration (~90–130 words, second person, cinematic), 2–3 choice points (2–3 options each), 1 random event between chapters (from 2-variant pool), authored `vignette` line (1–2 sentences) shown after each choice — this is what the AI Storyteller replaces.

## 3. AI Storyteller design + fallback

- **Backbone is hand-authored.** The story graph (chapters, choices, effects, endings) NEVER depends on AI. AI only renders *prose within the current chapter's bounds*: a 1–2 sentence vignette after each choice, and an optional epilogue coda line.
- **Provider:** Puter.js (`https://js.puter.com/v2/`, `puter.ai.chat`) — free, no key, no backend. Loaded lazily only when AI mode is toggled on (script injection on first enable).
- **Prompt construction:** system = "You are a cinematic second-person narrator… 1–2 sentences, present tense, no new plot, no new characters, stay inside: {chapter bounds}"; user = chapter name + narration summary + player's last choice + resulting stat shift + last 3 history entries.
- **Failure policy (automatic):** 12s timeout, any network/parse/API error → silently fall back to the authored `vignette` for that node. AI mode toggle shows status (`#ai-status`: ready/offline/fallback). If Puter fails 3× in a run, auto-disable AI for the run with a toast.
- **Never:** AI does not pick choices, change stats, invent endings, or alter flags. All game state comes from the authored graph.

## 4. Likely bugs + mitigations

| Bug | Mitigation |
|---|---|
| Choice buttons double-fire / skipped nodes | Single `advance()` gate + `busy` flag; buttons disabled on click until render completes |
| Stats drift out of 0–100 | Central `applyEffects()` clamps; test asserts bounds after full playthrough |
| localStorage corrupted/quota | try/catch around all storage; schema version key, wipe-and-continue on parse fail |
| AI script hangs page | Lazy-load only on toggle; 12s timeout; never `await` AI before rendering core text (fire-and-fill: render authored vignette immediately, swap in AI text if it arrives) |
| Ending `when()` overlap → wrong ending | Order endings triumphant→bittersweet→tragic; unit-test every ending predicate against synthetic states |
| Mobile overflow on long narration | Clamp narration panel with scroll; test at 390px width in harness |
| Audio autoplay block | Init AudioContext on first user gesture; mute toggle persisted |
| Puter blocked by ad-blocker | Caught as load failure → status "offline", authored text used; game unaffected |

## 5. UI/UX

- **Menu:** full-bleed title "YOUR LIFE AS" + "INTERACTIVE" neon, scenario cards (icon, title, tagline, endings-unlocked count), gallery strip, footer with VN tag.
- **Game:** top = chapter kicker card (e.g. "CHAPTER II — THE CALL"); narration in large serif, typewriter-fade-in; vignette in italic gold; stat bars always visible (icon + bar + number); choices as full-width cinematic buttons with hover glow + lock notes; delta chips (+10 Nerve) float after each choice; event modal interrupts between chapters.
- **Ending:** rank banner (TRIUMPHANT gold / BITTERSWEET violet / TRAGIC red), title, prose, stat recap, gallery toast "Ending unlocked".
- Maximalist but readable: particle canvas behind content, film-grain/vignette overlay, chapter-transition flash + whoosh.

## 6. SFX/VFX plan (WebAudio synth, canvas — zero assets)

- **SFX:** ui-click (short blip), choice-confirm (two-tone), chapter-whoosh (filtered noise sweep), event-sting (minor chord), stat-up (rising arp), stat-down (falling tone), ending-triumphant (major arpeggio), ending-bittersweet (soft piano-ish pluck), ending-tragic (low drone), ambient pad (quiet detuned oscillators, toggleable with mute).
- **VFX:** ambient floating motes; chapter change = radial burst + screen flash; choice = ripple from button; ending = rank-colored particle rain (gold/violet/ember); stat bar animated width transitions.

## 7. Testing (headless Chromium via CDP, plain Node)

1. Page loads, zero console errors, all 24+ DOM ids resolve.
2. Scripted full playthrough of Deep-Sea Diver: click first available choice at every node until ENDING; assert stats changed from 50s, ending unlocked in gallery, localStorage keys written.
3. Reload mid-run → save restores chapter/state.
4. AI mode ON with network blocked → all vignettes fall back to authored text, no errors.
5. Screenshot `assets/screenshot.png` from a real mid-story frame (narration + choices visible), 1280×800.
6. `node --check` on extracted script; mobile viewport 390px smoke test.
